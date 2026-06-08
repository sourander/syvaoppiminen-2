# 9: Tekoäly ja etiikka

!!! tip

    Linkit vaativat vielä läpikäyntiä ja tarkistusta. Ne on leikattu ja liimattu aiemmasta toteutuksesta, Raindrop-kirjastosta ja tekoälyn vinkkaamista linkeistä.

## Materiaali

Tällä teemaviikolla tarkastellaan tekoälyä teknisen järjestelmän lisäksi yhteiskunnallisena ilmiönä. Painopiste on erityisesti suurissa kielimalleissa, generatiivisessa tekoälyssä, tutkimuskäytön luotettavuudessa, regulaatiossa, datassa, immateriaalioikeuksissa, ympäristövaikutuksissa ja ihmisen roolissa tekoälyjärjestelmän käyttäjänä. Et tutustu näihin kaikkiin syvällisesti, vaan valitset sinulle läheisimmän tai mielestäsi tärkeimmän näkökulman, johon perehdyt tarkemmin. 

Tavoitteena ei ole muodostaa yhtä “oikeaa” mielipidettä tekoälystä, vaan oppia tunnistamaan riskejä ja arvioimaan tekoälyjärjestelmän käyttöä vastuullisesti.

## 1. (Tieteellinen) luotettavuus

LLM-malleja vertaillaan usein benchmark-tuloksilla, mutta pelkkä pistemäärä voi antaa virheellisen kuvan mallien todellisesta käyttäytymisestä. Erityisesti tutkimuskäytössä mallivalinta, promptit ja annotointiprosessi voivat vaikuttaa lopputuloksiin. Kuinka malliin voidaan luottaa? Onko tekoäly uhka tieteelle? Entä uutisille?

- Yang & Wang: *Benchmark Illusion: Disagreement among LLMs and Its Scientific Consequences*  
  https://arxiv.org/abs/2602.11898

- Stanford HAI: AI Index 2026  
  https://hai.stanford.edu/ai-index/2026-ai-index-report

- Zhao et al. 2026: LLM hallucinations in the wild: Large-scale evidence from non-existent citations  
  https://arxiv.org/abs/2605.07723

- Yle perustaa tiedon varmentamiseen erikoistuvan tiimin  
  https://yle.fi/a/20-10007870

## 2. Kriittinen ajattelu ja oppiminen

Tekoälytyökalut voivat tukea oppimista, mutta ne voivat myös vähentää käyttäjän omaa ajattelua, ongelmanratkaisua ja lähdekritiikkiä, jos niitä käytetään passiivisesti – vai kuinka on? Ota selvää.

- Gerlich, M. (2025): *AI Tools in Society: Impacts on Cognitive Offloading and the Future of Critical Thinking*  
  https://doi.org/10.3390/soc15010006

- Microsoft Research: *The Impact of Generative AI on Critical Thinking*  
  https://www.microsoft.com/en-us/research/publication/the-impact-of-generative-ai-on-critical-thinking/

- UNESCO: *Guidance for generative AI in education and research*  
  https://www.unesco.org/en/articles/guidance-generative-ai-education-and-research

- Kosmyna et al 2025: Your brain on ChatGPT  
  https://arxiv.org/abs/2506.08872

## 3. Lainsäädäntö

Euroopan unionin AI Act tuo tekoälyjärjestelmille riskiperustaisen sääntelymallin. Sen rinnalla on edelleen huomioitava GDPR, tekijänoikeus, tietoturva ja toimialakohtainen sääntely. Jos lainsäädäntö kiinnostaa, pengo asiaa.

- EU AI Act: virallinen säädösteksti  
  https://eur-lex.europa.eu/eli/reg/2024/1689/oj

- EU AI Act implementation timeline  
  https://artificialintelligenceact.eu/implementation-timeline/

- European Commission: AI Act  
  https://digital-strategy.ec.europa.eu/en/policies/regulatory-framework-ai

- European Commission: Guidelines for providers of general-purpose AI models  
  https://digital-strategy.ec.europa.eu/en/policies/guidelines-gpai-providers

- Tietosuojavaltuutetun toimisto: GDPR  
  https://tietosuoja.fi/usein-kysyttya-gdpr

## 4. Data, vinoumat ja edustavuus

Mallin laatu ja oikeudenmukaisuus riippuvat vahvasti datasta. Koulutusdata voi sisältää historiallisia vinoumia, puutteellista edustavuutta, virheellisiä annotaatioita, henkilötietoja tai tekijänoikeudellisesti epäselvää materiaalia.

- NIST Schwartz et al. 2022: *Towards a Standard for Identifying and Managing Bias in Artificial Intelligence*  
  https://www.nist.gov/publications/towards-standard-identifying-and-managing-bias-artificial-intelligence

- Hugging Face: Model Cards  
  https://huggingface.co/docs/hub/model-cards

- Wang & Russakovsky 2023: Overwriting Pretrained Bias with Finetuning Data  
  https://openaccess.thecvf.com/content/ICCV2023/papers/Wang_Overwriting_Pretrained_Bias_with_Finetuning_Data_ICCV_2023_paper.pdf

- Garimella et al. 2022: Demographic-Aware Language Model Fine-tuning as a Bias Mitigation Technique  
  https://aclanthology.org/2022.aacl-short.38/

- Ahmed et al. 2026: Extracting books from production language models  
  https://arxiv.org/abs/2601.02671

## 5. Tekoälymallien tekijänoikeudet, lisenssit ja avoin lähdekoodi

Tekoälyprojekteissa täytyy erottaa toisistaan koodi, data, mallipainot, dokumentaatio ja tuotokset. Jokaisella näistä voi olla eri lisenssi, eri käyttöehdot ja eri riskit.

- Open Source Initiative: Licenses  
  https://opensource.org/licenses

- Choose a License  
  https://choosealicense.com/

- Hugging Face: Model licenses  
  https://huggingface.co/docs/hub/repositories-licenses

## 6. Työolot, datan annotointi ja toimitusketjut

Generatiivisen tekoälyn taustalla on usein näkymätöntä ihmistyötä: datan keruuta, luokittelua, moderointia, arviointia ja sisällön suodatusta. Vastuullinen tekoäly edellyttää myös näiden toimitusketjujen tarkastelua.

- MOT: Applen tekoälyn kouluttajat paljastavat salatut säännöt: ”Sensuuria”  
  https://yle.fi/a/74-20224113

- Partnership on AI: Responsible Sourcing of Data Enrichment Services  
  https://partnershiponai.org/responsible-sourcing-considerations/

- Data Workers’ Inquiry  
  https://data-workers.org/

## 7. Turvallisuus, väärinkäyttö ja LLM-uhat

LLM-järjestelmiin liittyy uusia turvallisuusongelmia, kuten prompt injection, jailbreakit, datan vuotaminen, agenttien hallitsematon toiminta ja luottamuksellisen tiedon päätyminen vääriin paikkoihin. Muista, että älä kirjoita tästä teknologian vaan etiikan näkökulmasta. Voit pohtia myös tahallista väärinkäyttöä, kuten disinformaatiota tai deepfake-sisältöä.

- OWASP Top 10 for Large Language Model Applications  
  https://owasp.org/www-project-top-10-for-large-language-model-applications/

- YLE Marjatta Rautio: ”Iljettävää rikollisuutta” – lainoppineet kertovat, miten deepfake-pornoon voisi puuttua  
  https://yle.fi/a/74-20227385

## 8. Ympäristövaikutukset ja resurssien käyttö

Mallien koulutus ja käyttö kuluttavat energiaa, vettä, laskentakapasiteettia ja laitteistoja. Vastuullinen mallikehitys edellyttää myös mallin koon, käyttötarkoituksen ja resurssikulutuksen arviointia.

- Stanford HAI: AI Index 2026  
  https://hai.stanford.edu/ai-index/2026-ai-index-report

- Green Software Foundation  
  https://greensoftware.foundation/

- YLE Jussi Hanhivaara: Datakeskukset huolestuttavat myös Yhdysvalloissa – tutkimuksen mukaan sähkön hinta voi nousta jopa lähes 60 prosenttia  
  https://yle.fi/a/74-20227106

## 9. Autonomiset järjestelmät, vastuu ja ihmisen rooli

Kun tekoälyjärjestelmä tekee päätöksiä tai toimii osana kriittistä järjestelmää, täytyy kysyä kuka vastaa, kuka valvoo, mitä lokitetaan ja milloin ihmisen pitää puuttua toimintaan.

- Moral Machine  
  https://www.moralmachine.net/

- Future of Life Institute: Autonomous Weapons  
  https://futureoflife.org/project/autonomous-weapons/


## Tehtävä

Tekoälyn käyttö on tuonut monia uusia eettisiä ongelmia ohjelmointiin ja tuotesuunnitteluun. Esim. kenen vika on, jos autonomisesti kulkeva ajoneuvo ajaa kolarin ja ihminen loukkaantuu. Ohjeet:

* Tutustu yllä olevaan materiaaliin vähintään pintapuoleisesti.
* Tutustu valitsemaasi aihealueeseen syvällisemmin. Etsi myös muita lähteitä kuin yllämainitut.
* Tee ja palauta essee: Tekoäly ja etiikka.

Vaatimuksia ovat:

* Esseen tulee olla kirjoitettu suomeksi.
* Essee on yksittäinen PDF-tiedosto.
* Esseen tulee olla vähintään 500 sanaa leipätekstinä mitattuna.
* Esseen tulee käsitellä tekoälyn etiikkaa ja sen vaikutuksia yhteiskuntaan.
* Esseen tulee sisältää vähintään kolme eri lähdettä.
    * Käytä Vancouver-lähdeviittausta.
    * Lähteiden tulee olla luotettavia ja relevantteja aiheeseen liittyen.
    * Vähintään yhden lähteen tulee olla tieteellinen artikkeli, joka on vertaisarvioitu tai vähintään julkaistu luotettavassa julkaisussa.
    * Vähintään yhden lähteen tulee julkaistu vuoden sisällä. Se voi olla esimerkiksi uutisartikkeli tai blogikirjoitus.

Työn saa kirjoittaa valitsemallaan työkalulla. Suosittelen tutustumaan [Typst](https://typst.app/) -työkaluun. Saat tyylikkään, Arxiv-henkisen ulkonäön käyttämällä Typst Universesta [arkeion](https://typst.app/universe/package/arkheion/) -templaattia. Opettaja voi esitellä Typst:n käytön tukisessioissa.

!!! tip "Lisävinkki työnhakuun"

    Typst:llä on helppo kehittää CV ja eri työnantajille räätälöidyt saatekirjeet (engl. cover letter), ja niiden ylläpitäminen privaatissa Github-repositoriossa on yllättävän oiva tapa säilöä CV:t. 
    
    Tähän käyttötarkoitukseen löytyy useita valmiita templaatteja, kuten [basic-resume](https://typst.app/universe/package/basic-resume/) tai [modern-cv](https://typst.app/universe/package/modern-cv), joista ainakin jälkimmäinen vaatii fonttien asentamista käyttöjärjestelmään. Sen saa kuitenkin käyttöön vartissa.