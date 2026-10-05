# 2: CSC Roihu

On kovin tyypillistä, että koneoppimismalleja koulutetaan järjestelmällä, jossa ei ole graafista käyttöliittymää, ja joka voi olla monimutkainen ja vieras. Tämä viikko tutustuttaa sinut yhteen tällaiseen ympäristöön.

Tällä kurssilla käytetään CSC:n ympäristöä ja erityisesti Roihu-supertietokonetta. Keskeinen opetusmateriaali on CSC:n virallinen dokumentaatio, ja erityisesti koulutusmateriaali [CSC Computing Environment](https://csc-training.github.io/csc-env-eff/). Myös [Roihu disk areas](https://docs.csc.fi/computing/roihu-disk/) ja [Lustre file system](https://docs.csc.fi/computing/lustre/ ovat tärkeitä lähteitä). Tämän ensimmäisen viikon aihepiiriin kuuluuvat **CSC Computing Environment**-materiaalista:

- Basics:
  - CSC account and project
  - Setting up SSH keys
  - Login to Roihu
- Disk areas:s
  - Main disk areas (+ `lue`-komento)
- Module system:
  - Modules in Roihu

Lisäksi tutustutaan Roihun myötä tulleisiin muutoksiin, joista yksi tärkeimmistä on `ProjData`-levyalue. Entisten `Home`, `ProjAppl` ja `Scratch` -alueiden rinnalle on tullut `ProjData`, joka on tarkoitettu projektien hallinnoimien datasettien jakamiseen. Roihu Dataset -konseptissa datasetille voidaan luoda MyCSC-palvelussa erillinen Dataset Project. Kurssin opiskelijoilla on omat Student Projectinsa, mutta yhteistä aineistoa ei ole järkevää kopioida erikseen jokaisen opiskelijaprojektin levyalueelle. Dataset Project mahdollistaa aineiston hallitun jakamisen useiden projektien käyttöön.

SSH:n suhteen uutta on vaatimus, että sinulla on signed SSH key, joka on rekisteröity MyCSC:hen. Lue tästä tarkemmin [Setting up SSH keys](https://docs.csc.fi/computing/connecting/ssh-keys/#signing-public-key)-ohjeesta. Roihussa on uutta edellisiin verrattuna myös se, että CPU ja GPU noodeille on omat erilliset login-nodet.

## Roihu Dataset

Tällä kurssilla datasetti on jaettu opiskelijoille valmiiksi Roihu Dataset -projektin kautta. Tämä tarkoittaa, että yhteiset tiedostot näkyvät suoraan Roihun tiedostojärjestelmässä, eikä niitä tarvitse ladata erikseen verkon yli omaan kotihakemistoon. Tästä huolimatta CSC Allas -palvelu esitellään myös alempana tässä materiaalissa, koska se on yleinen tapa jakaa datasettiä tutkimusprojekteissa.

Kurssin datasetti löytyy hakemistosta: `/dataset/project_2021217/`. Huomaa, että tämä projektinumero on eri kuin opettajan kurssiprojekti tai sinun projektisi. Tämä on täysin oma, erillinen Roihu Dataset -projekti.

### Opettajan osuus

Projekti on opettajan hallinnoima, joten opettaja voi kirjoittaa siihen tiedostoja samalla tavalla kuin muihinkin levyalueisiin, joihin opettajalla on kirjoitusoikeus. Opettaja on ladannut datasetin `flower_photos.tgz`-paketin muodossa, joka on jaettu kaikille opiskelijoille. Opettaja voi ladata sen esimerkiksi seuraavalla tavalla:

```bash
# CSC Allas -palvelusta
module load allas
cd /dataset/project_2021217/
s3cmd get s3://flower-dataset/flower_photos.tgz
```

Tai, vaihtoehtoisesti, lähettäen sen omalta koneeltaan `scp`-komennolla:

```bash
# Omalla koneella
scp flower_photos.tgz roihu-gpu:/dataset/project_2021217/
```


### Opiskelijan osuus

Voit tarkastella hakemiston sisältöä ja paketin kokoa tavallisilla Linux-komennoilla:

```bash
# Listaa dataset-hakemiston sisältö
ls -lh /dataset/project_2021217/

# Kurkkaa paketin sisään (tulostaa tiedostojen määrän arkistossa)
tar tf /dataset/project_2021217/flower_photos.tgz | wc -l
```

!!! warning

    Ethän turhaan kopioi tai pura flower_photos.tgz-pakettia omaan kotihakemistoosi. Tämä tehdään myöhemmissä kurssin vaiheissa Slurm-skriptissä siten, että tiedostot puretaan ja ovat käytössä työtä suorittavan noodin paikallisessa työskentelyhakemistossa. Tällöin ne katoavat automaattisesti mennessään, kun Slurm-työ päättyy.

## Allas

### Opettajan osuus

CSC Allas on CSC:n tarjoama pilvipalvelu, *object storage*, joka muistuttaa kaupallista Amazon S3 -palvelua. Sitä voi käyttää useilla eri asiakassovelluksilla (ks. [Clients Comparison](https://docs.csc.fi/data/Allas/accessing_allas/#clients-comparison)), joista kenties sopivimman tarjonnan tuo S3cmd (ks. [The S3 Client](https://docs.csc.fi/data/Allas/using_allas/s3_client/)).

Opettaja on tehnyt setupin tätä kurssia varten seuraavilla komennoilla, jotka myötäilevät [Using Allas to host a data set for a research project](https://docs.csc.fi/data/Allas/allas_project_example/)-ohjetta:

```bash title="CSC Roihu"
module load allas
allas-conf project_2020238
```

Tämä operaatio kysyy salasanaa. Vaikka yleensä käytämme HAKA-tunnuksia, on tilanteita, joissa tulee olla myös perinteinen CSC-tunnuksen salasana. Sen voi luoda "Forgot password?"-linkistä MyCSC-sivulla. Oletaen, että syötit salasanan oikein, kotihakemistoosi syntyy tiedosto `.s3cfg`, joka sisältää seuraavanlaisen konfiguraation:

```toml title="~/.s3cfg"
[allas-project_2020238]
access_key   = 14ab79xxxxxxxxxxxxxxxxxxxxxxxxxx
secret_key   = adf913xxxxxxxxxxxxxxxxxxxxxxxxxx
host_base    = a3s.fi
host_bucket  = %(bucket)s.a3s.fi
human_readable_sizes = True
enable_multipart = True
signature_v2 = True
use_https = True
```

Tästä on aika jatkaa kotikoneella.

```bash title="Kotikone"
brew install rclone
rclone config
```

Interaktiivinen konfigurointi kysyy useita kysymyksiä, joiden vastaamisen seurauksena syntyy tiedosto `~/.config/rclone/rclone.conf`. Sen sisältö on seuraavanlainen:

```toml title="~/.config/rclone/rclone.conf"
[s3allas2026]
type = s3
provider = Other
access_key_id = 14ab79xxxxxxxxxxxxxxxxxxxxxxxxxx
secret_access_key = adf913xxxxxxxxxxxxxxxxxxxxxxxxxx
endpoint = a3s.fi
acl = private
```

Nyt on mahdollista luoda uusi bucket -- eli looginen, nimetty tiedostosäilö -- ja ladata sinne meidän `flower_photos.tgz`:

```bash title="Kotikone"
cd ~/Code/jani-public/flower-model-demo/gitlfs-store
rclone mkdir s3allas2026:flower-dataset
rclone copy flower_photos.tgz s3allas2026:flower-dataset
```

Tiedosto on nyt osoitteessa `https://a3s.fi/flower-dataset/flower_photos.tgz`, ja sen voisi ladata selaimella tai vaikka Swift/rclone-asiakasohjelmalla, jos pääsynhallinta eli ACL sallisi. No eipä salli! Muutetaan tilanne: tiedosto on nyt mahdollista jakaa joko valituille muille projekteille tai koko maailmalle julkiseksi. Alla kummatkin vaihtoehdot, joista jälkimmäinen on otettu käyttöön, koska dataset on CC-BY oikeuksin lisensoitu ja valmis jaettavaksi:

```bash title="CSC Roihu"
# Allas konfiguraatio
allas-conf

# Jakaminen yhdelle tietylle projektille
OTHER_PROJECT_UUID="3d5b0ae8e724b439a4cd16d1290"
s3cmd setacl --acl-grant=read:$OTHER_PROJECT_UUID s3://flower-dataset/flower_photos.tgz

# Bucket itsessään julkiseksi (listaus sallittu)
s3cmd setacl --acl-public s3://flower-dataset/

# Tiedosto (ja muutkin, jos niitä olisi) julkiseksi
s3cmd setacl --acl-public --recursive s3://flower-dataset/
```

!!! warning

    Julkinen S3-bucket ei ole sama asia kuin verkkopalvelimen julkinen hakemisto. Vaikka bucketin ACL sallii sen sisällön lukemisen ja listaamisen, osoite <https://a3s.fi/flower-dataset/> ei välttämättä näytä tiedostoluetteloa tavallisessa verkkoselaimessa.
    
    Swift-protokollassa julkinen container (eli *bucket* S3 terminologiassa) voidaan määrittää palauttamaan selaimelle tekstimuotoinen objektien luettelo. S3-protokollassa listaus tehdään S3-asiakasohjelmalla.

    Nämä ovat kaksi kilpailevaa protokollaa, joista Swift on siirtymässä pois käytöstä CSC:n palveluissa. Aiemmat supertietokoneet Puhti ja Mahti käyttivät `allas-conf`-vakiona Swiftiä. Roihu käyttää S3:sta.

### Opiskelijan osuus

Koska kurssin data on saatavilla suoraan Roihun `dataset`-levyalueelta, sinun ei tällä kurssilla tarvitse käyttää S3-asiakasohjelmia datan noutamiseen. On kuitenkin hyvä ymmärtää, miten julkiseksi jaettua S3-dataa voi hakea.

Jos haluat ladata tiedoston väliaikaisesti esimerkiksi omalle kotikoneellesi tai Roihun kotihakemistoon testailua varten, helpoin tapa julkisen datan kohdalla on käyttää tavallista verkkoselainta tai `curl`-komentoa. Julkisen tiedoston lataaminen onnistuu ilman monimutkaisia konfiguraatioita:

```bash
# Lataa CURL-asiakasohjelmalla
curl -O https://a3s.fi/flower-dataset/flower_photos.tgz

# Tarkista tiedostostojen määrä paketissa
tar tf flower_photos.tgz | wc -l

# Poista tiedosto, jos et enää tarvitse sitä
rm flower_photos.tgz
```

!!! tip

    Jos työurallasi päädyt käsittelemään ei-julkista dataa S3-yhteensopivissa palveluissa (kuten AWS S3 tai CSC Allas), käytetään tiedostojen siirtoon yleensä `s3cmd` tai `rclone` -ohjelmia, jotka vaativat autentikoinnin. Kannattaa varautumisen takia vähintään pintapuolisesti vilkaista AWS:n dokumentaatiota: [Bucket policies for Amazon S3](https://docs.aws.amazon.com/AmazonS3/latest/userguide/bucket-policies.html).


## Bonus: SSH-signeerauksen automatisointi

CSC:n omassa ohjeessa neuvotaan, kuinka SSH-signeeraus voidaan automatisoida heidän tarjoamalla apuriskriptillä. Näin sinun ei tarvitse joka kerta klikkailla MyCSC:ssä "Sign and download SSH certificate"-painiketta. Alla on ohjetta täydentävät vaiheet, joilla sen saa tavallisessa ==macOS/Linux-käyttöjärjestelmässä== helposti ajettavaksi. Ohje voi toimia myös Git Bashissä (Git for Windows), mutta opettaja ei ole testannut tätä. Symbolinen linkki (`ln -s`) voi aiheuttaa ongelmia Windows-ympäristössä.

```bash
# Ohjelma asennetaan käyttäjän kotihakemiston alle.
cd ~/.local/lib

# Kloonataan projekti
git clone https://github.com/CSCfi/certificate-helper-tool.git

# Tiedosto on jo ajettava, mutta varmistetaan se
chmod +x certificate-helper-tool/csc_cert.py

# Linkitetään ohjelma suoraan bin-hakemistoon
ACTUAL_LOC="$HOME/.local/lib/certificate-helper-tool/csc_cert.py"
PATH_LOC="$HOME/.local/bin/csc-cert-helper"
ln -s $ACTUAL_LOC $PATH_LOC

# Jatkossa voit ajaa sitä mistä tahansa hakemistosta näin:
csc-cert-helper --version

# Ja varsinainen signeeraus onnistuu näin:
csc-cert-helper -u jsourand
```

Korvaa yllä olevassa komennossa `jsourand` omalla CSC-käyttäjätunnuksellasi. Löydät sen MyCSC-portaalista. Komento tekee seuraavat:

1. Se avaa selaimen ja pyytää sinua kirjautumaan MyCSC:hen (HAKA-tunnuksella).
2. Se esittää sinulle 6-merkkisen koodin. Kopioi se leikepöydälle.
3. Liitä koodi terminaaliin ja paina Enter.
4. Se kysyy sinun SSH-passphrasea. Anna se ja paina Enter.

!!! tip "Entä jos on selainprofiileja?"

    Jos sinulla on useampi selainprofiili, kannattaa korvata symbolinen linkki pienellä shell-skriptillä, joka huolehtii selaimen profiilista. Esimerkiksi Chromella tämä hoituu niin, että avaat selaimen ko. profiilissa ja menet osoitteeseen `chrome://version`. Löydät **Profile Path** kohdasta profiilin nimen, kuten `Profile 1`. Tämän jälkeen seuraa ohjeita:

    ```bash
    # Tuhoa symbolinen linkki jos teit sen jo
    rm ~/.local/bin/csc-cert-helper

    # Luo shell srripti sen tilalle
    nvim ~/.local/bin/csc-cert-helper
    ```

    Aseta sisällöksi:

    ```bash
    #!/bin/bash

    # Force the specific browser profile just for this execution
    export BROWSER="google-chrome --profile-directory 'Profile 1' %s"

    # Execute the actual Python script and pass all command-line arguments to it
    exec ~/.local/lib/certificate-helper-tool/csc_cert.py "$@"
    ```

    Tee tiedostosta vielä ajettava ja aja se:

    ```bash
    chmod +x ~/.local/bin/csc-cert-helper
    csc-cert-helper -u jsourand
    ```

## Bonus: SSH Config

Voit helpottaa SSH-yhteyksien muodostamista jatkossa lisäämällä seuraavat rivit `~/.ssh/config`-tiedostoon:

```
Host roihu-cpu
    HostName roihu-cpu.csc.fi
    User jsourand
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes

Host roihu-gpu
    HostName roihu-gpu.csc.fi
    User jsourand
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes
```

Korvaa käyttäjätunnus `jsourand` omallasi. Tämän jälkeen voit kirjautua Roihuun yksinkertaisesti komennolla:

```bash
ssh roihu-gpu
```

## Videolla esitettävä

1. Näytät, että sinulla on oma CSC Student Project.
2. Kerrot lyhyesti, mikä on supertietokone jaettuna ympäristönä, ja erityisesti:
    - Mitä eroa on login ja compute nodeilla.
    - Miksi on tärkeää olla ajamatta raskasta laskentaa login nodella.
    - Selität, miksi CSC:n ympäristössä muut käyttäjät tulee ottaa huomioon, eikä esimerkiksi huvikseen tai laiskuuttaan jättää tuhoamatta turhaksi käyneitä tiedostoja.

3. Kirjaudut sisään SSH:lla, ja
    - Tulostat nykyisen työskentelyhakemiston.
    - Ajat komennont `csc-projects`.
    - Esittelet keskeiset levyalueet. Navigoi kuhunkin `cd`-komennolla.
    - Esittelet `ProjData`-levyalueen käyttötarkoituksen.
    - Näytät MyCSC-palvelussa, mistä uusi Dataset Project luotaisiin. Sinun ei tarvitse luoda projektia, vaikka sinulla saattaakin olla oikeudet siihen.
    - Lataat `flower_photos.tgz`-tiedoston kotihakemistoosi, tarkistat että se latautu oikein, ja poistat sen lopuksi. Voit ladata käyttäen sen joko:
        - CSC Allas -palvelua (esim. `curl` tai `s3cmd`), tai
        - Roihu Dataset -levyaluetta (esim. `cp`-komento).

4. Näytät, että tunnet moduulijärjestelmän:
    - Listaat moduulit.
    - Etsi moduuli, jossa esiintyy sana `pytorch`
    - Lataa tuorein pytorch-moduuli
    - Moduulin mukana tulee `uv`. Näytä komennon `uv self version` tuloste.

5. Näytät, että tunnet Lustren perusteet:
    - Selvitä, missä moduulissa tulee `lue`-binääri. Lataa moduuli.
    - Käytät `lue`-komentoa (esim. `lue ~`) ja selität, mitä se tekee.

6. Todistat, että CSC Allas on sinulle tuttu.
    - Esittelet, mikä CSC Allas on.
    - Selität, mitä object storage tarkoittaa.
    - Selität, mikä on S3 bucket.

Kokonaisuutena videosta tulee ilmetä, että ymmärrät CSC:n ajoympäristön perusrakenteen, osaat kirjautua palveluun SSH:lla, käyttää moduulijärjestelmää ja navigoida keskeisillä levyalueilla.

Sinun tulee ymmärtää Roihu Dataset -palvelun ja Dataset Projectin käyttötarkoitus, joiden kautta kurssin datasetti on jaettu keskitetysti kaikille opiskelijoille. Osaat paikantaa datasetin levyjärjestelmästä, ja lisäksi ymmärrät Allas-pilvitallennustilan roolin osana CSC-ympäristöä ladaten sieltä tiedoston väliaikaisesti ja poistaen tarpeettoman paikallisen kopion.

Ymmärrät, että CSC on jaettu ympäristö, minkä vuoksi sinulla on vastuu siitä, ettet kuormita järjestelmää tarpeettomasti etkä säilytä turhia tiedostokopioita.
