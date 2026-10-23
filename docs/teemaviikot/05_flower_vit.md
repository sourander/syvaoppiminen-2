# 5: Flower ViT

!!! warning

    Tämä materiaali on toteutettu hyvin, hyvin nopeasti. Opettaja testaa tämän vielä läpi.

Roihu on nyt käytössä, joten on aika kokeilla hieman suurempaa mallia. Materiaali on kirjoitettu ViT (Vision Transformer) -mallin pohjalta, mutta voit käyttää myös muita malleja. Sopivaa mallia voit etsiä [PyTorch Models and pre-trained weights](https://docs.pytorch.org/vision/main/models.html#table-of-all-available-classification-weights)-dokumentaatiosta. Jos haluat paukutella henkseleitä, niin esimerkiksi `ViT_L_16_Weights.IMAGENET1K_V1`-painot koostuvat huikeasta 300 miljoonasta parametrista. Jos käytössä on Adam ja kaikkia parametreja koulutetaan, pelkät painot, gradientit ja Adamin kaksi momenttia vievät fp32-tarkkuudella $4 \times 4 = 16$ tavua per parametri eli noin $300 \times 16 \approx 4,9$ GB muistia. Tämän päälle tulevat aktivaatiot. Voi olla mielenkiintoista kurkata, miltä muistinkäyttö näyttää MLflow:n tilastoissa!

Transformer-arkkitehtuuri käsitellään kunnolla vasta ensi viikolla. Tällä viikolla ViT toimii spoilerina tai maistiaisena. Vaikka arkkitehtuuri on kovin erilainen, mallia käytetään samalla tavalla kuin MobileNetiä eli esikoulutettuna backbonena, jonka päälle vaihdetaan oma luokittelupää. Sisäistä toimintaa ei tarvitse vielä ymmärtää yksityiskohtaisesti. Riittää, että tiedät seuraavat asiat ViT-B/16-mallista:

- Kuva ($224 \times 224$) pilkotaan $16 \times 16$ pikselin paloihin (*engl. patch*).
- Kukin pala muunnetaan 768-ulotteiseksi vektoriksi (*engl. token*). Tämän tekee konvoluutiokerros `conv_proj`, jonka kernelin koko ja askel ovat molemmat 16.
- Jono kulkee 12 peräkkäisen enkooderilohkon läpi. Nämä ovat varsinainen Transfomer-arkkitehtuuri.

Viikon kokonaisuus koostuu seuraavista osista:

- Flower Modelin `model.py` laajennetaan tukemaan useampaa mallikuitua (*engl. model family*) siten, että vanhat MobileNet-konfiguraatiot ja -checkpointit toimivat kuten ennenkin.
- ViT:lle luodaan oma konfiguraatiotiedosto.
- Muutokset testataan omalla koneella yhden epookin savutestillä (*engl. smoke test*), ennen kuin mitään lähetetään Roihuun.
- Koodi siirretään Roihuun viikolta 4 tutulla tavalla GitLabin kautta, ja koulutus käynnistetään `sbatch`-komennolla.
- ViT:n tuloksia verrataan MobileNetiin Roihun MLflow-sovelluksessa.

Opetusmateriaalia ovat:

- Dosovitskiy et al.: [An Image is Worth 16x16 Words: Transformers for Image Recognition at Scale](https://arxiv.org/abs/2010.11929). Tämä on ViT:n alkuperäinen julkaisu. Ensi viikon jälkeen sen lukeminen on helpompaa, mutta kuvat ja johdanto kannattaa silmäillä jo nyt.

## Työvaiheet

### 1. Tutustu ViT:n rakenteeseen

Ennen kuin muokkaat koodia, katso, miltä mallit näyttävät sisältä. Avaa `flower-model-demo`-hakemistossa REPL komennolla `uv run python` ja aja:

```python title="Oma kone (uv run python)"
import torchvision.models as models

mobilenet = models.get_model("mobilenet_v3_small", weights="DEFAULT")
vit = models.get_model("vit_b_16", weights="DEFAULT")

print(mobilenet)
print(vit)

print(len(mobilenet.features), mobilenet.classifier)
print(len(vit.encoder.layers), vit.heads)

for name, m in [("mobilenet", mobilenet), ("vit", vit)]:
    print(name, sum(p.numel() for p in m.parameters()))
```

Huomaa, että ViT-B/16:n painot ovat noin 330 MB, joten ensimmäinen lataus kestää hetken.

Tulosteesta näet, että mallit on koottu eri nimisistä osista:

| Osa                     | MobileNetV3                | ViT                          |
| ----------------------- | -------------------------- | ---------------------------- |
| Piirteitä tuottavat lohkot | `features`              | `encoder.layers` (12 kpl)    |
| Viimeinen normalisointi | (osa `features`-lohkoja)   | `encoder.ln`                 |
| Luokittelupää           | `classifier`               | `heads` (sisällä `head`)     |
| Luokittelupään syötteen leveys | `classifier[0].in_features` (576) | `heads.head.in_features` (768) |

Nykyinen `model.py` olettaa kaikkialla MobileNetin nimet (`model.classifier`, `model.features`). ViT:llä näitä attribuutteja ei ole, joten koodi kaatuisi heti. Seuraavissa kahdessa vaiheessa nimet piilotetaan mallikuitukohtaisten apufunktioiden taakse.

!!! tip "Käytätkö jotain muuta mallia?"

    Jos valitsit jonkin muun mallin kuin ViT:n, selvitä samalla tavalla `print(model)`-tulosteesta ja torchvisionin lähdekoodista, mitkä ovat sen piirrelohkojen ja luokittelupään attribuuttien nimet. Lisää sitten seuraavissa vaiheissa oma `case`-haaransa ViT:n rinnalle.

### 2. `model.py`: mallikuidun valinta

Konfiguraatioon lisätään uusi avain `model.model_family`. Varsinainen malli valitaan edelleen `model.backbone`-avaimella:

```yaml
model:
  model_family: vit              # mobilenetv3 | vit
  backbone: vit_b_16             # vit_b_16 | vit_b_32 | vit_l_16 | vit_l_32
```

Muokkaa tiedostoa `src/flowermodel/modelling/model.py` seuraavasti:

- Luokittelupään rakentaminen siirretään omaan funktioonsa `_build_classifier_head(in_features, model_cfg)`. Pää on täsmälleen sama kuin ennenkin, mutta nyt sitä voi käyttää kumpikin malliperhe. Pää ei ole ehkä ViT:n arkkitehtuurin mukainen, mutta riittääköön tämä meille.
- Nykyinen `_build_backbone` nimetään uudelleen muotoon `_build_mobilenetv3_backbone`, ja se käyttää uutta apufunktiota.
- Uusi funktio `_build_vit_backbone` lataa mallin samalla `models.get_model(...)`-kutsulla, lukee syötteen leveyden kohdasta `model.heads.head.in_features` ja korvaa koko `model.heads`-osan omalla päällä.
- `build_model()` lukee mallikuidun komennolla `model_cfg.get("model_family", "mobilenetv3")` ja valitsee sen perusteella oikean rakentajan.

??? example "model.py: mallin rakentaminen"

    ```python title="src/flowermodel/modelling/model.py"
    import torch.nn as nn

    import torchvision.models as models
    from omegaconf import DictConfig, ListConfig


    def _build_classifier_head(in_features: int, model_cfg) -> nn.Sequential:
        """Build the classifier head shared by all model families."""
        return nn.Sequential(
            nn.Linear(in_features, model_cfg.hidden_dim),
            nn.Hardswish(inplace=True),
            nn.Dropout(p=model_cfg.dropout, inplace=True),
            nn.Linear(model_cfg.hidden_dim, model_cfg.num_classes),
        )


    def _build_mobilenetv3_backbone(model_cfg) -> nn.Module:
        """Load a pretrained MobileNetV3 backbone and replace its classifier head. Note that this will download it to
        the TORCH_HOME. You may want to set this to a custom location when running on a CSC Supercomputer.

        Reference: https://docs.pytorch.org/vision/main/_modules/torchvision/models/mobilenetv3.html"""
        model = models.get_model(model_cfg.backbone, weights="DEFAULT")

        in_features = model.classifier[0].in_features # type: ignore

        model.classifier = _build_classifier_head(in_features, model_cfg)

        return model


    def _build_vit_backbone(model_cfg) -> nn.Module:
        """Load a pretrained Vision Transformer backbone and replace its classifier head. Note that this will download it to
        the TORCH_HOME. You may want to set this to a custom location when running on a CSC Supercomputer.

        Reference: https://docs.pytorch.org/vision/main/_modules/torchvision/models/vision_transformer.html"""
        model = models.get_model(model_cfg.backbone, weights="DEFAULT")

        in_features = model.heads.head.in_features # type: ignore

        model.heads = _build_classifier_head(in_features, model_cfg)

        return model


    # Fine-tuning helpers from step 3 go here


    def build_model(cfg: DictConfig | ListConfig) -> nn.Module:
        """Build and configure the model from the experiment config.

        The model family defaults to 'mobilenetv3' to keep older configs (and
        checkpoints saved from them) working unchanged."""
        model_cfg = cfg.model
        model_family = model_cfg.get("model_family", "mobilenetv3")

        match model_family:
            case "mobilenetv3":
                model = _build_mobilenetv3_backbone(model_cfg)

            case "vit":
                model = _build_vit_backbone(model_cfg)

            case _:
                raise ValueError(
                    f"Unknown model_family: '{model_family}'. "
                    "Expected one of: 'mobilenetv3', 'vit'."
                )

        _apply_fine_tune_strategy(model, model_cfg, model_family)
        model = model.to(cfg.device)

        return model
    ```

Koska backbone ladataan nimen perusteella, myös esimerkiksi `vit_b_32` tai `vit_l_16` toimii pelkästään konfiguraation arvoa vaihtamalla. Koodi on kirjoitettu taaksepäin yhteensopivaksi; vanha MobileNet-konfiguraatio toimii edelleen, koska getterit käyttävät vakioarvoja.

### 3. Mallikuitukohtainen hienosäätö

Funktio `_apply_fine_tune_strategy` tukee kolmea strategiaa: `all`, `classifier_only` ja `last_n_layers`. Nykyinen toteutus viittaa suoraan `model.features`- ja `model.classifier`-attribuutteihin. Siirrä nämä viittaukset apufunktioon `_get_fine_tune_targets`, joka palauttaa parin (piirrelohkot, luokittelupää) mallikuidun perusteella:

| `model_family` | Piirrelohkot     | Luokittelupää      |
| -------------- | ---------------- | ------------------ |
| `mobilenetv3`  | `model.features` | `model.classifier` |
| `vit`          | `model.encoder.layers` | `model.heads` |

ViT:ssä on lisäksi yksi yksityiskohta. Enkooderilohkojen jälkeen, ennen luokittelupäätä, on vielä viimeinen LayerNorm-kerros `model.encoder.ln`. Se ei kuulu `encoder.layers`-listaan, joten se jäisi `last_n_layers`-strategiassa jäädytetyksi, vaikka sen alapuolella olevia lohkoja koulutetaan. Tämän vuoksi lisätään vielä apufunktio `_get_final_norm_layers`, joka palauttaa ViT:lle `[model.encoder.ln]` ja muille tyhjän listan. MobileNetin toiminta ei siis muutu.

??? example "model.py: hienosäätö"

    ```python title="src/flowermodel/modelling/model.py"
    def _get_fine_tune_targets(model: nn.Module, model_family: str) -> tuple[list[nn.Module], nn.Module]:
        """Return the (feature blocks, classifier head) pairs for the given model family."""
        match model_family:
            case "mobilenetv3":
                return list(model.features), model.classifier # type: ignore

            case "vit":
                return list(model.encoder.layers), model.heads # type: ignore

            case _:
                raise ValueError(
                    f"Unknown model_family: '{model_family}'. "
                    "Expected one of: 'mobilenetv3', 'vit'."
                )


    def _get_final_norm_layers(model: nn.Module, model_family: str) -> list[nn.Module]:
        """Return normalization layers that should be trainable together with the
        last N feature blocks (e.g. the final LayerNorm of a ViT encoder)."""
        match model_family:
            case "vit":
                return [model.encoder.ln] # type: ignore

            case _:
                return []


    def _apply_fine_tune_strategy(model: nn.Module, model_cfg, model_family: str) -> None:
        """Freeze/unfreeze model parameters according to the configured strategy."""
        strategy = model_cfg.fine_tune.strategy

        features, classifier = _get_fine_tune_targets(model, model_family)

        # Start by freezing everything
        for param in model.parameters():
            param.requires_grad = False

        match strategy:
            case "all":
                for param in model.parameters():
                    param.requires_grad = True

            case "classifier_only":
                for param in classifier.parameters():
                    param.requires_grad = True

            case "last_n_layers":
                num_layers = model_cfg.fine_tune.num_layers
                trainable_blocks = features[-num_layers:] + _get_final_norm_layers(model, model_family)
                for block in trainable_blocks:
                    for param in block.parameters():
                        param.requires_grad = True
                for param in classifier.parameters():
                    param.requires_grad = True

            case _:
                raise ValueError(
                    f"Unknown fine_tune strategy: '{strategy}'. "
                    "Expected one of: 'classifier_only', 'last_n_layers', 'all'."
                )
    ```

Kun vaihe 4 on tehty, voit tarkistaa REPL:ssä, montako parametria kukin strategia todella kouluttaa:

```python title="Oma kone (uv run python)"
from flowermodel.config import load_config
from flowermodel.modelling.model import build_model

cfg = load_config("configs/experiments/vit_b_16_baseline.yaml")
cfg.device = "cpu"
model = build_model(cfg)

total = sum(p.numel() for p in model.parameters())
trainable = sum(p.numel() for p in model.parameters() if p.requires_grad)
print(f"{trainable:,} / {total:,}")
```

Kokeile samaa myös MobileNet-konfiguraatiolla. Tätä lukua tarvitset vaiheessa 8.

### 4. Konfiguraatio `vit_b_16_baseline.yaml`

Luo uusi konfiguraatiotiedosto MobileNet-baselinen rinnalle. Se on lähes sama kuin `mobilenet_v3_small_baseline.yaml`, mutta `model`-osio on uusi, eräkoko on pienempi ja `tracking`-osion nimet on vaihdettu:

```yaml title="configs/experiments/vit_b_16_baseline.yaml"
seed: 42
device: auto  # auto | cpu | cuda | mps

data:
  base_dir: data/flower_photos # includes train, val, test, online subdirs
  batch_size: 64
  num_workers: 6
  augmentations:

    # Note: this stack MUST include the 224x224 crop in form or another!
    - name: RandomResizedCrop
      size: [224, 224]
      scale: [0.8, 1.0]

    - name: RandomHorizontalFlip
      p: 0.5

    - name: ColorJitter
      brightness: 0.2
      contrast: 0.2
      saturation: 0.2
      hue: 0.1

model:
  model_family: vit              # mobilenetv3 | vit
  backbone: vit_b_16             # vit_b_16 | vit_b_32 | vit_l_16 | vit_l_32
  num_classes: 5
  dropout: 0.2
  hidden_dim: 1024
  fine_tune:
    strategy: last_n_layers      # classifier_only | last_n_layers | all
    num_layers: 2                # only used when strategy: last_n_layers; ViT-B/16 has 12 encoder blocks

loss:
  name: cross_entropy
  label_smoothing: 0.0

optimizer:
  name: adam
  lr: 1.0e-3
  weight_decay: 0.0

training:
  epochs: 15

tracking:
  experiment_name: FlowerModel_ViT_B_16_Baseline
  url: http://localhost:5001
  best_model_path: models/vit_b_16_baseline.pth
```

Voit halutessasi lisätä MobileNet-konfiguraatioon rivin `model_family: mobilenetv3`. Se ei ole pakollista oletusarvon ansiosta, mutta tekee konfiguraatiosta yksiselitteisemmän.

### 5. Savutesti omalla koneella

Roihun jonoon ei kannata lähettää koodia, jota ei ole kertaakaan ajettu. Käynnistä paikallinen MLflow ja aja yksi epookki:

```bash title="Oma kone"
cd ~/Code/syvaoppiminen-2-2026/<nimi>/flower-model-demo
just flow

uv run python src/flowermodel/train.py \
    --config configs/experiments/vit_b_16_baseline.yaml \
    --override training.epochs=1
```

Aja sama myös MobileNet-konfiguraatiolla. Näin varmistat, että vanha malli toimii muutosten jälkeen edelleen:

```bash title="Oma kone"
uv run python src/flowermodel/train.py \
    --config configs/experiments/mobilenet_v3_small_baseline.yaml \
    --override training.epochs=1
```

Savutestin tarkoitus on löytää kirjoitusvirheet, väärät attribuuttien nimet ja puuttuvat konfiguraatioavaimet. Mallin suorituskyvyllä ei ole tässä vaiheessa väliä. Tarkista MLflow:sta, että ajo valmistui ja metriikat sekä artefaktit tallentuivat.

!!! warning

    ViT-B/16 on noin 30 kertaa suurempi kuin MobileNetV3 Small, joten yksikin epookki voi kestää ilman GPU:ta pitkään. Jos paikallisen GPU:n muisti loppuu, pienennä eräkokoa komentoriviltä, esimerkiksi `--override training.epochs=1 data.batch_size=16`. Konfiguraatiotiedostoon ei tarvitse koskea.

### 6. `roihu_train.sh`: ViT-konfiguraatio

Viikon 4 työskriptissäsi koulutus käyttää oletuskonfiguraatiota (MobileNet), ja experimentin nimi sekä mallin tallennuspolku ovat MobileNetin arvoja. Muuta skriptin viimeistä `srun`-komentoa siten, että se käyttää ViT-konfiguraatiota `--config`-argumentilla ja experimentin nimi sekä mallin polku vaihtuvat. Muu skripti pysyy ennallaan:

```bash title="scripts/roihu_train.sh"
cd "$CLONE"
srun python -m flowermodel.train \
    --config configs/experiments/vit_b_16_baseline.yaml \
    --override \
    "data.base_dir=$TMPDIR/flower_photos" \
    "tracking.url=sqlite:///$DL2_SCRATCH/mlflow/mlruns.db" \
    "tracking.experiment_name=FlowerModel_ViT_B_16_Roihu" \
    "tracking.best_model_path=$DL2_SCRATCH/models/vit_b_16_roihu.pth"
```

!!! tip

    Jos haluat, saat toki tehdä tiedostosta kopion ja nimetä sen uusiksi.

Konfiguraation polku on suhteellinen, joten sen on oltava suhteessa kloonin juureen. Siksi skripti siirtyy `cd "$CLONE"` -komennolla kloonin hakemistoon ennen koulutusta.

### 7. Koulutus Roihussa

Commitoi ja puske muutokset omalta koneeltasi:

```bash title="Oma kone"
cd ~/Code/syvaoppiminen-2-2026/<nimi>
git add .
git commit -m "Add ViT backbone support"
git push
```

Kirjaudu Roihuun SSH:lla, päivitä klooni ja lähetä ViT-koulutus jonoon:

```bash title="CSC Roihu"
cd $DL2_PROJ/04_flower_at_roihu/<nimi>
git pull

cd flower-model-demo
sbatch scripts/roihu_train.sh
squeue --me
```

Työ käyttää edelleen `gputest`-partitiota. Baseline-konfiguraation 15 epookkia mahtuvat sen aikarajaan. Ensimmäisellä kerralla työ lataa ViT:n painot `TORCH_HOME`-hakemistoon `/scratch`-levyalueelle, ja myöhemmät ajot käyttävät ladattuja painoja.

Kun työ on valmis, tarkastele sitä tutuilla komennoilla (`seff`, `sacct`) ja lue tuloste `slurm-<jobid>.out`. Jos työ kaatuu, vertaa virheilmoitusta paikalliseen savutestiin. Ero johtuu todennäköisesti Roihu-kohtaisesta asiasta, kuten polusta tai ympäristömuuttujasta, eikä mallin koodista.

### 8. Tulokset ja vertailu

Avaa Roihun MLflow-sovellus kuten viikolla 4. Experimentin `FlowerModel_ViT_B_16_Roihu` alta pitäisi löytyä ajo.

Tavoitteena on, että ViT-mallin **`best/f1_val` on 95 % tai parempi**. Jos baseline ei yllä tähän, säädä hyperparametreja. Säädettäviä ovat esimerkiksi:

- `model.fine_tune.strategy` ja `model.fine_tune.num_layers`: kuinka suuri osa enkooderista koulutetaan.
- `optimizer.lr` ja `optimizer.weight_decay`.
- `model.dropout` ja `model.hidden_dim`.
- `loss.label_smoothing`.
- `training.epochs`.

Esikoulutetun mallin hienosäätöön pätee nyrkkisääntö: mitä suurempi osa backbonesta avataan koulutettavaksi, sitä varovaisemmin painoja kannattaa muuttaa. Oikeat arvot sinun tulee kuitenkin löytää kokeilemalla. Kokeiluja voi tehdä joko luomalla uusia konfiguraatiotiedostoja tai käyttämällä `--override`-mekanismia. Muista kuitenkin, että konfiguraatiotiedoston tulee kertoa, millä arvoilla lopullinen malli on koulutettu.

Kun tavoite on saavutettu, vertaa ViT:tä Roihussa koulutettuun MobileNetiin:

- F1-metriikat validointi- ja testidatalla.
- Koulutuksen kesto (`sacct` tai MLflow).
- Koulutettavien parametrien määrä (ks. vaihe 3).

Pohdi, onko F1-metriikan parannus kustannusten arvoinen. Ota huomioon myös vaiheessa 1 näkemäsi koko ja se, että viikolla 1 käytetty selaininferenssi ajaa mallia käyttäjän omalla laitteella.

### 9. Oppimisnopeuden ajastin (valinnainen)

!!! tip

    Tämä tehtävä on valinnainen, syventävä lisätehtävä, jolla on ainoastaan arvosanaa korottava vaikutus.

Tähän asti oppimisnopeus on pysynyt koko koulutuksen ajan vakiona. Transformer-malleja hienosäädettäessä on tavallista käyttää ajastinta (*engl. learning rate scheduler*), joka muuttaa oppimisnopeutta koulutuksen aikana. Toteutetaan ajastin, jossa on kaksi vaihetta:

1. **Lämmittely** (*engl. warmup*): oppimisnopeus alkaa 1 %:sta perusarvosta ja kasvaa lineaarisesti perusarvoon `warmup_epochs` epookin aikana.
2. **Kosinilasku** (*engl. cosine decay*): loppuepookkien aikana oppimisnopeus laskee kosinikäyrän mukaisesti arvoon `eta_min`.

Ajastinta päivitetään kerran epookissa. Konfiguraatiossa ajastin on `optimizer`-osion alla:

```yaml
optimizer:
  name: adam
  lr: 1.0e-3
  weight_decay: 0.0
  scheduler:
    name: cosine                 # none | cosine
    warmup_epochs: 2
    eta_min: 1.0e-6
```

Muutoksia tarvitaan kolmeen tiedostoon. Kaikki muutokset ovat taaksepäin yhteensopivia: konfiguraatio, jossa ei ole `scheduler`-osiota, kouluttaa vakio-oppimisnopeudella kuten ennenkin.

**`src/flowermodel/modelling/optim.py`.** `build_optimizer()` välittää kaikki `optimizer`-osion avaimet `name`-avainta lukuun ottamatta optimoijalle. Adam ei tunne `scheduler`-argumenttia, joten se pitää poistaa ennen optimoijan luontia:

```python title="src/flowermodel/modelling/optim.py"
    opt_kwargs = dict(optimizer_cfg)
    opt_name = opt_kwargs.pop("name")
    opt_kwargs.pop("scheduler", None)
```

Lisää samaan tiedostoon uusi funktio `build_scheduler`. Funktio käyttää PyTorchin `LambdaLR`-ajastinta, joka kertoo perusoppimisnopeuden epookin perusteella lasketulla kertoimella:

??? example "build_scheduler"

    ```python title="src/flowermodel/modelling/optim.py"
    import math

    # ...

    def build_scheduler(cfg, optimizer: optim.Optimizer) -> optim.lr_scheduler.LRScheduler | None:
        """Build an optional learning-rate scheduler from config.

        Configs without an `optimizer.scheduler` section (all older configs) keep
        their constant learning rate and return None. The scheduler is stepped once
        per epoch by the Trainer, so the epoch-0 learning rate is applied when the
        scheduler is built."""
        if "scheduler" not in cfg.optimizer:
            return None

        scheduler_cfg = cfg.optimizer.scheduler
        scheduler_kwargs = dict(scheduler_cfg)
        scheduler_name = scheduler_kwargs.pop("name")
        warmup_epochs = scheduler_kwargs.pop("warmup_epochs", 0)
        min_lr = scheduler_kwargs.pop("eta_min", 0.0)
        epochs = cfg.training.epochs

        match scheduler_name:
            case "none":
                return None

            case "cosine":
                base_lr = optimizer.param_groups[0]["lr"]
                min_lr_factor = min_lr / base_lr if base_lr > 0 else 0.0
                warmup_epochs = min(warmup_epochs, max(epochs - 1, 0))
                warmup_start_factor = 0.01
                decay_epochs = max(epochs - 1 - warmup_epochs, 1)

                def lr_lambda(epoch: int) -> float:
                    if warmup_epochs > 0 and epoch < warmup_epochs:
                        return warmup_start_factor + (
                            1.0 - warmup_start_factor
                        ) * epoch / warmup_epochs
                    progress = (epoch - warmup_epochs) / decay_epochs
                    progress = min(max(progress, 0.0), 1.0)
                    return min_lr_factor + (1.0 - min_lr_factor) * 0.5 * (
                        1.0 + math.cos(math.pi * progress)
                    )

                return optim.lr_scheduler.LambdaLR(optimizer, lr_lambda=lr_lambda)

            case _:
                raise ValueError(
                    f"Unknown scheduler: '{scheduler_name}'. "
                    "Expected one of: 'cosine', 'none'."
                )
    ```

**`src/flowermodel/modelling/training.py`.** `Trainer` saa uuden valinnaisen argumentin `scheduler: optim.lr_scheduler.LRScheduler | None = None`, joka tallennetaan `self.scheduler`-attribuuttiin. `fit()`-metodissa ajastinta askelletaan kerran epookin lopussa validoinnin ja parhaan mallin tarkistuksen jälkeen, ja sen hetkinen oppimisnopeus kirjataan MLflow:iin:

```python title="src/flowermodel/modelling/training.py"
            if self.scheduler is not None:
                self.scheduler.step()
                self.tracker.log_metric(
                    "train/lr", self.optimizer.param_groups[0]["lr"], step=epoch
                )
```

**`src/flowermodel/train.py`.** Ajastin rakennetaan optimoijan jälkeen ja annetaan `Trainer`-luokalle:

```python title="src/flowermodel/train.py"
from flowermodel.modelling.optim import build_loss, build_optimizer, build_scheduler

# ...
    optimizer = build_optimizer(cfg, model)
    scheduler = build_scheduler(cfg, optimizer)

# ...
        trainer = Trainer(model, optimizer, loss, cfg=cfg, tracker=tracker, scheduler=scheduler)
```

Savutestaa muutos paikallisesti muutaman epookin ajolla ja tarkista MLflow:sta, että `train/lr`-käyrä näyttää odotetulta. Kouluta sitten Roihussa ViT ajastimen kanssa ja vertaa tulosta vakio-oppimisnopeudella koulutettuun ajoon.

Selvitä:

- Miksi `warmup_epochs` ja `eta_min` poistetaan (`pop`) sanakirjasta, vaikka niitä ei välitetä minnekään?
- Miksi esikoulutettua mallia hienosäädettäessä alkuun voi olla hyödyllistä käyttää pientä oppimisnopeutta, vaikka luokittelupää on alustettu satunnaisesti?

### 10. FirecREST ja Justfile (valinnainen)

!!! tip

    Tämä tehtävä on valinnainen, syventävä lisätehtävä, jolla on ainoastaan arvosanaa korottava vaikutus.

Jos teit viikolla 4 FirecREST-osuuden, huomaat, että `just roihu-submit` lähettää aina skriptin `scripts/roihu_train.sh` oletusarvoilla, eli se kouluttaa MobileNetin. Laajenna reseptiä siten, että sillä voi käynnistää myös ViT-koulutuksen. Tähän ei anneta valmista ratkaisua, mutta tässä on muutama vinkki:

- Tutki, mitä `roihu-submit`-reseptin `payload`-sanakirjan `env`-kenttä jo nyt välittää työlle. Samaa reittiä voi välittää myös muita muuttujia.
- Just-resepti voi ottaa parametreja (ks. `roihu-log lines="100"`), ja reseptin Python-koodi näkee paikallisen shellin ympäristömuuttujat `os.environ`-sanakirjassa.
- Kun arvo lähetetään omalta koneeltasi, Roihun shell ei laajenna sen sisältämiä muuttujia. Esimerkiksi merkkijono `$DL2_SCRATCH/models/vit.pth` päätyisi skriptille sellaisenaan. Mieti, miten polku kannattaa tällöin antaa.
- Kaikkien ajojen tuloste kirjoitetaan samaan `slurm-flower.out`-tiedostoon. Mieti, onko tämä ongelma.

## Videolla esitettävä

1. Esittele ViT:n rakenne lyhyesti ja vertaa sitä MobileNetiin.
    - Kerro, mitä tarkoittavat patch, token ja luokkatoken, ja montako tokenia 224 × 224 -kuvasta syntyy.
    - Näytä `print(model)`-tulosteesta, missä ovat ViT:n enkooderilohkot, viimeinen LayerNorm ja luokittelupää.

2. Esittele muutokset koodiin.
    - Näytä `model.py`-tiedostosta, miten mallikuitu valitaan ja miten hienosäätö toimii kummallekin mallikuidulle.
    - Perustele, miksi `model_family` luetaan oletusarvon kanssa ja miksi `encoder.ln` otetaan mukaan `last_n_layers`-strategiaan.
    - Näytä `vit_b_16_baseline.yaml` ja `roihu_train.sh`-skriptin muutettu `srun`-komento.

3. Näytä paikallinen savutesti sekä ViT- että MobileNet-konfiguraatiolla.

4. Näytä, miten koodi päätyy Roihuun, ja lähetä ViT-koulutus `sbatch`-komennolla. Tarkastele valmistunutta työtä `seff`-komennolla.

5. Näytä ajo Roihun MLflow-sovelluksessa ja osoita, että `best/f1_val` on vähintään 95 %.
    - Kerro, mitä hyperparametreja säädit ja miksi.

6. Esittele ViT:n ja MobileNetin vertailu: F1-metriikat, koulutuksen kesto ja koulutettavien parametrien määrä. Pohdi, kumpaa mallia käyttäisit ja missä tilanteessa.

7. (Valinnainen) Näytä oppimisnopeuden ajastimen toteutus, `train/lr`-käyrä MLflow:ssa ja vertailu vakio-oppimisnopeudella koulutettuun ajoon.

8. (Valinnainen) Käynnistä ViT-koulutus `just roihu-submit` -komennolla ja selitä, miten muutit reseptiä.

Kokonaisuutena videosta tulee ilmetä, että osaat vaihtaa koulutusprojektin backbonen suurempaan malliin rikkomatta aiempia konfiguraatioita tai checkpointeja. Ymmärrät, miksi muutokset testataan ensin paikallisesti ennen supertietokoneen resurssien käyttöä, ja osaat ajaa saman koodin Roihussa eri konfiguraatioilla ilman, että työskripti haarautuu.