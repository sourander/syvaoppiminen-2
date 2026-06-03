# 3: Slurm

Tällä viikolla tutustutaan CSC:hen ajoympäristönä sekä erityisesti Roihu-supertietokoneeseen. Keskeinen opetusmateriaali on CSC:n virallinen dokumentaatio, ja erityisesti koulutusmateriaali [CSC Computing Environment](https://csc-training.github.io/csc-env-eff/). Tämän ensimmäisen viikon aihepiiriin kuuluu sisällysluettelon mukaisesti:

Tällä viikolla jatketaan CSC:n ajoympäristöön tutustumista, ja erityisesti keskitytään Slurm-jonojärjestelmään. Opitaan, miten Slurm toimii, miten tehtäviä ajetaan Slurmilla. Tavoitteena on kouluttaa yksinkertainen CNN-malli MNIST-dataa vasten siten, että koulutuksen käynnistää Slurm-työjono, ja koulutus tapahtuu Roihun GPU-nodella (partitiolla `gputest`).

Opetusmateriaalia ovat:

* Viiko viikolta tuttu [CSC Computing Environment](https://csc-training.github.io/csc-env-eff/). Tarkemmin nämä sivut:
    * Batch job tutorial - Serial jobs
    * Using sacct and seff to understand resource usage of finished jobs
    * Get an overview of the resource usage of recent jobs
* CSC:n Running jobs -ohjeet: [Computing > Running jobs](https://docs.csc.fi/computing/running/getting-started/).
* Slurm-dokumentaatio: https://slurm.schedmd.com/documentation.html

MNIST:n koulutukseen riittää seuraava, hyvin lyhyt skripti:

```python title="lenet_mnist.py"
import os
import torch
import torch.nn as nn
import torchvision.transforms.v2 as T

from torch.utils.data import DataLoader
from torchvision.datasets import MNIST

DEVICE = "cuda"; assert not torch.cuda.is_available(), "You are in Roihu, right?"
DATA_PATH = os.environ["LOCAL_SCRATCH"]

# LeNet-5 (Rosebrock MNIST version)
model = nn.Sequential(
    nn.Conv2d(1, 20, kernel_size=5, padding=2),  # 28x28x20
    nn.ReLU(),
    nn.MaxPool2d(2),  # 14x14x20
    nn.Conv2d(20, 50, kernel_size=5, padding=2),  # 14x14x50
    nn.ReLU(),
    nn.MaxPool2d(2),  # 7x7x50
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

## Videolla esitettävä

1. Kerrot lyhyesti, mikä Slurm on, ja mihin sitä käytetään HPC-ympäristöissä.
2. Näytät esimerkin Slurm-työskriptistä:
    
    * Erittelet keskeiset SBATCH-direktiivit.
    * Esittelet, mitkä komennot skriptissä käynnistävät koulutuksen.


3. Siirrä koulutusskripti Roihulle. 

    * Luo lokaali `lenet_mnist.py`-skripti koneellesi
    * Kopioi Roihulle `scp` tai `rsync`-komennolla
    * Näytä, että se perustellussa lokaatiossa.

4. Näytät, miten Slurm-työjono käynnistetään `sbatch`-komennolla.
5. Näytät, miten työn tilaa seurataan `squeue`-komennolla.
6. Näytät, miten työn resurssien käyttöä tarkastellaan `sacct`- ja `seff`-komennoilla.
7. Näytät jobin tuottamat tulostustiedostot

    * `slurm-<jobid>.out` (tai `slurm-<jobid>.err`)

Kokonaisuutena videosta tulee ilmetä, että ymmärrät Slurmin perusidean, osaat ajaa työn GPU-nodessa ja osaat tarkastella ajon tuloksia ja resurssien käyttöä.
