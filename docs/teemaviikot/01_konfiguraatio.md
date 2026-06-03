# 1: Configuration-driven training

Tällä viikolla tutustumme [jani-public/flower-model-demo](https://gitlab.dclabra.fi/jani-public/flower-model-demo)-toteutukseen. Sinun tehtäväsi on kloonata repo, ottaa se käyttöön, tutustua sen rakenteeseen ja ajaa mallin koulutus paikallisesti. Kun mallin koulutus onnistuu:

* säädä hyperparametreja, kunnes mallin `val/F1_score`-mittari on 90 % tai parempi.
* tämän jälkeen aja koulutus myös testidataa vasten.
* tallenna malli `models/`-hakemistoon.
* konvertoi malli ONNX-muotoon.
* ota käyttöön `www/`-hakemistossa oleva demo, joka käyttää ONNX-muotoista mallia.

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
