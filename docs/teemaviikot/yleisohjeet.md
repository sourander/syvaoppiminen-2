# Yleisohjeet

## Essee

Kurssilla on yksi ainut essee-tehtävä, eli Tekoäly ja etiikka, palautetaan Reppuun PDF-tiedostona. Essee on kirjoitettava suomeksi. Lue tarkemmat ohjeet tehtävän [9: Tekoäly ja etiikka](09_etiikka.md) -dokumentista. Tehtävä arvioidaan Hyväksytty/Hylätty -arvosanoin, ja se on pakollinen kurssin suorittamisen kannalta, kuten kaikki muutkin tehtävät.

## Repositorio

!!! warning

    Käytäthän opettajan sinulle antamaasi repositoriota, jotta opettajalla on automaattisesti pääsy sinun työhösi. Varmista, että repositoryn juuressa on `README.md`-tiedosto, josta alkaen käyttäjä kuljetetaan eri ohjeiden äärelle. Jos dokumentaatio on repositorion punainen lanka, niin `README.md` on lankakerän pää. Siitä on hyvä linkittää kaikkiin ohjeisiin, jotta opettaja löytää ne helposti.

Käytä repositoriossa ns. monorepo-rakennetta, jossa kaikki harjoitukset sijaitsevat samassa repositoriossa. Jokaisella harjoituksella on oma kansio. Esimerkiksi:

```
.
├── README.md
├── flower-model-demo
│   └── Justfile.yml
├── parallelism
│   ├── MEMO.md
│   └── compose.yml
├── fine-tuning
│   └── MEMO.md
└── <ID>
    ├── ...
    └── ...
```

Tarkka repositorion rakenne on sinun päätettävissä. Varmista kuitenkin, että se on selkeä siten, että opettaja löytää `README.md`-tiedoston ja sitä kautta muut harjoitukset. On esimerkiksi täysin sallittua antaa juokseva järjestysnumero tekemisille harjoituksilla, kuten `01_FLOWER`, `02_PARALLELISM` jne.

!!! example "Kurssi syntyy prosessissa"

    Tämä toteutus järjestetään ensimmäistä kertaa syksyllä 2026, joten ns. opettajan esimerkki repositorion rakenteesta elää toteutuksen ajan. Opettajalla ei ole kristallipalloa, jolla näkisi Roihun tulevaisuuteen.
    
    Seuraa Discordia ja sähköpostia, niin pysyt ajan tasalla mahdollisista vinkeistä.

!!! tip "CLI-työskentely"

    Kun ajat harjoituksia, siirry kyseiseen hakemistoon, ja aja komennot siellä. Esimerkiksi jos tutustut opettajan tekemään Flower Model -aihioon, niin:

    ```bash
    git clone https://gitlab.dclabra.fi/jani-public/flower-model-demo.git
    cd flower-model-demo
    uv run python
    >>> from flowermodel.config import load_config_from_args
    >>> ... # do your thing
    ```

## Videot

Lähes kaikki kurssin tehtävät ovat videopalautuksia. Video palautetaan linkkinä. Niiden arviointiin käytetän Arviointityökalua asteikolla 0-5.

Videon tekemisen ohjeistus on ulkoistettu [HedgeDoc: Videotetävän yleisohjeistus](https://gitlab.dclabra.fi/wiki/s/WX2xszsjJe) -dokumenttiin, koska sama ohjeistus on käytössä useilla kursseilla.
