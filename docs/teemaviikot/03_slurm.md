# 3: Slurm

Tällä viikolla tutustutaan CSC:hen ajoympäristönä sekä erityisesti Roihu-supertietokoneeseen. Keskeinen opetusmateriaali on CSC:n virallinen dokumentaatio, ja erityisesti koulutusmateriaali [CSC Computing Environment](https://csc-training.github.io/csc-env-eff/).

Tarkemmin viikon aihe on Slurm-jonojärjestelmä. Opitaan, miten Slurm toimii ja kuinka tehtäviä ajetaan Slurmilla. Tavoitteena on kouluttaa yksinkertainen CNN-malli MNIST-dataa vasten siten, että koulutus suoritetaan Slurmin kautta lähetettävänä eräajotyönä, ja koulutus tapahtuu Roihun GPU-solmulla (partitiolla `gputest`). Kyseinen `gputest` on lyhyisiin testiajoihin tarkoitettu partitio. Tutustu sen rajoituksiin CSC:n dokumentaatiosta itsenäisesti.

Opetusmateriaalia ovat:

- Viime viikolta tuttu [CSC Computing Environment](https://csc-training.github.io/csc-env-eff/). Tarkemmin nämä sivut:
  - Batch job tutorial - Serial jobs
  - Using sacct and seff to understand resource usage of finished jobs
  - Get an overview of the resource usage of recent jobs
- CSC:n Running jobs -ohjeet: [Computing > Running jobs](https://docs.csc.fi/computing/running/getting-started/).
- Slurm-dokumentaatio: https://slurm.schedmd.com/documentation.html

## Koulutettava malli

MNIST:n koulutukseen riittää seuraava, hyvin lyhyt skripti:

```python title="lenet_mnist.py"
import os
import torch
import torch.nn as nn
import torchvision.transforms.v2 as T

from torch.utils.data import DataLoader
from torchvision.datasets import MNIST

DEVICE = "cuda"
assert torch.cuda.is_available(), "CUDA GPU is not available. Did the Slurm job request a GPU?"
print(f"Using GPU: {torch.cuda.get_device_name(0)}")


DATA_PATH = os.environ["TMPDIR"]
print(f"Temporary data directory: {DATA_PATH}")

# LeNet-style CNN (Rosebrock MNIST version)
model = nn.Sequential(
    nn.Conv2d(1, 20, kernel_size=5, padding=2),   # 20x28x28
    nn.ReLU(),
    nn.MaxPool2d(2),                              # 20x14x14
    nn.Conv2d(20, 50, kernel_size=5, padding=2),  # 50x14x14
    nn.ReLU(),
    nn.MaxPool2d(2),                              # 50x7x7
    nn.Flatten(),
    nn.Linear(7 * 7 * 50, 500),
    nn.ReLU(),
    nn.Linear(500, 10),
).to(DEVICE)

# MNIST
transform = T.Compose([T.ToImage(), T.ToDtype(torch.float32, scale=True),])
train_ds = MNIST(DATA_PATH, train=True, download=True, transform=transform)
test_ds = MNIST(DATA_PATH, train=False, download=True, transform=transform)
train_loader = DataLoader(train_ds, batch_size=64, shuffle=True)
test_loader = DataLoader(test_ds, batch_size=64, shuffle=False)

# Training setup
criterion = nn.CrossEntropyLoss()
opt = torch.optim.Adam(model.parameters(), lr=1e-3)

# Train
for epoch in range(10):
    model.train()

    for x, y in train_loader:
        x, y = x.to(DEVICE), y.to(DEVICE)

        opt.zero_grad()
        loss = criterion(model(x), y)
        loss.backward()
        opt.step()

    # Evaluate
    model.eval()
    correct = 0
    total = 0

    with torch.no_grad():
        for x, y in test_loader:
            pred = model(x.to(DEVICE)).argmax(1)
            correct += (pred.cpu() == y).sum().item()
            total += y.size(0)

    print(f"Epoch {epoch+1}: accuracy = {100 * correct / total:.2f}%")
```

Yllä olevassa skriptissä on käytössä `TMPDIR`-ympäristömuuttuja, johon sinun kannattaa perehtyä huolella. Kyseessä on Roihun uusi ominaisuus (ks [Available batch job partitions > Local storage on Roihu nodes](https://docs.csc.fi/computing/running/batch-job-partitions/#local-storage-on-roihu-nodes)). Edellisessä Puhti-supertietokoneessa piti erikseen pyytää Slurm-jobissa varaus nvme-levytilasta, jonka lokaatio tallentui `LOCAL_SCRATCH`-ympäristömuuttujaan. Roihussa on käytössä `TMPDIR`, joka on automaattisesti saatavilla; tarpeen mukaan on mahdollista pyytää myös `LOCAL_SCRATCH`-tilaa, mutta tämä on sallittua vain suurimmilla partitioilla. Tarkista dokumentaatiosta, kuinka monta gigaa levytilaa on käytettävissä. 

Kannattaa myös selvittää, millaisissa aineistoissa solmukohtaisesta tallennustilasta on hyötyä ja miksi esimerkiksi suurta määrää pieniä tiedostoja ei yleensä kannata käsitellä suoraan jaetulla`/scratch`-levyalueelta. Tämä selitetään hyvinkin ymmärrettävällä tavalla FAQ-artikkelissa [Which directory should I use to analyze many small files?](https://docs.csc.fi/support/faq/local_scratch_for_data_processing/).

!!! warning

    Huomaa, että $TMPDIR on työnaikainen väliaikaishakemisto, jonka sisältö ei säily työn päätyttyä. Tässä tehtävässä MNIST ladataan siksi uudelleen jokaisessa ajossa. Jos Pytorchin latauspalvelin on alhaalla, me emme voi tehdä meidän työtämme. Tämän takia oma, pysyvä, lokaali kopio MNIST-datasta olisi parempi käytäntö. Näin me toimimme Flower Datasetin kanssa.

## Slurm

En anna valmista skriptiä, vaan sinun tulee itse kirjoittaa Slurm-työskripti, joka suorittaa yllä olevan `lenet_mnist.py`-koulutuksen. Tutustu CSC:n dokumentaatioon ja Slurm-dokumentaatioon, ja kirjoita oma Slurm-työskripti.

TODO! Tähän tulee CSC:n Slurm-tutoriaaliin linkki, kunhan se on julkaistu Noppe-palvelussa.

Jos CSC:n dokumentaatiossa mainitut MPI, OpenMPI ja muut herättävät kiinnostusta, voimme keskustella näistä aiheista kurssin Iltanuotioilla.

## Videolla esitettävä

1. Kerro lyhyesti, mikä Slurm on ja mihin sitä käytetään HPC-ympäristössä.

2. Esittele laatimasi Slurm-työskripti.
 
    - Selitä työn laskutusprojekti, partitiovalinta, aikaraja ja GPU-varaus.
    - Näytä, miten tarvittava ohjelmistoympäristö ladataan.
    - Osoita `srun`-komento, joka käynnistää Python-ohjelman.

3. Siirrä koulutusskripti ja Slurm-työskripti Roihulle.
    - Luo `lenet_mnist.py` paikallisella koneellasi.
    - Kopioi tiedostot Roihulle `scp`- tai `rsync`-komennolla.
    - Näytä tiedostojen sijainti Roihussa ja perustele valinta lyhyesti.

4. Lähetä työ jonoon `sbatch`-komennolla ja ota talteen komennon
   palauttama job ID.

5. Seuraa työn tilaa `squeue`-komennolla.
    - Jos työ on jo valmistunut, selitä, miksi se ei enää näy
        `squeue`-tulosteessa.

6. Tarkastele valmistuneen työn tietoja `sacct`- ja `seff`-komennoilla.
    - Tulkkaa lyhyesti ainakin työn tila, suoritusaika ja resurssien käyttö.

7. Näytä työn tuottama `slurm-<jobid>.out`-tiedosto sekä mahdollinen
   erillinen virhetiedosto.
    - Osoita tulosteesta, että PyTorch havaitsi CUDA-GPU:n.
    - Osoita, että koulutus valmistui ja tuotti järkevän tarkkuuden.

Kokonaisuutena videosta tulee ilmetä, että ymmärrät Slurmin perusidean,
osaat varata työlle tarkoituksenmukaiset resurssit, suorittaa työn
Roihun GPU-solmulla sekä tarkastella ajon tuloksia ja resurssien käyttöä.
