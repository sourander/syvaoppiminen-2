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

## Työvaiheet

### 1: LoRA-todiste

Tutustu siihen, kuinka LoRA toimii. Tähän on hyvä apu Nicole Koenigsteinin kirjassa _Transformers in Action_. Alla on Marimo Notebook, joka myötäilee kirjan luvun 8 esimerkkiä.

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
- Millä rankin arvolla $hat{W}$ vastaa alkuperäistä matriisia $W$ täydellisesti (pois lukien pyöristysvirheet)?
- Miten tämä liittyy _Johdatus koneoppimiseen_ -kurssilta tuttuun dimensiovähennykseen?

Tässä demonstraatiossa matalan rankin matriisit muodostetaan SVD:n avulla. Varsinaisessa LoRA-menetelmässä matriisit $A$ ja $B$ koulutetaan gradienttien avulla. LoRA käyttää niiden tuloa painomatriisin päivityksen esittämiseen.

$$
W' = W + \Delta W, \qquad \Delta W = BA
$$

### 2: Coming soon

TODO.

## Videolla esitettävä

1. TODO

## Lähteet

[^moroneyFineTuningAI]: Moroney, L. _Fine-Tuning AI_. O'Reilly. 2026.
[^huyenAIEngineering]: Huyen, C. _AI Engineering_. O'Reilly. 2024.
[^nicoleTransformers]: Koenigstein, N. _Transformers: The Definitive Guide_. O'Reilly. 2026.
