# 2: CSC Roihu

!!! warning

    Tämä materiaali on yhä hieman kesken. Opettaja odottaa CSC:ltä vahvistusta, että Roihu Dataset on luotu.

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

Lisäksi tutustutaan Roihun myötä tulleisiin muutoksiin, joista kenties tärkein on `ProjData`-alue. Entisten `Home`, `ProjAppl` ja `Scratch` kylkeen on tullut tuo uutukainen, joka on tarkoitettu **datasettien jakamiseen projektien kesken**. Tällä kurssilla kullakin opiskelijalla on oma student project, mutta olisi tilan haaskausta ladata `n_oppilasta` kertaa Flower Dataset. Siispä opettaja on luonut "dataset project":n MyCSC-sivulla. Sinun ei tarvitse omaa luoda, mutta sinun tulee tietää, mistä se luotaisiin jos tarvitsisit sitä.

SSH:n suhteen uutta on vaatimus, että sinulla on signed SSH key, joka on rekisteröity MyCSC:hen. Lue tästä tarkemmin [Setting up SSH keys](https://docs.csc.fi/computing/connecting/ssh-keys/#signing-public-key)-ohjeesta. Roihussa on uutta edellisiin verrattuna myös se, että CPU ja GPU noodeille on omat erilliset login-nodet.

## Allas

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

Nyt on mahdollista luoda uusi bucket -- eli looginen, nimetty tiedostsäilö -- ja ladata sinne meidän `flower_photo.tgz`:

```bash title="Kotikone"
cd ~/Code/jani-public/flower-model-demo/gitlfs-store
rclone mkdir s3allas2026:flower-dataset
rclone copy flower_photos.tgz s3allas2026:flower-dataset
```

Tiedosto on nyt osoitteessa `https://a3s.fi/flower-dataset/flower_dataset.tgz`, ja sen voisi ladata selaimella tai vaikka Swift/rclone-asiakasohjelmalla, jos pääsynhallinta eli ACL sallisi. No eipä salli! Muutetaan tilanne: tiedosto on nyt mahdollista jakaa joko valituille muille projekteille tai koko maailmalle julkiseksi. Alla kummatkin vaihtoehdot, joista jälkimmäinen on otettu käyttöön, koska dataset on CC-BY oikeuksin lisensoitu ja valmis jaettavaksi:

```bash title="CSC Roihu"
# Allas konfiguraatio (-k tallentaa salasanan ympäristömuuttujaan)
allas-conf

# Jakaminen yhdelle tietylle projektille
OTHER_PROJECT_UUID="3d5b0ae8e724b439a4cd16d1290"
s3cmd setacl --acl-grant=read:$OTHER_PROJECT_UUID s3://my_fishbucket

# Jakaminen koko maailmalle
s3cmd setacl --acl-public s3://fish-bucket
```

Jatkossa osoite [/v1/flower-dataset/](https://a3s.fi/swift/v1/flower-dataset/) palauttaa tiedostolistauksen ja tarkka tiedoston osoite [/v1/flower-dataset/flower_photos.tgz](https://a3s.fi/swift/v1/flower-dataset/flower_photos.tgz) palauttaa tiedoston. Jatkossa tiedoston voi ladata mihin tahansa projektiin Roihu-ympäristössä komennolla:

```bash
# Roihu
module load allas
s3cmd get s3://flower-dataset/flower_photos.tgz
```

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
ssh roihu
```

## Videolla esitettävä

1. Näytät, että sinulla on oma CSC student project.
2. Kerrot lyhyesti, mikä on supertietokone jaettuna ympäristönä, ja erityisesti:
   - Mitä eroa on login ja compute nodeilla.
   - Miksi on tärkeää olla ajamatta raskasta laskentaa login nodella.
   - Selität, miksi CSC:n ympäristössä muut käyttäjät tulee ottaa huomioon, eikä esimerkiksi huvikseen tai laiskuuttaan jättää tuhoamatta turhaksi käyneitä tiedostoja.

3. Kirjaudut sisään SSH:lla, ja
   - Tulostat nykyisen työskentelyhakemiston.
   - Ajat komennont `csc-projects`.
   - Esittelet keskeiset levyalueet. Navigoi kuhunkin `cd`-komennolla.
   - Näytät, mistä löytyy `flower_photos.tgz`-tiedosto (Roihu Dataset Project).

4. Näytät, että tunnet moduulijärjestelmän:
   - Listaat moduulit.
   - Etsi moduuli, jossa esiintyy sana `pytorch`
   - Lataa tuorein pytorch-moduuli
   - Moduulin mukana tulee `uv`. Näytä komennon `uv self version` tuloste.

5. Näytät, että tunnet Lustren perusteet:
   - Selvitä, missä moduulissa tulee `lue`-binääri. Lataa moduuli.
   - Käytät `lue`-komentoa (esim. `lue ~`) ja selität, mitä se tekee.

6. Kerrot lyhyesti, mikä Allas on ja mihin sitä käytetään.

Kokonaisuutena videosta tulee ilmetä, että ymmärrät CSC:n ajoympäristön perusrakenteen, osaat kirjautua palveluun SSH:lla, osaat käyttää moduulijärjestelmää, osaat navigoida hakemistoissa ja tiedät, mistä data ja resurssit löytyvät. Ymmärrät, että CSC on jaettu ympäristö, ja sinulla on tämän vuoksi vastuu siitä, että et kuormita järjestelmää turhaan.
