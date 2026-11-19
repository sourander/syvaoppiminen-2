# 8: Kielimallin hienosäätö

!!! warning

    Voi olla, että tämä viikko poistetaan kurssilta kokonaan. Pitää pohtia, ettei kurssi kasva taas liian isoksi. Jos ei, tämä nojaa vahvasti **Fine-Tuning AI**-kirjan prerelease versioon. Sitä kirjoittaa Laurence Moroney ja se julkaistaan July 2027.

    Toinen hyvä lähde on CSC:n [llm-fine-tuning-examples](https://github.com/CSCfi/llm-fine-tuning-examples) repo.

    Ja lisäksi vielä HF:n community article [Fine-Tuning Your First Large Language Model (LLM) with PyTorch and Hugging Face](https://huggingface.co/blog/dvgodoy/fine-tuning-llm-hugging-face), jossa tuunataan Phi-3 Mini 4K puhumaan Yoda-englantia.

    Kaiken kukkuraksi me haluaisimme, että tämä loggaa MLflow:iin, eli seuraten MLflow:n dokumentaatiota: [Fine-Tuning Open-Source LLM using QLoRA with MLflow and PEFT](https://mlflow.org/docs/latest/ml/deep-learning/transformers/tutorials/fine-tuning/transformers-peft/)

Tarkasti rajattuun tehtävään hienosäädetyn tekoälyn lupaus on suuri: _"we'll see why a small, open source model running on a single GPU can and should beat the world's smartest AI in your specific domain."_ [^moroneyFineTuningAI]. Tämä ei kuitenkaan ole hienosäädön automaattinen lopputulos, vaan hypoteesi, joka täytyy osoittaa erillisellä testiaineistolla. Moroney esittelee kirjassaan taulukon, jossa on kärkimallina senhetkinen huippumalli GPT-5.5 Pro ja vastineena edullisimmassa päässä on DeepSeek V4 Pro. Miljoonaa tokenia kohden näiden hinta on input/output...

- GPT: \$30/\$180
- DeepSeek: \$0.44/\$0.87

Jos malleja vertailee eräajosssa, esimerkiksi sopimusanalyysin kontekstissa, niin 5000 dokumentin käsittely maksaisi GPT-5.5 Prolla noin 8000 dollaria ja DeepSeekillä noin 60 dollaria. Mikäli luet tätä merkittävästi myöhemmin kuin syksyllä 2026, mallit ja hinnat ovat voineet muuttua. Voit kurkata tätä esimerkiksi [OpenCode Zen](https://opencode.ai/docs/console/models/#pricing)-hinnastosta, joka on pay-as-you-go -tyyppinen inferenssipalvelu.

Hienosäädön idea on, että löydetään pieni, edullinen malli, joka suoriutuu tehtävästä riittävän hyvin. Termi _riittävän hyvin_ on bisneskysymys ja vaatii tuekseen metriikoita, aivan kuten mikä tahansa muukin koneoppimisen projekti. Se, onko tuloste monivalinta, tietyn skeeman mukainen JSON vai esimerkiksi vapaatekstitiivistelmä, vaikuttaa evaluointimetriikkaan. Tässä materiaalissa evaluointi jätetään tekemättä, mutta voit tutustua siihen esimerkiksi Hugging Facen [The LLM Evaluation Guidebook](https://huggingface.co/spaces/OpenEvals/evaluation-guidebook)-dokumentista tai kirjaston [Inspect AI](https://inspect.aisi.org.uk/tutorial.html) tutoriaalista.

Se, kannattaako mallille antaa vain parempi konteksti RAG-tyylisiä ratkaisuja, skill-tiedostoja tai muita kehotemuotoilun keinoja käyttäen vai hienosäätä malli erikseen, riippuu tapauksesta. Chip Huyen esittelee keskeiseksi hienosäätöä puoltavaksi syyksi sen, että tarvittu ==käytös== ei ole ollut osa sen koulutusta, kuten esimerkiksi tiettyyn skeemaan sopiva JSON-vastaus tai tietty SQL-dialekti. Syiksi vältellä hienosäätöä mainitaan, että se vaatii kehotemuotoiluun verrattuna merkittävän määrän ajallista panostusta, ja monissa tapauksissa kehotemuotoilu riittää tai johtaa parempiin tuloksiin. Jos kyse on siitä, että mallilla ei ole pääsyä ajantasaiseen informaatioon, RAG-tyylinen ratkaisu lienee oikea valinta. Chipin suositus on: _"explore more advanced solutions only if prompting alone proves inadequate"_ [^huyenAIEngineering]

Me ollaan täällä kuitenkin oppimassa hienosäätöä, joten unohdetaan toistaiseksi se, onko tässä mitään järkeä, ja hienosäädetään malli. Käyttämämme metodi on PEFT (Parameter-Efficient Fine-Tuning), joka mahdollistaa mallin hienosäädön ilman, että koko mallia tarvitsee kouluttaa uudelleen -- vain lisätyt parametrit koulutetaan. Perinteisessä PEFT:ssä malliin lisätään parametrejä, joten mallin inferenssinopeus hidastuu [^huyenAIEngineering]. PEFT-metodeista käytetyimmäksi Chip mainitsee LoRA:n (Low-Rank Adaptation), mutta muitakin löytyy, kuten QLoRA (Quantized Low-Rank Adaptation), DoRA (Direction and Magnitude Decomposition), LoftQ (Quantization-Aware LoRA Initialization) [^nicoleTransformers]. Tutustu ihmeessä ainakin pintapuolisesti Hu et al. paperiin [LoRA: Low-Rank Adaptation of Large Language Models](https://arxiv.org/abs/2106.09685). Avuksi voit ottaa Paper Explained -kanavan videon [
LoRA Explained Visually](https://youtu.be/DfYNsH3xK1Y?si=zEgG0685hJLYeqFC).

## LoRA todiste

## Videolla esitettävä

1. TODO

## Lähteet

[^moroneyFineTuningAI]: Moroney, L. _Fine-Tuning AI_. O'Reilly. 2026.
[^huyenAIEngineering]: Huyen, C. _AI Engineering_. O'Reilly. 2024.
[^nicoleTransformers]: Koenigstein, N. _Transformers: The Definitive Guide_. O'Reilly. 2026.
