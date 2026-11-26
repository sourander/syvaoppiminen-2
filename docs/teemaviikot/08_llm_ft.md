# 8: Kielimallin hienosäätö

Tarkasti rajattuun tehtävään hienosäädetyn tekoälyn lupaus on suuri: _"we'll see why a small, open source model running on a single GPU can and should beat the world's smartest AI in your specific domain."_ [^moroneyFineTuningAI]. Tämä ei kuitenkaan ole hienosäädön automaattinen lopputulos, vaan hypoteesi, joka täytyy osoittaa erillisellä testiaineistolla. Moroney esittelee kirjassaan taulukon, jossa on kärkimallina senhetkinen huippumalli GPT-5.5 Pro ja vastineena edullisimmassa päässä on DeepSeek V4 Pro. Miljoonaa tokenia kohden näiden hinta on input/output...

- GPT: \$30/\$180
- DeepSeek: \$0.44/\$0.87

Jos malleja vertailee eräajossa, esimerkiksi sopimusanalyysin kontekstissa, niin 5000 dokumentin käsittely maksaisi GPT-5.5 Prolla noin 8000 dollaria ja DeepSeekillä noin 60 dollaria. Mikäli luet tätä merkittävästi myöhemmin kuin syksyllä 2026, mallit ja hinnat ovat voineet muuttua. Voit kurkata tätä esimerkiksi [OpenCode Zen](https://opencode.ai/docs/console/models/#pricing)-hinnastosta, joka on pay-as-you-go -tyyppinen inferenssipalvelu.

Hienosäädön idea on, että löydetään pieni, edullinen malli, joka suoriutuu tehtävästä riittävän hyvin. Termi _riittävän hyvin_ on bisneskysymys ja vaatii tuekseen metriikoita, aivan kuten mikä tahansa muukin koneoppimisen projekti. Se, onko tuloste monivalinta, tietyn skeeman mukainen JSON vai esimerkiksi vapaatekstitiivistelmä, vaikuttaa evaluointimetriikkaan. Tässä materiaalissa evaluointi jätetään tekemättä, mutta voit tutustua siihen esimerkiksi Hugging Facen [The LLM Evaluation Guidebook](https://huggingface.co/spaces/OpenEvals/evaluation-guidebook)-dokumentista tai kirjaston [Inspect AI](https://inspect.aisi.org.uk/tutorial.html) tutoriaalista.

Se, kannattaako mallille antaa vain parempi konteksti RAG-tyylisiä ratkaisuja, skill-tiedostoja tai muita kehotemuotoilun keinoja käyttäen vai hienosäätää malli erikseen, riippuu tapauksesta. Chip Huyen esittelee keskeiseksi hienosäätöä puoltavaksi syyksi sen, että tarvittu ==käytös== ei ole ollut osa sen koulutusta, kuten esimerkiksi tiettyyn skeemaan sopiva JSON-vastaus tai tietty SQL-dialekti. Syiksi vältellä hienosäätöä mainitaan, että se vaatii kehotemuotoiluun verrattuna merkittävän määrän ajallista panostusta, ja monissa tapauksissa kehotemuotoilu riittää tai johtaa parempiin tuloksiin. Jos kyse on siitä, että mallilla ei ole pääsyä ajantasaiseen informaatioon, RAG-tyylinen ratkaisu lienee oikea valinta. Chipin suositus on: _"explore more advanced solutions only if prompting alone proves inadequate"_ [^huyenAIEngineering].

Me ollaan täällä kuitenkin oppimassa hienosäätöä, joten unohdetaan toistaiseksi se, onko tässä mitään järkeä, ja hienosäädetään malli. Käyttämämme metodi on PEFT (Parameter-Efficient Fine-Tuning), joka mahdollistaa mallin hienosäädön ilman, että koko mallia tarvitsee kouluttaa uudelleen -- vain lisätyt parametrit koulutetaan. Perinteisessä PEFT:ssä malliin lisätään parametrejä, joten mallin inferenssinopeus hidastuu [^huyenAIEngineering]. PEFT-metodeista käytetyimmäksi Chip mainitsee LoRA:n (Low-Rank Adaptation), mutta muitakin löytyy, kuten QLoRA (Quantized Low-Rank Adaptation), DoRA (Direction and Magnitude Decomposition), LoftQ (Quantization-Aware LoRA Initialization) [^nicoleTransformers]. Tutustu ihmeessä ainakin pintapuolisesti Hu et al. paperiin [LoRA: Low-Rank Adaptation of Large Language Models](https://arxiv.org/abs/2106.09685). Avuksi voit ottaa Paper Explained -kanavan videon [LoRA Explained Visually](https://youtu.be/DfYNsH3xK1Y?si=zEgG0685hJLYeqFC).

## Työvaiheet

### 1: LoRA-todiste

Tutustu siihen, kuinka LoRA toimii. Tähän on hyvä apu Nicole Koenigsteinin kirjassa _Transformers in Action_. Alla on Marimo Notebook, joka myötäilee kirjan luvun 8 esimerkkiä [^nicoleTransformers2025].

``` python title="lora_proof.py"
# /// script
# requires-python = ">=3.14"
# dependencies = [
#     "marimo>=0.25.2",
#     "numpy>=2.5.5",
# ]
# ///

import marimo

__generated_with = "0.25.2"
app = marimo.App(width="medium")


@app.cell
def _():
    import marimo as mo
    import numpy as np

    return mo, np


@app.cell(hide_code=True)
def _(mo):
    mo.md(r"""
    # LoRA Proof

    The cell below is modified from Listing 8.11 in Nicole Koenigstein's book *Transformers in Action*, Manning, 2025.
    """)
    return


@app.cell
def _(np, rank_slider):
    def lora_decomposition(W, r):
        np.random.seed(42)
        B = np.zeros((W.shape[0], r))
        A = np.random.normal(0, 1, (r, W.shape[1]))

        U, S, Vt = np.linalg.svd(W, full_matrices=False)
        B[:, :r] = U[:, :r] * np.sqrt(S[:r])
        A[:r, :] = np.sqrt(S[:r])[:, np.newaxis] * Vt[:r, :]

        W_approx = B @ A

        return B, A, W_approx

    d = 6
    rng = np.random.default_rng(42)
    W = rng.integers(0, 10, size=(d, d))

    r = rank_slider.value

    B, A, W_approx = lora_decomposition(W, r)
    return A, B, W, W_approx


@app.cell
def _(mo):
    rank_slider = mo.ui.slider(start=1, stop=6, label="Rank", value=5)
    rank_slider
    return (rank_slider,)


@app.cell(hide_code=True)
def _(A, B, W, W_approx, mo):
    common_args = {"precision": 3, "disabled": True}

    _B = mo.ui.matrix(B, label="$B$", **common_args)
    _A = mo.ui.matrix(A, label="$A$", **common_args)
    _W = mo.ui.matrix(W, label="$W$", **common_args) 
    _W2 = mo.ui.matrix(W_approx, label=r"$\hat{W}$", **common_args)

    mo.hstack(
        [
            mo.vstack([_B, _A]),
            mo.vstack([_W, _W2]),
        ]
    )
    return


if __name__ == "__main__":
    app.run()
```

Luo tiedosto koneellesi ja aja se komennolla:

``` bash
uvx marimo edit --sandbox lora_proof.py
```

Säädä rank-liukusäädintä sallittujen arvojen (1-6) välillä ja tarkkaile, kuinka matriisi $\hat{W}$ lähestyy alkuperäistä matriisia $W$. Pohdi samalla:

- Miten approksimaatio muuttuu, kun rank-arvo kasvaa?
- Millä rankin arvolla $\hat{W}$ vastaa alkuperäistä matriisia $W$ täydellisesti (pois lukien pyöristysvirheet)?
- Miten tämä liittyy _Johdatus koneoppimiseen_ -kurssilta tuttuun dimensiovähennykseen?

Tässä demonstraatiossa matalan rankin matriisit muodostetaan SVD:n avulla. Varsinaisessa LoRA-menetelmässä matriisit $A$ ja $B$ koulutetaan gradienttien avulla. LoRA käyttää niiden tuloa painomatriisin päivityksen esittämiseen.

$$
W' = W + \Delta W, \qquad \Delta W = BA
$$

### 2: Kielimallin hienosäätö (Yoda-demo)

Tässä työvaiheessa hienosäädetään pienehkö avoin kielimalli (`microsoft/Phi-3-mini-4k-instruct`) puhumaan Yoda-tyylistä englantia `dvgodoy/yoda_sentences`-aineistolla käyttäen PEFT/LoRA-menetelmää ja MLflow-seurantaa.

Koulutuskoodia ei tarvitse kirjoittaa itse. Valmis toteutus löytyy GitLab-repositoriosta [peft-yoda-demo](https://gitlab.dclabra.fi/jani-public/peft-yoda-demo), joka yhdistelee Daniel Voigt Godoyn Hugging Face -tutoriaalia, MLflow:n PEFT-opasta sekä CSC:n hienosäätöesimerkkejä. Tavoitteena on ajaa hienosäätö ja inferenssi läpi sekä ymmärtää koodin ja käytettyjen menetelmien toimintaperiaate. Alla vielä linkit alkuperäisiin lähteisiin:

- [Fine-Tuning Your First Large Language Model (LLM) with PyTorch and Hugging Face](https://huggingface.co/blog/dvgodoy/fine-tuning-llm-hugging-face)
- [Fine-Tuning Open-Source LLM using QLoRA with MLflow and PEFT](https://mlflow.org/docs/latest/ml/deep-learning/transformers/tutorials/fine-tuning/transformers-peft/)
- [CSC llm-fine-tuning-examples](https://github.com/CSCfi/llm-fine-tuning-examples)

Ennen kuin jatkat pidemmälle, tutustu yllä oleviin linkkeihin. Niiden lukeminen auttaa sinua tekemään tasokkaan, faktoiltaan kunnossa olevan perjantaidemon.

#### A: Repositorion kloonaus ja paikallinen ajo

Kloonaa esimerkkirepositorio omalle koneellesi:

``` bash title="Oma kone"
git clone https://gitlab.dclabra.fi/jani-public/peft-yoda-demo.git
cd peft-yoda-demo
```

Tutustu ensin repositorion `README.md`- ja `Justfile`-tiedostoihin, joihin komennot ja parametrit on koottu.

==Jos omalla koneellasi on== sopiva CUDA-näytönohjain (vähintään noin 6–8 GB VRAM; opettajan testikoneella Nvidia RTX 3060 Ti:llä QLoRA-koulutus vei noin 5,2 GB muistia ja kesti noin 3 minuuttia), suorita ensin paikallinen savutesti (_smoke test_), koulutus ja inferenssi:

``` bash title="Oma kone"
# Käynnistä paikallinen MLflow-palvelin (Docker Compose, portti 5001)
just flow

# Aja nopea savutesti (rank=1, 40 näytettä, 5 askelta)
just smoketest

# Aja varsinainen paikallinen QLoRA-koulutus (rank=8, 3 epookkia)
just localtrain

# Testaa hienosäädetyn mallin ja perusmallin eroa yksittäisellä kehotteella
just infer 'Supercomputers are like normal computers, but in cluster and with fast network.'

# Testaa eräinferenssiä prompts.txt-tiedoston kehotteilla
just infer-batch
```

Tarkastele ajon jälkeen paikallisen MLflow:n käyttöliittymästä (`http://localhost:5001`) koulutusmetriikoita, GPU-muistin käyttöä sekä lokitettua `generations/yoda_generations.md`-vertailutaulukkoa.

``` bash title="Oma kone"
# Sammuta lopuksi paikallinen MLflow-palvelin
just noflow
```

!!! note "Jos koneellasi ei ole sopivaa GPU:ta"

    Paikalliset ajot ovat valinnaisia. Jos sinulla ei ole sopivaa näytönohjainta, voit siirtyä suoraan koodin lukemiseen ja Roihu-ajoon.

#### B: Tutustu koodiin ja vertaa tutoriaaliin

Riippumatta siitä, ajoitko koulutuksen paikallisesti vai et, lue huolellisesti repositorion keskeiset tiedostot:

- `src/peft_yoda_demo/finetune_yoda.py`
- `src/peft_yoda_demo/infer_yoda.py`
- `src/peft_yoda_demo/common.py`
- `Justfile` ja `slurm/roihu_finetune_yoda.sh`

Lue samalla rinnalla Daniel Voigt Godoyn Hugging Face -tutoriaali [Fine-Tuning Your First Large Language Model (LLM) with PyTorch and Hugging Face](https://huggingface.co/blog/dvgodoy/fine-tuning-llm-hugging-face). Paikanna koodista tutoriaalin keskeiset vaiheet ja varmista, että osaat selittää korkealla tasolla, mitä koulutus- ja inferenssiskriptit tekevät. Tarkemmat asiat, jotka sinun tulee käydä läpi ja selittää, on listattu alla kohdassa _Videolla esitettävä_.

#### C: Hienosäätö CSC Roihussa

Seuraavaksi ajetaan sama hienosäätö CSC:n Roihu-supertietokoneen GH200-GPU:lla. Muista, että Roihun skripti hyödyntää viikolla 4 määrittämääsi `$DL2_SCRATCH`-ympäristömuuttujaa.

**Päivitä ensin projektinumerosi** oman CSC Student Projectisi mukaisesti (`project_XXXXXXX`) kahteen tiedostoon:

- `Justfile`: muuttuja `roihu_dir` (`/projappl/project_XXXXXXX/peft_yoda_demo`)
- `slurm/roihu_finetune_yoda.sh`: rivi `#SBATCH --account=project_XXXXXXX`

**Kopioi tiedostot Roihuun** omalta koneeltasi:

``` bash title="Oma kone"
just roihu-push
```

**Lähetä koulutustyö jonoon** GPU-kirjautumissolmulta (`roihu-gpu`):

``` bash title="CSC Roihu (roihu-gpu)"
cd /projappl/project_XXXXXXX/peft_yoda_demo
sbatch roihu_finetune_yoda.sh
```

Seuraa työn etenemistä komennolla `squeue --me` ja tarkista valmistunut tuloste tiedostosta `slurm-<jobid>.out`. Avaa tämän jälkeen MLflow Roihun verkkokäyttöliittymässä (tietokanta `$DL2_SCRATCH/mlflow/mlruns.db`) ja tutki koulutuksen parametreja, häviökäyriä, GPU-muistin kulutusta sekä `generations/yoda_generations.md`-artefaktia.

#### D: Inferenssi CSC Roihussa

Kirjoita oma Slurm-skripti (esimerkiksi `roihu_infer_yoda.sh`), joka ajaa eräinferenssin `infer_yoda.py`-skriptillä ja `prompts.txt`-tiedostolla Roihun `gputest`-partitiolla.

!!! tip "Vinkki inferenssiskriptin kirjoittamiseen"

    Katso mallia `roihu_finetune_yoda.sh`-skriptistä (muista `module load python-pytorch/2.13` ja `export HF_HOME="$DL2_SCRATCH/hf_home"`) sekä `Justfile`-tiedoston `infer-batch`-komennosta. Lisää komentoon myös `--no-qlora`-lippu, jotta malli ladataan Roihussa samalla `bfloat16`-tarkkuudella kuin koulutuksessa.

Lähetä inferenssityö jonoon (`sbatch roihu_infer_yoda.sh`) ja tarkista `slurm-<jobid>.out`-tulosteesta, miten perusmallin (`Base`) ja hienosäädetyn mallin (`Yoda`) vastaukset eroavat toisistaan. ==Base-mallista poiketa Yoda-mallin pitäisi. Jos samanlaisia vastaukset ovat, lokia tutkia sinun täytyy.==

## Videolla esitettävä

1.  **LoRA:n perusidea:**

    - Näytä `lora_proof.py`-notebook ja säädä rank-liukusäädintä ($r = 1 \dots 6$).
    - Selitä omin sanoin, miten rank-arvo vaikuttaa approksimaatioon ja koulutettavien parametrien määrään sekä miten idea liittyy dimensiovähennykseen.

2.  **Keskeiset käsitteet:**

    - Kerro, millaisissa tilanteissa kielimallin hienosäätö on perusteltua verrattuna pelkkään kehotemuotoiluun tai RAG-ratkaisuihin.
    - Mitä tarkoittaa **kvantisoitu** malli?
    - Mitä eroa on **LoRA:lla** ja **QLoRA:lla**?

3.  **Koodin toiminta korkealla tasolla:**

    - Mitä `apply_chat_template` tekee?
    - Miltä alkuperäinen `dvgodoy/yoda_sentences` data näyttää (vinkki: Hugging Facen sivut)?
    - Mitä `SFTTrainer` tekee ja miten siihen liityy `SFTConfig`?

4.  **Koulutus ja MLflow-tulokset Roihussa:**

    - Käynnistä koulutus. Siinä kestää pari minuuttia; käytä tämä muiden vaiheiden selittämiseen.
    - Näytä koulutuksen jälkeen (Roihun) MLflow ja esittele koulutus- ja evaluointihäviöt.

5.  **GPU-muistin kulutuksen vertailu:**

    - Näytä MLflow:sta, kuinka paljon GPU-muistia (VRAM) LoRA-hienosäätö kulutti Roihussa.
    - Vertaa tätä lukemaa omalla koneellasi (tai opettajan koneella) ajettuun QLoRA-ajoon (~5,2 GB).
    - Pohdi kahden yllä olevan luvun eroavaisuuden syytä.

6.  **Inferenssi Roihussa:**

    - Näytä itse kirjoittamasi inferenssin Slurm-skripti.
    - Aja skripti ja näytä sen `slurm-<jobid>.out`-tuloste.
    - Vertaile tulosteen `Base`- ja `Yoda`-vastauksia.
    - Pohdi, miten mallin laatua voisi arvioida järjestelmällisesti oikeassa projektissa.

Kokonaisuutena videosta tulee ilmetä, että ymmärrät PEFT-, LoRA- ja QLoRA-hienosäädön peruskäsitteet, tunnistat kielimallin hienosäätöprosessin keskeiset vaiheet koodista sekä osaat ajaa hienosäädön ja inferenssin Roihussa.

## Yhteenveto

Kurssin päätteeksi ympyrä sulkeutuu: palaamme alkuviikoilta tuttuun tapaan erottaa koulutuksen asetukset itse koulutuskoodista. Meidän Yoda-skriptissä parametrit annetaan yksinkertaisuuden vuoksi komentoriviltä, mutta TRL:ssä on jako jo valmis: `SFTConfig` kokoaa hyperparametrit ja muut asetukset yhteen konfiguraatio-olioon, kun taas `SFTTrainer` huolehtii varsinaisesta koulutuksesta.

Sama perusratkaisu nähtiin jo viikoilla 1 ja 5, jolloin asetukset luettiin YAML-tiedostosta `OmegaConf`:n avulla. Tuolloin rakenne toteutettiin itse, mutta TRL tarjoaa sen valmiina osana kirjastoa. Laajemmassa projektissa asetukset voidaan lukea YAML-tiedostosta `TrlParser`-luokalla (`--config config.yaml`). Kokeiden välillä tarvitsee tällöin vaihtaa vain konfiguraatiota, ei muokata lähdekoodia. Tarvittaessa yksittäiset asetukset voidaan silti yliajaa komentoriviltä.

## Lähteet

[^moroneyFineTuningAI]: Moroney, L. _Fine-Tuning AI_. O'Reilly. 2026.
[^huyenAIEngineering]: Huyen, C. _AI Engineering_. O'Reilly. 2024.
[^nicoleTransformers]: Koenigstein, N. _Transformers: The Definitive Guide_. O'Reilly. 2026.
[^nicoleTransformers2025]: Koenigstein, N. _Transformers in Action_. Manning. 2025.
