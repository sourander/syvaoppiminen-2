# 2: CSC Roihu

!!! warning "TODO"

    Tämä materiaali on täysin kesken. Kirjoitushetkellä (June 2026) CSC Roihu on yhä vain pilottikäyttäjille avoinna.


On kovin tyypillistä, että koneoppimismalleja koulutetaan järjestelmällä, jossa ei ole graafista käyttöäliittymää, ja joka voi olla monimutkainen ja vieras. Tämä viikko tutustuttaa sinut yhteen tällaiseen ympäristöön.

Tällä kurssilla käytetään CSC:n ympäristöä ja erityisesti Roihu-supertietokonetta. Keskeinen opetusmateriaali on CSC:n virallinen dokumentaatio, ja erityisesti koulutusmateriaali [CSC Computing Environment](https://csc-training.github.io/csc-env-eff/). Tämän ensimmäisen viikon aihepiiriin kuuluuvat:

* Basics: 
    * CSC account and project
    * Setting up SSH keys
    * Login to Roihu
* Disk areas:
    * Main disk areas (+ `lue`-komento)
* Module system:
    * Modules in Roihu

Lisäksi tutustutaan Roihun myötä tulleisiin muutoksiin, joista kenties tärkein on `ProjData`-alue. Entisten `Home`, `ProjAppl` ja `Scratch` kylkeen on tullut tuo uutukainen, joka on tarkoitettu **datasettien jakamiseen projektien kesken**. Tällä kurssilla kullakin opiskelijalla on oma student project, mutta olisi tilan haaskausta ladata `n_oppilasta` kertaa Flower Dataset. Siispä opettaja on luonut "dataset project":n MyCSC-sivulla. Sinun ei tarvitse omaa luoda, mutta sinun tulee tietää, mistä se luotaisiin jos tarvitsisit sitä.

SSH:n suhteen uutta on vaatimus, että sinulla on signed SSH key, joka on rekisteröity MyCSC:hen. Dokumentation previkka löytyy: https://csc-guide-preview.2.rahtiapp.fi/origin/roihu/computing/connecting/ssh-keys/#signing-public-key

Tutustutaan myös SSH key forwarding -käsitteeseen ja pintapuolisesti siihen, kuinka voimme käyttää Allasta. Opettaja lataa saman datasetin myös Altaaseen, jotta voimme kurssin aikana harjoitella kummankin käyttöä. Koulutuksen aikana data ladataan juuri sille GPU nodelle, joka sinulle arpoutuu: koulutuksen aikana JPG:tä ei siis ladata verkon yli vaan käytetään paikallista, noden omaan väliaikaiseen käyttöön tarkoitettua nVME-levyä.

## Videolla esitettävä

1. Näytät, että sinulla on oma CSC student project.
2. Kerrot lyhyesti, mikä on supertietokone jaettuna ympäristönä, ja erityisesti:

    * Mitä eroa on login ja compute nodeilla.
    * Miksi on tärkeää olla ajamatta raskasta laskentaa login nodella.
    * Selität, miksi CSC:n ympäristössä muut käyttäjät tulee ottaa huomioon, eikä esimerkiksi huvikseen tai laiskuuttaan jättää tuhoamatta turhaksi käyneitä tiedostoja.

3. Kirjaudut sisään SSH:lla, ja

    * Tulostat nykyisen työskentelyhakemiston.
    * Esittelet keskeiset levyalueet. Navigoi kuhunkin `cd`-komennolla.
    * Löydät erityisesti `flower_photos.tgz`-tiedoston.
    * Käytät `lue`-komentoa ja selität, mitä se tekee.
  
4. Näytät, että tunnet moduulijärjestlemän:

    * Listaat moduulit.
    * Näytät, miten moduuli ladataan.

5. Kerrot lyhyesti, mikä Allas on ja mihin sitä käytetään.

    * TODO. Tähän tulee ehkä konkreettinen steppi, kunhan Roihu on koeponnistettu. 

Kokonaisuutena videosta tulee ilmetä, että ymmärrät CSC:n ajoympäristön perusrakenteen, osaat liikkua siellä ja tiedät, mistä data ja resurssit löytyvät.
