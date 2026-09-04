# 1: Configuration-driven training

Tällä viikolla tutustumme [jani-public/flower-model-demo](https://gitlab.dclabra.fi/jani-public/flower-model-demo)-toteutukseen. Sinun tehtäväsi on kloonata repo, ottaa se käyttöön, tutustua sen rakenteeseen ja ajaa mallin koulutus paikallisesti. Kun mallin koulutus onnistuu:

* säädä hyperparametreja, kunnes mallin `best/f1_val`-metriikka on 90 % tai parempi.
* tämän jälkeen aja koulutus myös testidataa vasten.
* tallenna malli `models/`-hakemistoon.
* konvertoi malli ONNX-muotoon.
* ota käyttöön `www/`-hakemistossa oleva demo, joka käyttää ONNX-muotoista mallia.

## Kurssin repositorion alustus

Teet kaikki tämän kurssin tehtävät annettuun repositorioon, joka on luotu sinulle. Se löytyy esimerkiksi osoitteesta `gitlab.dclabra.fi/syvaoppiminen-2-2026/joannadurries`, jos on vuosi 2026, ja sinun nimi sattuu olemaan Joanna Durries. Seuraa seuraavia vaiheita:

## 1. Kloonaa kurssin repositorio

```bash
# Tee hakemisto sopivaan paikkaan, esimerkiksi:
mkdir -p ~/Code/syvaoppiminen-2-2026
cd ~/Code/syvaoppiminen-2-2026

# Varmista, että Git LFS on asennettu ja käytössä. Komento 
# on idempotentti, joten voit ajaa sen varmuuden vuoksi.
git lfs install

# Kloonaa kurssin repositorio
# LUE INFOLAATIKKO ALTA
```

!!! info "LUE MINUT"

    Tarkat komennot, jotka sinun tulee ajaa, voit kopioida suoraan GitLabista. Tyhjä repositoriosi sisältää ohjeen otsikolla **Create a new repository**. Aja kyseisen otsikon alla olevasta koodiblokista komennot. Komentojen ajamisen jälkeen voit päivittää GitLab:ssa näkymän, ja sinulla pitäisi näkyä ohjeiden tilalle repositorion sisältö (eli yksi tyhjä README.md-tiedosto). Tämän jälkeen voit jatkaa alla olevia ohjeita.

## 2. Tuo flower-model-demo repoosi

Kloonaa [jani-public/flower-model-demo](https://gitlab.dclabra.fi/jani-public/flower-model-demo) -repoosi ja tee tästä kopiosta sellainen, ettei se ole enää jatkossa Git-repositorio, vaan ihan tavallinen sinun omistama hakemisto. Alla komennot:

```bash
# Kloonaa opettajan repo
git clone https://gitlab.dclabra.fi/jani-public/flower-model-demo.git

# Tuhoa nestatun repositorion .git-hakemisto, jolloin se ei ole enää Git-repositorio
rm -rf flower-model-demo/.git
```

## 3. Ignooraa suuri datatiedosto

Tiedosto `flower-model-demo/gitlfs-store/flower_photos.tgz` on gzipattu tar-arkisto, joka on **218 MB** kokoinen. Jos 30 opiskelijaa tekee tästä kukin oman kopion, haaskaamme turhaan yli 6 GB yhteiskäyttöistä levytilaa. Siksi tämä tiedosto tulee merkata Git ignoreen siten, ettei sitä turhaan lisätä juuri sinun repositorioon. Löydät tiedoston jatkossa opettajan repositoriosta, jos sitä tarvitset. Lisää seuraava rivi repositoriosi `.gitignore`-tiedostoon:

```title="~/Code/syvaoppiminen-2-2026/yourname/.gitignore"
# Ignore large data file
flower-model-demo/gitlfs-store/flower_photos.tgz
```

!!! tip "Sama komentoriviltä"

    Jos haluat tehdä tämän komentoriviltä, voit ajaa seuraavat komennot:

    ```bash
    cd ~/Code/syvaoppiminen-2-2026/yourname
    echo '# Ignore large data file' >> .gitignore
    echo 'flower-model-demo/gitlfs-store/flower_photos.tgz' >> .gitignore
    ```

    Näitä komentoja ei tarvitse ajaa, jos muokkasit tiedosto esimerkiksi Visual Studio Codessa.

Jos haluat varmistaa, että hylkyehto (*engl. ignore rule*) on lisätty oikein, voit ajaa alla näkymän komennon, ja tarkistaa, että tiedosto ==ei ole== listassa:

```bash
git status -u
```

## 4. Puske uusi koodi remoteen

Nyt koodi on oikein paikoillaan, eikä isoja tiedostoja ole kylkiäisenä, joten voit puskea muutokset GitLab-repositorioosi:

```bash
git add .
git commit -m "Added flower-model-demo"
git push
```

## Videolla esitettävä

1. Näytä, että datasetti on purettu `data/{train,val,test,online}` -hakemistoihin.
2. Näytät, mistä YAML-konfiguraatiotiedosto löytyy.
3. Kommentoit lyhyesti, mitä hyperparametreja päädyit säätämään ja miksi.
4. Näytät MLflow:ssa vähintään 5 eri runia eri hyperparametreilla yleisnäkymässä.
5. Esittelet näistä tarkemmin 2 runia

     * näytät parametrit, ja
     * metriikat (erityisesti F1).

6. Näytät selaininferenssin (ONNX) toiminnassa.  

    Syötä sille `data/online/{{class}}/kuva.jpg`.

Kokonaisuutena videota katsomalla tulisi tulla selväksi, että sinä ymmärrät, mitä konfiguraatiovetoinen koulutus tarkoittaa, miten se on toteutettu tässä esimerkissä ja miten sitä käytetään.
