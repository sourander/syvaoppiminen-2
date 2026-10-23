# 4: Flower Model @ Roihu

!!! warning

    Tämä materiaali on toteutettu hyvin, hyvin nopeasti. Opettaja testaa tämän vielä läpi.

Tällä viikolla yhdistetään kolmen edellisen viikon opit. Viikolla 1 koulutit Flower Modelin omalla koneellasi, viikolla 2 tutustuit Roihuun ja viikolla 3 Slurmiin. Nyt sama Flower Model koulutetaan Roihun GPU-solmulla. Lopputuloksena koulutus käynnistyy yhdellä komennolla omalta koneeltasi, ja tulokset ovat nähtävissä Roihun verkkokäyttöliittymässä ajettavassa MLflow:ssa.

Viikon kokonaisuus koostuu seuraavista osista:

- Koodi synkronoidaan omalta koneelta Roihulle **GitLabin kautta** (`git push` paikallisesti, `git pull` Roihussa) eikä `scp`- tai `rsync`-komennoilla. Roihussa oleva klooni tehdään lukuoikeudet antavalla Project Access Tokenilla.
- Koulutus ajetaan Slurm-eräajona `gputest`-partitiolla. Datasetti puretaan työn aikana solmun paikalliseen `$TMPDIR`-hakemistoon suoraan Roihu Datasetistä.
- MLflow-data kirjoitetaan SQLite-tietokantaan projektin `/scratch`-levyalueelle, ja tuloksia tarkastellaan Roihun verkkokäyttöliittymän MLflow-sovelluksella.
- Koulutus käynnistetään omalta koneelta SSH-yhteyden yli (tai valinnaisesti **FirecREST**-rajapinnan kautta).

Opetusmateriaalia ovat:

- GitLab: [Project access tokens](https://docs.gitlab.com/user/project/settings/project_access_tokens/) ja [Clone repository using personal access token](https://docs.gitlab.com/user/profile/personal_access_tokens/#clone-repository-using-personal-access-token). Jälkimmäinen koskee henkilökohtaista tokenia, mutta kloonaus toimii samalla periaatteella myös projektin tokenilla.
- Viime viikolta tutut [Available batch job partitions](https://docs.csc.fi/computing/running/batch-job-partitions/) ja [CSC Computing Environment](https://csc-training.github.io/csc-env-eff/).
- Just-komentotyökalun käsikirja: [Just Programmer's Manual](https://just.systems/man/en/).
- CSC: jos teet valinnaisen FirecREST-osuuden, niin CSc:n dokumentaatio [FirecREST](https://docs.csc.fi/computing/firecrest/) sekä Roihun FirecREST-rajapinnan dokumentaatio: <https://api.roihu.csc.fi/v1/docs/> ovat tutustumisen arvoista sisältöä.

!!! info "Missä repositoriossa työskentelet?"

    Jatkat työskentelyä omassa kurssirepositoriossasi (esim. `syvaoppiminen-2-2026/joannadurries`), jonne toit viikolla 1 `flower-model-demo`-hakemiston. Alla olevissa esimerkeissä `<nimi>` tarkoittaa oman repositoriosi nimeä ja `project_XXXXXXX` oman CSC Student Projectisi numeroa. Korvaa nämä omillasi.

    Koska `flower_photos.tgz` on ignoorattu repositoriossasi (ks. viikko 1), Git LFS:ää ei tarvita Roihussa lainkaan. Datasetti luetaan Roihu Datasetistä `/dataset/project_2021217/flower_photos.tgz`.

## Työvaiheet

### 1. Gitignore

Tällä viikolla syntyy tiedostoja, joiden ei kuulu päätyä Gitiin: salaisuuksia sisältävä `.env` sekä Roihun ajojen sivutuotteet. Lisää `flower-model-demo/.gitignore`-tiedostoon seuraavat rivit:

```title="flower-model-demo/.gitignore"
# Secrets and Roihu run artifacts
.env
mlruns/
slurm-*.out
```

### 2. Ympäristömuuttujat Roihussa

Roihun polut ja muut asetukset kannattaa kerätä yhteen tiedostoon, joka ladataan automaattisesti jokaisen kirjautumisen yhteydessä. Kirjaudu Roihuun ja luo kotihakemistoosi tiedosto `~/.syvaoppiminen2_env` esimerkiksi `nano`- tai `vim`-editorilla. Aseta sisällöksi:

```bash title="~/.syvaoppiminen2_env"
export DL2_PROJ=/projappl/project_XXXXXXX
export DL2_SCRATCH=/scratch/project_XXXXXXX
export DL2_DATASET=/dataset/project_2021217
```

Huomaa, että `DL2_DATASET` on kaikille sama, koska datasetti on jaettu kurssin yhteisestä Roihu Dataset -projektista. Kaksi muuta osoittavat sinun omaan projektiisi.

Lisää seuraavaksi `.bashrc`-tiedostoon rivi, joka lataa yllä olevan tiedoston. Alla oleva komento lisää rivin vain, jos sitä ei vielä ole, joten sen voi ajaa turvallisesti useamman kerran:

```bash title="CSC Roihu"
grep -qF '.syvaoppiminen2_env' ~/.bashrc || echo '[ -f ~/.syvaoppiminen2_env ] && source ~/.syvaoppiminen2_env' >> ~/.bashrc
source ~/.bashrc
```

Tarkista, että muuttujat ovat käytössä:

```bash title="CSC Roihu"
echo "$DL2_PROJ"
echo "$DL2_SCRATCH"
echo "$DL2_DATASET"
```

Jatkossa, jos muutat tiedostoa `~/.syvaoppiminen2_env`, aja `source ~/.syvaoppiminen2_env` tai kirjaudu uudelleen sisään, jotta muutokset tulevat voimaan.

### 3. Project Access Token

Roihun tulee pystyä kloonaamaan ja päivittämään yksityinen repositoriosi. SSH-avaimesi ei ole Roihussa, eikä sitä sinne kuulukaan kopioida. Sen sijaan luodaan repositoriolle oma Project Access Token, joka antaa ainoastaan oikeuden lukea repositoriota.

Luo token oman repositoriosi asetuksista GitLabissa. Tutustu ensin GitLabin [Project access tokens](https://docs.gitlab.com/user/project/settings/project_access_tokens/) -ohjeeseen. Tokenille on seuraavat vaatimukset:

- Nimi kertoo käyttötarkoituksen, esimerkiksi `roihu-clone`.
- Voimassaoloaika ulottuu vähintään kurssin loppuun.
- Rooli on `Developer`.
- Scopeista valitaan **ainoastaan** `read_repository`.

Kopioi token heti talteen, sillä GitLab näyttää sen vain kerran. Jos token pääsee hukkumaan, luo uusi.

!!! warning "Miksi vain lukuoikeus?"

    Kloonauksen jälkeen token on tallessa Roihussa olevan kloonin `.git/config`-tiedostossa remoten osoitteessa. Pidä huoli siitä, että token voi korkeintaan lukea yhtä repositoriota. Tällöin mahdollisen vuodon vahingot jäävät pieniksi. Koodia ei koskaan pusketa Roihusta: muutokset tehdään omalla koneella ja pusketaan sieltä SSH-avaimella.

Lisää tokenin tiedot Roihussa `~/.syvaoppiminen2_env`-tiedoston loppuun:

```bash title="~/.syvaoppiminen2_env"
export GITLAB_HOST=gitlab.dclabra.fi
export GITLAB_PATH=syvaoppiminen-2-2026/<nimi>
export GITLAB_TOKEN=glpat-xxxxxxxxxxxxxxxxxxxx
```

Lataa tiedosto uudelleen:

```bash title="CSC Roihu"
source ~/.syvaoppiminen2_env
```

### 4. Kloonaa repositorio Roihuun

Koodi ja skriptit kuuluvat projektin `projappl`-levyalueelle. Kloonaa repositoriosi viikkokohtaiseen hakemistoon:

```bash title="CSC Roihu"
mkdir -p $DL2_PROJ/04_flower_at_roihu
cd $DL2_PROJ/04_flower_at_roihu

git clone "https://oauth2:${GITLAB_TOKEN}@${GITLAB_HOST}/${GITLAB_PATH}.git"
```

Flower Model löytyy nyt hakemistosta `$DL2_PROJ/04_flower_at_roihu/<nimi>/flower-model-demo`.

Jatkossa päivittäinen työnkulku on seuraava:

```bash
# Oma kone: muokkaa, commitoi, puske
cd ~/Code/syvaoppiminen-2-2026/<nimi>
git add .
git commit -m "..."
git push

# Roihu: päivitä klooni
cd $DL2_PROJ/04_flower_at_roihu/<nimi>
git pull
```

Myöhemmin vaiheessa 10 viimeinenkin osa automatisoidaan niin, ettei sinun tarvitse kirjautua Roihuun lainkaan.

### 5. Roihu MLflow

Roihun verkkokäyttöliittymä osoitteessa <https://www.roihu.csc.fi> tarjoaa useita interaktiivisia sovelluksia, jotka käynnistyvät laskentasolmulle ja avautuvat selaimeen. Yksi näistä on MLflow.

![Roihu Control Panel Interactive Apps](../images/04-interactive-apps.png)

**Kuva 1:** _Roihussa on useita interaktiivisia sovelluksia, jotka voi ajaa suoraan sivustolta `roihu.csc.fi` verkkoselaimesta käsin. Me käynnistämme palvelun MLflow._

Tärkeää on ymmärtää, että tämä MLflow on ainoastaan **käyttöliittymä**, ei tracking-palvelin. Koulutusskripti ei ota siihen yhteyttä verkon yli, kuten viikolla 1 kontissa pyörivään MLflow:iin. Sen sijaan koulutustyö kirjoittaa MLflow-datan suoraan SQLite-tietokantaan projektin `/scratch`-levyalueelle, ja käyttöliittymä lukee samaa tiedostoa. MLflow-sovelluksen ei siis tarvitse olla käynnissä koulutuksen aikana.

1. Kirjaudu osoitteeseen <https://www.roihu.csc.fi> (MFA vaaditaan).
2. Valitse **Apps > MLflow**.
3. Täytä käynnistyslomake:
    - Oletusarvot ovat pääosin kunnossa.
    - Valitse moduuliksi `python-pytorch/2.13`, jotta MLflow-versio on sama kuin koulutustyössä. Muuten tietokannan skeeman versiot voivat olla ristiriidassa.
    - Aseta MLflow-tiedostojen sijainniksi `sqlite:////scratch/project_XXXXXXX/mlflow/mlruns.db`.
4. Rastita **Save settings** ja anna asetuksille nimi, esimerkiksi `flowermodel`. Seuraavalla kerralla lomaketta ei tarvitse täyttää uudestaan.
5. Paina **Launch**. Odota hetki, ennen kuin painat **Connect to MLflow**, koska sovelluksen käynnistyminen kestää jonkin aikaa.

Tietokantaa ei ole vielä olemassa, joten näkymä on tyhjä. Tämä muuttuu vaiheessa 8. Sulje sessio, kun et enää tarvitse sitä, jotta se ei turhaan varaa resursseja.

!!! tip

    Huomaa kauttaviivojen määrä SQLite-osoitteessa. Selvitä, miksi absoluuttiseen polkuun tarvitaan neljä kauttaviivaa.

### 6. Datan valmistelu `$TMPDIR`-hakemistoon

Skriptissä `scripts/prepare_flower_photos.py` arkiston polku ja kohdehakemisto ovat kovakoodattuja:

```python
TAR_FILE = Path("gitlfs-store/flower_photos.tgz")
OUT_DIR = Path("data/flower_photos")
```

Roihussa arkisto luetaan Roihu Datasetistä, ja purettu datasetti kirjoitetaan työn aikaiseen `$TMPDIR`-hakemistoon. Muokkaa skriptiä siten, että se ottaa vastaan kaksi komentoriviargumenttia:

| Argumentti | Oletusarvo                       | Tarkoitus                                     |
| ---------- | -------------------------------- | --------------------------------------------- |
| `--tar`    | `gitlfs-store/flower_photos.tgz` | Polku `flower_photos.tgz`-arkistoon           |
| `--out`    | `data/flower_photos`             | Kohdehakemisto jaetulle (split) datasetille   |

Oletusarvojen ansiosta paikallinen käyttö (`uv run python scripts/prepare_flower_photos.py`) toimii täsmälleen kuten ennenkin. Varmista tämä myös kokeilemalla. Pythonin standardikirjaston `argparse`-moduuli riittää tähän.

!!! tip

    Viikon 3 materiaalissa käsiteltiin, miksi suurta määrää pieniä tiedostoja ei kannata käsitellä suoraan jaetulta Lustre-levyalueelta. Flower-datasetissä on tuhansia pieniä JPEG-tiedostoja, joten sama periaate pätee tähänkin.

### 7. Slurm-työskripti `scripts/roihu_train.sh`

Kirjoita uusi Slurm-työskripti `scripts/roihu_train.sh`, joka suorittaa koulutuksen Roihun GPU-solmulla. Kirjoitit vastaavan skriptin jo viikolla 3, joten valmista skriptiä en anna. Skriptin tulee täyttää seuraavat vaatimukset:

- Laskutetaan omaa projektiasi. Partitio on `gputest`, ja varataan yksi GH200-GPU. Mieti, montako CPU-ydintä kannattaa varata. Vinkki: katso konfiguraatiotiedoston `data.num_workers`.
- Skripti tarkistaa, että `DL2_PROJ`, `DL2_SCRATCH` ja `DL2_DATASET` on asetettu, ja keskeyttää selkeällä virheilmoituksella, jos niitä ei ole.
- Ohjelmistoympäristöksi ladataan moduuli `python-pytorch/2.13`.
- `flowermodel`-pakettia ei ole asennettu moduuliin. Selvitä, miten saat Pythonin löytämään sen kloonin `src/`-hakemistosta. Vinkki: `PYTHONPATH`.
- Esikoulutetut painot ladataan verkosta. Ohjaa ne talteen `/scratch`-levyalueelle eikä kotihakemistoon. Vinkki: `TORCH_HOME`.
- Datasetti valmistellaan `$TMPDIR`-hakemistoon edellisen vaiheen `prepare_flower_photos.py`-skriptillä.
- Koulutus käynnistetään `srun`-komennolla viikolla 1 tutuksi tulleella `--override`-mekanismilla. Konfiguraatiotiedostoon ei tehdä muutoksia, vaan Roihu-kohtaiset arvot annetaan komentoriviltä. Ylikirjoitettavat avaimet ovat:
    - `data.base_dir` osoittaa `$TMPDIR`-hakemistoon valmisteltuun datasettiin.
    - `tracking.url` osoittaa SQLite-tietokantaan `$DL2_SCRATCH/mlflow/mlruns.db`.
    - `tracking.experiment_name` on oma, esimerkiksi `FlowerModel_MobileNetV3_S_Roihu`, jotta Roihun ajot erottuvat.
    - `tracking.best_model_path` osoittaa `/scratch`-levyalueelle.
- Skripti luo tarvittavat hakemistot (`mkdir -p`) ennen kuin mitään kirjoitetaan niihin.

Skriptin pitää toimia myös vaiheessa 10, jossa työ lähetetään FirecREST-rajapinnan kautta. Siinä tilanteessa työ ei käynnisty kirjautumissolmun shellistä, minkä vuoksi tarvitaan kaksi lisäpalaa, joiden keksiminen itse olisi kohtuutonta.

**Kloonin sijainti ja koodin päivitys.** FirecREST-työ kertoo kloonin sijainnin `FLOWER_CLONE`-muuttujassa ja pyytää päivittämään koodin muuttujalla `ROIHU_PULL=1`. Käsin `sbatch`-komennolla lähetetyssä työssä käytetään hakemistoa, josta työ lähetettiin:

```bash
CLONE="${FLOWER_CLONE:-${SLURM_SUBMIT_DIR:-}}"
: "${CLONE:?FLOWER_CLONE or SLURM_SUBMIT_DIR must be set}"

# FirecREST submissions set ROIHU_PULL=1 to sync code from GitLab first.
if [[ "${ROIHU_PULL:-0}" == "1" ]]; then
    git -C "$CLONE" pull --ff-only
fi
```

**Moduulijärjestelmän alustus.** FirecREST-työ ei peri `module`-komentoa kirjautumisshellistä. Ilman alla olevaa lohkoa työ kaatuu heti virheeseen `module: command not found`. Lohko kuuluu ennen `module load` -riviä:

```bash
# FirecREST jobs and other non-interactive shells do not inherit the module
# system from the login shell, so initialize CSC's environment when needed.
# The profile script is not `set -u` clean, hence the temporary `set +u`.
if ! command -v module >/dev/null 2>&1; then
    export CSC_ENV_INIT_NON_INTERACTIVE=yes
    set +u
    source /etc/profile.d/zz-csc-env.sh
    set -u
fi
```

Rivit `set +u` ja `set -u` ovat tarpeen vain, jos skriptisi alussa on `set -euo pipefail` (suositeltavaa). Selvitä, mitä kukin näistä optioista tekee.

### 8. Ensimmäinen ajo käsin

Puske muutokset omalta koneeltasi, päivitä klooni Roihussa ja lähetä työ jonoon:

```bash title="CSC Roihu"
cd $DL2_PROJ/04_flower_at_roihu/<nimi>
git pull

cd flower-model-demo
sbatch scripts/roihu_train.sh
squeue --me
```

Kun työ on valmis, tarkastele sitä viikolta 3 tutuilla komennoilla (`seff`, `sacct`) ja lue tuloste `slurm-<jobid>.out`. Avaa sitten vaiheen 5 MLflow-sessio. Experimentin `FlowerModel_MobileNetV3_S_Roihu` (tai oman nimesi) alta pitäisi löytyä ajo parametreineen, metriikoineen ja artefakteineen.

!!! tip "Valinnainen: oma virtuaaliympäristö"

    Kirjoitushetkellä `python-pytorch/2.13`-moduulissa on kaikki Flower Modelin tarvitsemat paketit, myös `omegaconf`. Jos ajo kuitenkin kaatuu puuttuvaan importiin, luo kerran kirjautumissolmulla virtuaaliympäristö, joka käyttää moduulin paketteja ja johon asennetaan vain puuttuvat:

    ```bash title="CSC Roihu"
    module load python-pytorch/2.13
    cd $DL2_SCRATCH

    # Keep pip's cache on scratch instead of the small home directory
    export PIP_CACHE_DIR="$PWD/pip-cache"

    python -m venv --system-site-packages .venv
    source .venv/bin/activate
    pip install <puuttuva-paketti>

    # Clean up the cache to save CSC disk space
    rm -rf "$PIP_CACHE_DIR"
    ```

    Tällöin Slurm-skriptissä aktivoidaan `module load` -rivin jälkeen myös tämä ympäristö. Älä tee tätä, ellei se ole tarpeen.

### 9. Versioiden yhteensovitus

Paikallisessa `uv`-projektissa ja Roihun moduulissa voi olla eri versiot samoista paketeista. Eri MLflow-versiot voivat esimerkiksi tuottaa yhteensopimattomia tietokantoja, ja eri PyTorch-versiot voivat tuottaa eri tuloksia. Roihun moduulin versioihin ei voi vaikuttaa, joten on helpompaa sovittaa paikallinen ympäristö Roihun mukaiseksi.

Tallenna moduulin pakettiluettelo tiedostoon:

```bash title="CSC Roihu"
module load python-pytorch/2.13
pip freeze > ~/pip-freeze-python-pytorch-213
```

Etsi luettelosta (esim. `grep`-komennolla) niiden pakettien versiot, jotka ovat `pyproject.toml`-tiedostosi `dependencies`-listassa, ja kiinnitä ne paikallisesti:

```bash title="Oma kone"
uv add <paketti>==<versio>
```

Muista myös, että paikallinen MLflow-palvelin ajetaan kontissa (`containers/mlflow/compose.yml`). Kontin imagen version kannattaa vastata Python-kirjaston versiota.

### 10. FirecREST ja Justfile (valinnainen)

!!! tip

    Tämä tehtävä on valinnainen, syventävä lisätehtävä, jolla on ainoastaan arvosanaa korottava vaikutus.

FirecREST on CSC:n REST-rajapinta Roihuun. Sen avulla voit lähettää Slurm-töitä, listata niitä ja lukea tiedostoja suoraan omalta koneeltasi ilman SSH-yhteyttä. Tutustu [CSC:n FirecREST-ohjeeseen](https://docs.csc.fi/computing/firecrest/) ja [rajapinnan dokumentaatioon](https://api.roihu.csc.fi/v1/docs/). Käytämme vain `gpu`-alijärjestelmää. Toinen Roihun alijärjestelmä on `cpu`.

#### 10.1 Henkilökohtainen token

1. Luo token osoitteessa <https://my.csc.fi/firecrest-token>.
2. Kopioi token heti. Se näytetään vain kerran ja on voimassa 24 tuntia.
3. Kirjoita token omalla koneellasi tiedostoon `flower-model-demo/.env`, jonka lisäsit `.gitignore`-tiedostoon vaiheessa 1:

```bash title="flower-model-demo/.env"
FIRECREST_PAT=...
```

Testaa tokenia `curl`-komennolla. Tämä ei vaadi vielä `just`-työkalua:

```bash title="Oma kone"
source .env
curl https://api.roihu.csc.fi/v1/compute/gpu/jobs -H "Authorization: Bearer ${FIRECREST_PAT}"
```

Vastauksena pitäisi tulla JSON-muotoinen listaus töistäsi.

#### 10.2 Justfile

`flower-model-demo`-hakemistossa on valmiina `Justfile`, jossa on reseptit `flow` ja `noflow` paikallisen MLflow:n käynnistämiseen ja pysäyttämiseen. Korvaa sen sisältö alla olevalla. Muuta `roihu_clone`-muuttuja osoittamaan omaan klooniisi.

??? example "Justfile"

    ```just title="flower-model-demo/Justfile"
    # Load .env (FIRECREST_PAT) from this directory
    set dotenv-load := true

    # The FireCREST recipes below are Python, executed with `uv run --script`
    set script-interpreter := ['uv', 'run', '--script']

    # Show available commands by default
    default:
        @just --list

    # Roihu paths; edit if the project number or directory changes.
    roihu_clone := "/projappl/project_XXXXXXX/04_flower_at_roihu/<nimi>/flower-model-demo"
    firecrest_url := "https://api.roihu.csc.fi/v1"

    # Start local MLflow
    flow:
        docker compose -f containers/mlflow/compose.yml up -d

    # Stop local MLflow
    noflow:
        docker compose -f containers/mlflow/compose.yml down

    # Submit pull + train to Roihu via FireCREST (requires FIRECREST_PAT in .env)
    [script]
    roihu-submit:
        import json, os, subprocess, sys, urllib.error, urllib.request

        if "FIRECREST_PAT" not in os.environ:
            sys.exit("FIRECREST_PAT is not set - create a token at https://my.csc.fi/firecrest-token and add it to .env")

        def git(*args):
            return subprocess.run(["git", *args], capture_output=True, text=True)

        if git("status", "--porcelain").stdout.strip():
            print("Working tree is dirty. Commit your changes first:", file=sys.stderr)
            print(git("status", "--short").stdout, file=sys.stderr, end="")
            sys.exit(1)
        if git("rev-parse", "--verify", "-q", "@{u}").returncode:
            sys.exit("No upstream branch. Push first: git push -u origin HEAD")
        if git("rev-parse", "HEAD").stdout != git("rev-parse", "@{u}").stdout:
            sys.exit("Local commits are not pushed. Run 'git push' first.")

        clone = "{{roihu_clone}}"
        with open("scripts/roihu_train.sh", encoding="utf-8") as handle:
            script = handle.read()

        payload = {"job": {
            "name": "flower-roihu",
            "workingDirectory": clone,
            "standardOutput": f"{clone}/slurm-flower.out",
            "env": {"FLOWER_CLONE": clone, "ROIHU_PULL": "1"},
            "script": script,
        }}
        request = urllib.request.Request(
            "{{firecrest_url}}/compute/gpu/jobs",
            data=json.dumps(payload).encode(),
            headers={
                "Authorization": "Bearer " + os.environ["FIRECREST_PAT"],
                "Content-Type": "application/json",
            },
        )
        try:
            with urllib.request.urlopen(request) as response:
                job_id = json.load(response)["jobId"]
        except urllib.error.HTTPError as error:
            sys.exit(f"FireCREST error {error.code}: {error.read().decode().strip()}")
        print(f"submitted job {job_id}")
        print("follow with: just roihu-jobs, then: just roihu-log")

    # List Roihu GPU jobs from the last 24 hours
    [script]
    roihu-jobs:
        import json, os, sys, urllib.error, urllib.request

        if "FIRECREST_PAT" not in os.environ:
            sys.exit("FIRECREST_PAT is not set - create a token at https://my.csc.fi/firecrest-token and add it to .env")

        request = urllib.request.Request(
            "{{firecrest_url}}/compute/gpu/jobs?time_window=24h",
            headers={"Authorization": "Bearer " + os.environ["FIRECREST_PAT"]},
        )
        try:
            with urllib.request.urlopen(request) as response:
                jobs = json.load(response).get("jobs") or []
        except urllib.error.HTTPError as error:
            sys.exit(f"FireCREST error {error.code}: {error.read().decode().strip()}")

        for job in jobs:
            elapsed = "?"
            for task in job.get("tasks") or []:
                value = (task.get("time") or {}).get("elapsed")
                if value is not None:
                    elapsed = str(value)
            state = (job.get("status") or {}).get("state", "?")
            print(f"{job['jobId']:>8}  {job.get('name', ''):<28} {state:<10} {elapsed:>5}s  {job.get('partition', '')}")

    # Show the tail of the latest Roihu training log
    [script]
    roihu-log lines="100":
        import json, os, sys, urllib.error, urllib.parse, urllib.request

        if "FIRECREST_PAT" not in os.environ:
            sys.exit("FIRECREST_PAT is not set - create a token at https://my.csc.fi/firecrest-token and add it to .env")

        path = "{{roihu_clone}}/slurm-flower.out"
        query = urllib.parse.urlencode({"path": path, "lines": "{{lines}}"})
        request = urllib.request.Request(
            f"{{firecrest_url}}/filesystem/gpu/ops/tail?{query}",
            headers={"Authorization": "Bearer " + os.environ["FIRECREST_PAT"]},
        )
        try:
            with urllib.request.urlopen(request) as response:
                print(json.load(response)["output"]["content"], end="")
        except urllib.error.HTTPError as error:
            sys.exit(f"no log yet at {path} (FireCREST error {error.code})")

    # Cancel a Roihu job by id (useful when a job hangs at 0 % utilisation)
    [script]
    roihu-cancel job_id:
        import os, sys, urllib.error, urllib.request

        if "FIRECREST_PAT" not in os.environ:
            sys.exit("FIRECREST_PAT is not set - create a token at https://my.csc.fi/firecrest-token and add it to .env")

        job_id = "{{job_id}}"
        request = urllib.request.Request(
            f"{{firecrest_url}}/compute/gpu/jobs/{job_id}",
            method="DELETE",
            headers={"Authorization": "Bearer " + os.environ["FIRECREST_PAT"]},
        )
        try:
            with urllib.request.urlopen(request) as response:
                pass
        except urllib.error.HTTPError as error:
            sys.exit(f"FireCREST error {error.code}: {error.read().decode().strip()}")
        print(f"cancelled job {job_id}")
    ```

Roihu-reseptit ovat Python-skriptejä, jotka ajetaan `uv run --script` -komennolla (ks. `set script-interpreter`). Ne käyttävät vain Pythonin standardikirjastoa, joten muita riippuvuuksia ei tarvita. Jos `just` valittaa `[script]`-attribuutista, päivitä `just` tuoreempaan versioon.

| Resepti                     | Mitä tekee                                                                                       |
| --------------------------- | ------------------------------------------------------------------------------------------------ |
| `just roihu-submit`         | Tarkistaa, että kaikki muutokset on commitoitu ja puskettu, ja lähettää koulutustyön FirecRESTiin |
| `just roihu-jobs`           | Listaa Roihu-työsi viimeisen 24 tunnin ajalta                                                    |
| `just roihu-log [lines]`    | Näyttää viimeisimmän ajon tulosteen (`slurm-flower.out`) lopun, oletuksena 100 riviä             |
| `just roihu-cancel <jobid>` | Peruu työn                                                                                       |

Tutustu `roihu-submit`-reseptiin tarkasti ja selvitä:

- Miksi resepti kieltäytyy toimimasta, jos työhakemistossa on commitoimattomia muutoksia tai puskemattomia committeja?
- Paikallinen `scripts/roihu_train.sh` lähetetään rajapinnalle `script`-kentässä. Mitä etua tästä on verrattuna siihen, että Roihussa ajettaisiin kloonissa jo valmiiksi olevaa skriptiä?
- Mihin rajapinnan kenttiä `workingDirectory`, `standardOutput` ja `env` käytetään? Löydät vastaukset rajapinnan dokumentaatiosta.

Huomaa myös, että työn sisällä **ei** ajeta `sbatch`-komentoa. FirecREST lähettää skriptin itse eräajona, joten koulutus ajetaan suoraan lähetetyssä skriptissä.

Tyypillinen työnkulku on jatkossa:

```bash title="Oma kone"
cd ~/Code/syvaoppiminen-2-2026/<nimi>/flower-model-demo
git add .
git commit -m "..."
git push
just roihu-submit
just roihu-jobs
just roihu-log
```

!!! warning "Tunnettuja ongelmia"

    - Token vanhenee 24 tunnissa. Kun reseptit alkavat epäonnistua virheellä `FireCREST error 401`, luo uusi token ja päivitä `.env`.
    - Roihussa koulutustyö on muutaman kerran jumittunut heti käynnistyttyään: MLflow:iin ilmestyy ajo, mutta GPU:n ja CPU:n käyttöaste pysyy nollassa, eikä metriikoita kirjoiteta. Tällöin peru työ ja lähetä se uudelleen:

        ```bash
        just roihu-cancel <jobid>
        just roihu-submit
        ```

### 11. Toistettavuuskoe

Flower Modelin konfiguraatiossa on `seed`, ja koulutus kutsuu `seed_everything`-funktiota. Selvitä kokeellisesti, kuinka toistettavia tulokset todella ovat:

1. Aja sama konfiguraatio kahdesti peräkkäin paikallisesti.
2. Aja sama konfiguraatio kahdesti peräkkäin Roihussa.
3. Vertaa metriikoita saman laitteiston ajojen kesken.
4. Vertaa metriikoita eri laitteistojen ajojen kesken.

Onnistuiko koodi olemaan toistettava (1) samalla laitteistolla ja (2) millä tahansa laitteistolla? Jos sinulla ei ole Nvidian GPU:ta, pyydä vertailutulos kurssikaverilta, jolla on. Pohdi, mistä mahdolliset erot johtuvat.

## Videolla esitettävä

1. Selitä kokonaisuuden työnkulku: mihin GitLabia, Project Access Tokenia, Slurmia, FirecRESTiä ja Roihun MLflow-sovellusta käytetään.
    - Perustele, miksi Project Access Tokenilla on vain lukuoikeus.

2. Näytä Roihussa `~/.syvaoppiminen2_env`-tiedosto ilman tokenin arvoa sekä klooni `projappl`-levyalueella.
    - Perustele, miksi koodi on `projappl`-alueella, MLflow-tietokanta ja mallit `/scratch`-alueella ja purettu datasetti `$TMPDIR`-hakemistossa.

3. Esittele muutokset koodiin.
    - Näytä `prepare_flower_photos.py`-skriptin uudet argumentit ja osoita, että paikallinen käyttö toimii edelleen.
    - Esittele `roihu_train.sh`: resurssivaraus, ohjelmistoympäristö, datan valmistelu ja `--override`-arvot.

4. Lähetä työ käsin `sbatch`-komennolla ja tarkastele valmistunutta työtä `seff`-komennolla.

5. Näytä ajo Roihun MLflow-sovelluksessa: parametrit, metriikat (erityisesti F1) ja artefaktit.

6. Näytä `pyproject.toml`-tiedostosta kiinnitetyt versiot ja kerro, miten päädyit niihin.

7. (Valinnainen) Tee pieni muutos koodiin tai konfiguraatioon, commitoi ja puske se, ja käynnistä koulutus komennolla `just roihu-submit`.
    - Seuraa työtä komennoilla `just roihu-jobs` ja `just roihu-log`.
    - Selitä, mitä `roihu-submit` tarkistaa ennen lähettämistä ja miksi.

8. Esittele toistettavuuskokeen tulokset ja pohdi, mistä mahdolliset erot johtuvat.

Kokonaisuutena videosta tulee ilmetä, että osaat ajaa olemassa olevan koulutusprojektin supertietokoneella ilman, että koodi tai konfiguraatio haarautuu paikallisesta versiosta. Ymmärrät, miten koodi kulkee GitLabin kautta Roihuun, miten Slurm-työ käyttää eri levyalueita tarkoituksenmukaisesti ja miten koulutustulokset tallentuvat MLflow:iin. Osaat lisäksi käynnistää ja seurata koulutusta omalta koneeltasi FirecREST-rajapinnan kautta sekä huolehtia siitä, etteivät salaisuudet päädy versionhallintaan.
