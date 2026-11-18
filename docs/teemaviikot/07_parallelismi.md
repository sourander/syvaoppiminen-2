# 7: Parallelismi

Tällä viikolla tutustutaan rinnakkaislaskentaan (_engl. parallel computing_) supertietokoneympäristössä. Ennen kuin hyppäämme usean GPU:n koulutukseen PyTorchilla, opettelemme rinnakkaislaskennan peruskäsitteet prosessoripuolella käyttäen **MPI**- ja **OpenMP**-tekniikoita. Lopuksi koulutamme neuroverkon neljällä Nvidia GH200 -GPU:lla käyttäen PyTorchin `DistributedDataParallel` (DDP) -luokkaa.

!!! note

    Jos tältä viikolta ei jää mitään muuta mieleen, niin paina vähintään tämä mieleesi: **lisäraudan varaaminen ei tee ohjelmastasi nopeampaa, jos ohjelma ei ole kirjoitettu hyödyntämään rinnakkaisuutta**. Varsinkin puhdas Python ajetaan vain yhdellä ytimellä, joskin kirjastot kuten NumPy käyttävät konepellin alla OpenMP:tä tai BLAS-kirjastoja, jotka jakavat kuormaa useille CPU-ytimille.

Tutustu ihan vähimmillään seuraaviin materiaaleihin:

- [YouTube: A friendly introduction to distributed training (ML Tech Talks)](https://www.youtube.com/watch?v=S1tN9a4Proc)
- [YouTube: Distributed ML Talk @ Berkeley](https://youtu.be/9TwTKt50ZG8?si=PshtXLgsyBRS3uMa)
- CSC:n DDP-esimerkkirepositorio: [CSCfi/pytorch-ddp-examples](https://github.com/CSCfi/pytorch-ddp-examples)

Lisäksi on suositeltavaa silmäillä CSC-materiaalia, jonka slidet löytyvät [CSC Summer School in High-Performance Computing](https://csc-training.github.io/summerschool/main/index.html) -sivustolta. Erityisesti kannattaa tutustua seuraaviin osioihin:

-   Introduction to High-Performance Computing (HPC)
    - 3: Parallel Computing
-   Message Passing Interface (MPI)
    - 1: Introduction to MPI
    - 2: Point-to-point communication I
-   HPC and AI
    - 2: Scaling PyTorch Models: Single vs Multi-GPU Training and Techniques  

!!! tip

    Jos Summer Schoolin HTML-versiossa eivät toimi kuvat, löydät samoista dioista PDF-versiot kun selaat sivua hieman alaspäin.

## Rinnakkaislaskennan perusidea

Yksinkertaisin tapa hahmottaa rinnakkaislaskentaa on tarkastella ongelmaa, joka on luonnostaan helposti rinnakkaistettava (_engl. [embarrassingly parallel](https://en.wikipedia.org/wiki/Embarrassingly_parallel)_). Hyvä arkipäiväinen esimerkki on taulukon lukujen $[x_0, x_1, x_2, x_3]$ summaaminen (_reduction with sum_).

Jos käytössä on vain **yksi suorittaja** (oli se sitten yksittäinen prosessoriydin tai laskutoimituksia paperilla tekevä ihminen), summa on laskettava vaihe kerrallaan: ensin lasketaan $x_0 + x_1$, sen jälkeen välisummaan lisätään $x_2$, ja lopuksi tulokseen lisätään $x_3$:

``` mermaid
flowchart LR
    x0["x_0"] --> sum1["+"]
    x1["x_1"] --> sum1
    sum1 -->|"x_0 + x_1"| sum2["+"]
    x2["x_2"] --> sum2
    sum2 -->|"x_0 + x_1 + x_2"| sum3["+"]
    x3["x_3"] --> sum3
    sum3 --> res["Summa"]
```

Jos käytössä on **kaksi suorittajaa**, data voidaan puolittaa. Ensimmäinen suorittaja laskee alkioiden $x_0$ ja $x_1$ summan samanaikaisesti, kun toinen suorittaja laskee alkioiden $x_2$ ja $x_3$ summan. Lopuksi osasummat yhdistetään:

``` mermaid
flowchart LR
    subgraph w1["Suorittaja 1"]
        x0["x_0"] --> sum1["+"]
        x1["x_1"] --> sum1
    end
    subgraph w2["Suorittaja 2"]
        x2["x_2"] --> sum2["+"]
        x3["x_3"] --> sum2
    end
    sum1 -->|"x_0 + x_1"| sum3["+"]
    sum2 -->|"x_2 + x_3"| sum3
    sum3 --> res["Summa"]
```

Tämä on **dataparallelismin** perusajatus: jokainen suorittaja tekee saman laskentaoperaation, mutta omalle osalleen datasta, ja lopuksi tulokset yhdistetään. Toivon mukaan on loogista, että suorittajia voi olla tässä tapauksessa yhtä monta kuin datapisteitä. Jos ynnättävien elementtien määrä on 100 ja sinulla on 999 workeriä, niin 899 suorittajaa jää tyhjäkäynnille.

!!! warning

    On olemassa myös luonnostaan peräkkäisiä ongelmia, joiden rinnakkaistaminen on vaikeaa tai mahdotonta. Jos laskennan vaihe $t$ tarvitsee syötteekseen vaiheen $t-1$ tuloksen, kuten kielimallin tuottaessa tekstiä sana kerrallaan tai neuroverkon päivittäessä painojaan askeleittain, laskentaa ei voi suorittaa samanaikaisesti.

    Näiden ääripäiden väliin mahtuu liuta eri tavoin rinnakkaistettavia ongelmia ja niiden ratkaisuun eri määrin rinnakkaistuvia algoritmeja.

## Prosessi- ja säietason rinnakkaisuus: MPI ja OpenMP

Supertietokoneessa rinnakkaisuus jakautuu kahteen eri tasoon sen mukaan, miten muisti on järjestetty:

1. **Prosessitason rinnakkaisuus (MPI, _Message Passing Interface_):** Jokaisella prosessilla on oma eristetty muistialueensa. Prosessit voivat sijaita samalla laskentasolmulla (_engl. node_) tai kokonaan eri laskentasolmuilla verkon yli. Koska prosessit eivät näe toistensa muistia suoraan, MPI ratkaisee sen, miten dataa siirretään ja synkronoidaan prosessien välillä viesteinä (esimerkiksi lähettämällä dataa juuriprosessilta kaikille muille).
2. **Säietason rinnakkaisuus (OpenMP, _Open Multi-Processing_):** Yksittäisen laskentasolmun sisällä on kymmeniä tai satoja prosessoriytimiä (_engl. CPU cores_), jotka jakavat saman keskusmuistin. OpenMP ratkaisee paikallisen rinnakkaistamisen: yksi prosessi voi käynnistää rinnakkaisen laskentavaiheen (kuten `for`-silmukan) ajaksi useita säikeitä (_engl. threads_), jotka jakavat työn prosessoriydinten kesken ja lukevat samaa muistia. Kun silmukka on suoritettu, säikeet yhdistyvät jälleen yhdeksi pääsäikeeksi.

Tyypillisessä suurteholaskennan hybridimallissa nämä kaksi yhdistetään: **MPI** hoitaa kommunikoinnin eri prosessien ja laskentasolmujen välillä, ja kunkin MPI-prosessin sisällä **OpenMP** valjastaa prosessille varatut paikalliset prosessoriytimet rinnakkaiseen laskentaan.

Perinteisesti MPI- ja OpenMP-ohjelmat kirjoitetaan C-, C++- tai Fortran-kielellä. Koska tämä kurssi on Python-pohjainen, käytämme Python-kirjastoja (`mpi4py` ja `numba`), jolloin vältymme erilliseltä kääntämiseltä ja voimme keskittyä itse rinnakkaisuuden toimintaan.

## GPU-parallelismi syväoppimisessa

Kun neuroverkkoja koulutetaan tai ajetaan usealla GPU:lla, rinnakkaistamismenetelmät eroavat toisistaan pääasiassa siinä, **mitä pilkotaan**: koulutusdataa, mallin kerroksia vai yksittäisen kerroksen sisäisiä laskentaoperaatioita.

-   **Dataparallelismi** (_engl. data parallelism_): Jokaisella GPU:lla on täysi kopio mallista, ja kukin GPU käsittelee eri osan koulutuserästä (_engl. batch_). Vastavirran (_engl. backpropagation_) yhteydessä GPU:t synkronoivat gradienttinsa, jotta jokainen mallikopio päivittyy yhtenäisesti. Dataparallelismi on suoraviivainen ja tehokas menetelmä silloin, kun malli ja sen koulutustila mahtuvat yhden GPU:n muistiin.[^viehmannDLwithPytorch]
-   **Tensor-parallelismi** (_engl. tensor parallelism_): GPU:t jakavat tensorit tai laskentaoperaatiot yksittäisten kerrosten sisällä. Esimerkiksi suuri painomatriisi voidaan osittaa siten, että eri GPU:t laskevat matriisikertolaskun osia rinnakkain. Tämä mahdollistaa sellaisten kerrosten ajamisen, jotka ovat liian suuria yhdelle GPU:lle, mutta GPU:iden välinen jatkuva tiedonsiirto lisää kommunikointiviivettä.[^huyenAIEngineering][^freglyAISystems] Kuten Huyen[^huyenAIEngineering] tiivistää tensor-parallelismin hyödyn ja kompromissin:

    > "First, it makes it possible to serve large models that don’t fit on single machines. Second, it reduces latency. The latency benefit, however, might be reduced due to extra communication overhead."

-   **Pipeline-parallelismi** (_engl. pipeline parallelism_): Mallin peräkkäiset kerrokset ryhmitellään vaiheiksi (_engl. stages_), jotka sijoitetaan eri GPU:ille. Koulutuserä pilkotaan pienempiin mikroeriin (_engl. micro-batches_), jolloin eri GPU:t voivat käsitellä eri mikroeriä samanaikaisesti liukuhihnan tavoin. Osa GPU:ista joutuu kuitenkin odottamaan toimettomana silloin, kun liukuhihna täyttyy tai tyhjenee; näitä odotusaikoja kutsutaan _pipeline-kupliksi_ (_engl. pipeline bubbles_).[^viehmannDLwithPytorch][^freglyAISystems]

Suurten kielimallien ja raskaiden neuroverkkojen koulutuksessa hyödynnetään lisäksi edistyneempiä strategioita:

- **Ositettu dataparallelismi** (_engl. sharded data parallelism_): Menetelmät kuten PyTorchin FSDP (_Fully Sharded Data Parallel_) ja DeepSpeedin ZeRO vähentävät päällekkäistä muistinkulutusta pilkkomalla koulutustilan GPU:iden kesken. Esimerkiksi ZeRO-1 osittaa optimoijan tilat, ja ZeRO-2 osittaa myös gradientit eri prosessien kesken.[^viehmannDLwithPytorch]
- **Sekvenssiparallelismi** (_engl. sequence parallelism_): Pitkät syötesekvenssit voidaan jakaa usealle GPU:lle huomionmekanismin (_engl. attention_) laskenta- ja muistivaatimusten pienentämiseksi, mikä tosin vaatii lisäkommunikaatiota GPU:iden välillä.
- **Asiantuntijaparallelismi** (_engl. expert parallelism_): _Mixture-of-Experts_ (MoE) -malleissa eri GPU:t ylläpitävät eri asiantuntija-aliverkkoja (_engl. experts_), ja reititin (_engl. router_) ohjaa syötteet valituille asiantuntijoille.[^freglyAISystems]

Käytännössä suuren mittakaavan koulutuksessa näitä strategioita yhdistellään. Kuten Viehmann[^viehmannDLwithPytorch] toteaa:

> "n-dimensional parallelism combines multiple parallelism strategies, such as data, model, pipeline, and tensor parallelism, to maximize scalability and efficiency for large-scale training."

## Työvaiheet

Tämä harjoitus tehdään vaiheittain. Koska aihepiiri sisältää paljon uutta terminologiaa, suurin osa koodista on annettu valmiina ja vain muutama rivi on jätetty täydennettäväksi. Tärkeintä on, että ajat jokaisen vaiheen Roihussa ja pystyt videolla selittämään, mitä kussakin vaiheessa tapahtuu.

!!! info "Projektinumero ja kirjautumissolmut"

    Korvaa kaikissa alla olevissa skripteissä ja komennoissa `project_XXXXXXX` oman CSC Student Projectisi numerolla (tai käytä aiemmilla viikoilla luomaasi `$DL2_PROJ`-ympäristömuuttujaa).

    Muista, että Roihussa on erilliset kirjautumissolmut CPU-puolelle (`roihu-cpu.csc.fi`, x86_64-arkkitehtuuri) ja GPU-puolelle (`roihu-gpu.csc.fi`, ARM/GH200-arkkitehtuuri). Levyalue `/projappl` näkyy kuitenkin molemmille, joten tiedostot tarvitsee kopioida Roihuun vain kerran.

### 1. Työhakemiston valmistelu

Luo omalle koneellesi hakemisto tämän viikon skripteille (esimerkiksi oman kurssirepositoriosi alle `07_parallelism/`) ja luo vastaava hakemisto Roihun `/projappl`-levyalueelle:

``` bash title="Oma kone"
ssh roihu-cpu mkdir -p /projappl/project_XXXXXXX/07_parallelism
```

Voit kopioida tiedostoja Roihuun sitä mukaa kun luot niitä, tai kaikki kerralla `scp`-komennolla:

``` bash title="Oma kone"
scp *.py *.sh roihu-cpu:/projappl/project_XXXXXXX/07_parallelism/
```

### 2. MPI: Prosessien välinen rinnakkaisuus

Aloitetaan puhtaasta MPI-esimerkistä. Roihun `python-pytorch`-moduuli sisältää valmiiksi `mpi4py`-kirjaston, joten erillisiä asennuksia ei tarvita.

Luo tiedosto `tester.py`. Esimerkin Python-koodi on kirjasta _Scientific Computing with Python_[^fuhrerSciComp]. Täydennä koodiin puuttuva `if`-lause siten, että prosessi `rank == 2` laskee potenssin `a**b` samalla tyylillä kuin prosessit 0 ja 1:

``` python title="tester.py"
from mpi4py import MPI

comm = MPI.COMM_WORLD  # making a communicator instance
rank = comm.Get_rank()  # querying for the numeric identifier of the core
size = comm.Get_size()  # the total number of cores assigned

a = 15
b = 2
if rank == 0:
    print(f"Core {rank} computes {a}+{b}={a+b}")
if rank == 1:
    print(f"Core {rank} computes {a}*{b}={a*b}")
# TODO: Lisää ehto rankille 2, joka tulostaa potenssin a**b
```

Luo seuraavaksi Slurm-työskripti `tester.sh`, joka varaa CPU-puolen `test`-partitiolta kolme MPI-tehtävää (`--ntasks=3`):

``` bash title="tester.sh"
#!/bin/bash
#SBATCH --job-name=mpi-tester
#SBATCH --account=project_XXXXXXX
#SBATCH --partition=test
#SBATCH --time=00:02:00
#SBATCH --ntasks=3
#SBATCH --mem-per-cpu=1000M
#SBATCH --chdir=/projappl/project_XXXXXXX/07_parallelism

module load python-pytorch
srun python3 tester.py
```

Kopioi `tester.py` ja `tester.sh` Roihuun, kirjaudu CPU-kirjautumissolmulle (`ssh roihu-cpu`) ja lähetä työ jonoon:

``` bash title="CSC Roihu (roihu-cpu)"
cd /projappl/project_XXXXXXX/07_parallelism
sbatch tester.sh
```

Kun työ on valmistunut, tarkista tuloste:

``` bash title="CSC Roihu (roihu-cpu)"
squeue --me
cat slurm-<jobid>.out
```

Tulosteen pitäisi näyttää seuraavanlaiselta (rivien järjestys voi vaihdella, sillä prosessit suoritetaan rinnakkain):

``` text
Core 1 computes 15*2=30
Core 2 computes 15**2=225
Core 0 computes 15+2=17
```

!!! note "Miksi `mpiexec`-komentoa ei tarvita Slurm-skriptissä?"

    Jos ajaisit ohjelman komentoriviltä ilman Slurmia, käynnistäisit kolme prosessia komennolla `mpiexec -n 3 python3 tester.py`. Slurm-eräajossa `srun` lukee suoraan `#SBATCH --ntasks=3` -varauksen ja käynnistää automaattisesti kolme MPI-prosessia (rankit 0, 1 ja 2).

### 3. OpenMP: Säietason rinnakkaisuus

Seuraavaksi kokeilemme yhden prosessin sisäistä säikeistystä OpenMP:n avulla. Käytämme `numba`-kirjastoa, joka kääntää Python-funktion konekieleksi (`@njit(parallel=True)`) ja rinnakkaistaa `prange`-silmukan OpenMP-säikeille.

Luo tiedosto `openmp.py`. Täydennä `parallel_sum`-funktion silmukkaan kaksi riviä: tallenna nykyisen säikeen tunniste taulukkoon `thread_ids[i]` funktiolla `get_thread_id()` ja kasvata muuttujaa `total` alkiolla `arr[i]`:

``` python title="openmp.py"
import numpy as np
from numba import njit, prange, get_num_threads, get_thread_id
from numba.np.ufunc.parallel import threading_layer


@njit(parallel=True)
def parallel_sum(arr, thread_ids):
    total = 0
    for i in prange(len(arr)):
        # TODO: Tallenna säikeen tunniste (get_thread_id()) taulukkoon thread_ids[i]
        # TODO: Lisää arr[i] muuttujaan total
        pass
    return total


data = np.arange(1, 9, dtype=np.int64)
thread_ids = np.empty(len(data), dtype=np.int32)

total = parallel_sum(data, thread_ids)

print(f"Threading layer: {threading_layer()}")
print(f"Active OpenMP threads: {get_num_threads()}")
for i in range(len(data)):
    print(f"Element {data[i]} processed by OpenMP thread {thread_ids[i]}")
print(f"Parallel sum: {total}")
```

Luo Slurm-työskripti `openmp.sh`. Huomaa ero edelliseen vaiheeseen: nyt pyydämme vain **yhden prosessin** (`--ntasks=1`), mutta varaamme sille **neljä prosessoriydintä** (`--cpus-per-task=4`):

``` bash title="openmp.sh"
#!/bin/bash
#SBATCH --job-name=python-openmp
#SBATCH --account=project_XXXXXXX
#SBATCH --partition=test
#SBATCH --time=00:02:00
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=4
#SBATCH --mem-per-cpu=1000M
#SBATCH --chdir=/projappl/project_XXXXXXX/07_parallelism

module load python-pytorch

# Asetetaan OpenMP-säikeiden määrä varattujen prosessoriydinten mukaan
export OMP_NUM_THREADS=${SLURM_CPUS_PER_TASK:-1}

# Sidotaan säikeet fyysisiin prosessoriytimiin
export OMP_PLACES=cores
export OMP_PROC_BIND=spread

# Ohjeistetaan Numbaa käyttämään OpenMP-taustajärjestelmää
export NUMBA_THREADING_LAYER=omp

srun --cpu-bind=cores python3 openmp.py
```

Kopioi tiedostot Roihuun ja lähetä työ jonoon `roihu-cpu`-solmulla:

``` bash title="CSC Roihu (roihu-cpu)"
sbatch openmp.sh
```

Tarkista tulosteesta (`cat slurm-<jobid>.out`), että `Threading layer: omp` ja `Active OpenMP threads: 4` tulostuvat oikein, kahdeksan alkiota jakautuvat säikeille 0–3 ja summaksi tulee `36`.

### 4. Hybridilaskenta: MPI + OpenMP

Yhdistetään nyt prosessitason (MPI) ja säietason (OpenMP) rinnakkaisuus. Alla olevassa `hybrid.py`-skriptissä jokainen MPI-prosessi selvittää oman `rank`-numeron ja solmun nimen. Sen jälkeen prosessi 0 lähettää viestin kaikille MPI-prosesseille `comm.bcast`-metodilla (_engl. broadcast_), ja lopuksi jokainen MPI-prosessi käynnistää neljä OpenMP-säiettä.

Luo tiedosto `hybrid.py`:

``` python title="hybrid.py"
from mpi4py import MPI
from numba import njit, prange, get_thread_id
import numpy as np


@njit(parallel=True)
def openmp_threads(thread_ids):
    for i in prange(len(thread_ids)):
        thread_ids[i] = get_thread_id()


# MPI information
comm = MPI.COMM_WORLD
rank = comm.Get_rank()
ntasks = comm.Get_size()
node = MPI.Get_processor_name()

print(f"Hello from MPI rank {rank} of {ntasks} on {node}")

# Rank 0 broadcasts a message to all MPI processes
message = "Hello from MPI rank 0" if rank == 0 else None
message = comm.bcast(message, root=0)

print(f"MPI rank {rank} received: {message}")

# Each MPI process starts four OpenMP threads
thread_ids = np.empty(4, dtype=np.int32)
openmp_threads(thread_ids)

for thread_id in thread_ids:
    print(f"MPI rank {rank}, OpenMP thread {thread_id}")
```

Luo Slurm-työskripti `hybrid.sh`, joka varaa **kaksi MPI-prosessia** (`--ntasks=2`) ja kummallekin **neljä prosessoriydintä** (`--cpus-per-task=4`), eli yhteensä $2 \times 4 = 8$ ydintä:

``` bash title="hybrid.sh"
#!/bin/bash
#SBATCH --job-name=python-hybrid
#SBATCH --account=project_XXXXXXX
#SBATCH --partition=test
#SBATCH --time=00:02:00
#SBATCH --nodes=1
#SBATCH --ntasks=2
#SBATCH --cpus-per-task=4
#SBATCH --mem-per-cpu=1000M
#SBATCH --chdir=/projappl/project_XXXXXXX/07_parallelism

module load python-pytorch

export OMP_NUM_THREADS=${SLURM_CPUS_PER_TASK:-1}
export OMP_PLACES=cores
export OMP_PROC_BIND=spread

# Tell Numba to use OpenMP
export NUMBA_THREADING_LAYER=omp

srun --cpu-bind=cores python3 hybrid.py
```

Kopioi `hybrid.py` ja `hybrid.sh` Roihuun ja aja työ `roihu-cpu`-solmulla:

``` bash title="CSC Roihu (roihu-cpu)"
sbatch hybrid.sh
```

Tarkista tulosteesta, että molemmat MPI-rankit (0 ja 1) vastaanottavat viestin rankilta 0 ja käynnistävät kumpikin omat OpenMP-säikeensä 0–3.

### 5. Hybridilaskenta GPU-solmulla (ARM-arkkitehtuuri)

Kokeillaan uteliaisuudesta, toimiiko täsmälleen sama `hybrid.py`-koodi myös Roihun GPU-solmuilla, joissa prosessorit ovat x86_64-arkkitehtuurin sijaan ARM-pohjaisia Nvidia Grace -prosessoreita (osa GH200 Superchip -kokonaisuutta).

Tee kopio `hybrid.sh`-skriptistä nimellä `hybrid_gpu.sh` ja tee siihen seuraavat muutokset:

1. Vaihda partitioksi `#SBATCH --partition=gputest`.
2. Poista rivi `#SBATCH --mem-per-cpu=1000M` ja lisää sen tilalle GPU-varaus `#SBATCH --gres=gpu:gh200:1` (koska `gputest` noudattaa allokaatiotyyppiä, jossa muisti määräytyy automaattisesti varattujen GPU:iden mukaan).

Kopioi `hybrid_gpu.sh` Roihuun, **vaihda GPU-kirjautumissolmulle** (`ssh roihu-gpu`) ja lähetä työ jonoon:

``` bash title="CSC Roihu (roihu-gpu)"
cd /projappl/project_XXXXXXX/07_parallelism
sbatch hybrid_gpu.sh
```

Tarkista `slurm-<jobid>.out`-tiedostosta, mille laskentasolmulle työ päätyi ja että tuloste vastaa CPU-puolen ajoa.

### 6. Monen GPU:n koulutus: PyTorch DDP

Siirrytään lopuksi varsinaiseen syväoppimisen dataparallelismiin usealla GPU:lla. Käytämme CSC:n virallisesta esimerkkirepositoriosta [CSCfi/pytorch-ddp-examples](https://github.com/CSCfi/pytorch-ddp-examples) löytyvää `mnist_ddp.py`-skriptiä, joka kouluttaa konvoluutioverkon MNIST-aineistolla käyttäen PyTorchin `DistributedDataParallel` (DDP) -luokkaa.

#### A: Lataa ja tutki `mnist_ddp.py`

Lataa `mnist_ddp.py` suoraan CSC:n GitHub-repositoriosta työhakemistoosi (voit tehdä tämän omalla koneellasi tai suoraan Roihussa):

``` bash
curl -O https://raw.githubusercontent.com/CSCfi/pytorch-ddp-examples/master/mnist_ddp.py
```

Avaa `mnist_ddp.py` editorissa ja etsi sieltä seuraavat dataparallelismin kannalta keskeiset kohdat (sinun ei tarvitse muokata tiedostoa, mutta sinun tulee osata selittää nämä kohdat videolla):

1. **Datan lataaminen vain yhdellä prosessilla (`RANK == 0`):**

``` python
if int(os.environ['RANK']) == 0:
    print('Downloading dataset (if needed) to', datadir)
    _ = MNIST(root=datadir, train=True, transform=transforms.ToTensor(),
              download=True)
```

Jos neljä prosessia yrittäisi ladata ja purkaa saman tiedoston `/scratch`-levylle täsmälleen yhtä aikaa, syntyisi kilpailutilanne (_engl. race condition_). Siksi vain prosessi 0 lataa aineiston ennen kuin prosessiryhmä alustetaan.

1. **Prosessiryhmän alustus ja laitteen valinta (`nccl` ja `LOCAL_RANK`):**

``` python
dist.init_process_group(backend='nccl')
rank_zero = dist.get_rank() == 0
device_id = int(os.environ['LOCAL_RANK'])
```

`nccl` (_NVIDIA Collective Communications Library_) on Nvidian GPU:iden väliseen nopeaan kommunikointiin optimoitu taustajärjestelmä. `LOCAL_RANK` (0, 1, 2 tai 3) kertoo prosessille, mitä solmun neljästä GPU:sta sen kuuluu käyttää.

1. **Mallin kääriminen `DistributedDataParallel`-luokkaan ja datan osittaminen (`DistributedSampler`):**

``` python
model = DistributedDataParallel(model, device_ids=[device_id])
...
train_sampler = DistributedSampler(train_dataset)
train_loader = DataLoader(dataset=train_dataset, batch_size=batch_size,
                          shuffle=False, num_workers=args.workers,
                          pin_memory=True, sampler=train_sampler)
```

`DistributedSampler` huolehtii siitä, että jokainen GPU saa jokaisella epookilla oman erillisen neljänneksensä koulutusdatasta. `DistributedDataParallel` puolestaan synkronoi gradientit kaikkien neljän GPU:n välillä automaattisesti `loss.backward()`-kutsun aikana.

#### B: Slurm-skripti neljälle GPU:lle (`run-ddp-gpu4.sh`)

Luo tiedosto `run-ddp-gpu4.sh` (mukautettu CSC:n `CSCfi/pytorch-ddp-examples`-repositoriosta Roihun `gputest`-partitiolle). Yhdellä Roihun GH200-solmulla on 4 GPU:ta ja 72 prosessoriydintä per GPU, eli koko solmun varauksessa pyydetään `--gres=gpu:gh200:4` ja `--cpus-per-task=288`:

``` bash title="run-ddp-gpu4.sh"
#!/bin/bash
#SBATCH --job-name=mnist-ddp
#SBATCH --account=project_XXXXXXX
#SBATCH --partition=gputest
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=288
#SBATCH --time=00:15:00
#SBATCH --gres=gpu:gh200:4
#SBATCH --chdir=/projappl/project_XXXXXXX/07_parallelism

module purge
module load python-pytorch

srun torchrun --standalone --nnodes=1 --nproc_per_node=4 mnist_ddp.py --epochs=100
```

Huomaa, että Slurm-tasolla `--ntasks=1`, koska `torchrun` käynnistää itse neljä työläisprosessia (`--nproc_per_node=4`) solmun sisällä ja asettaa niille `RANK`- ja `LOCAL_RANK`-ympäristömuuttujat.

#### C: Aja DDP-koulutus ja tarkastele resurssien käyttöä

Varmista, että sekä `mnist_ddp.py` että `run-ddp-gpu4.sh` ovat Roihussa hakemistossa `/projappl/project_XXXXXXX/07_parallelism/`, ja lähetä työ jonoon **`roihu-gpu`**-kirjautumissolmulta:

``` bash title="CSC Roihu (roihu-gpu)"
cd /projappl/project_XXXXXXX/07_parallelism
sbatch run-ddp-gpu4.sh
```

Seuraa työn etenemistä ja tarkista tuloste, kun ajo on valmis:

``` bash title="CSC Roihu (roihu-gpu)"
squeue --me
cat slurm-<jobid>.out
```

Tarkastele lopuksi valmistuneen työn resurssien käyttöä `seff`- ja `sacct`-komennoilla:

``` bash title="CSC Roihu (roihu-gpu)"
# Yhteenveto työn kestosta, CPU- ja muistitehokkuudesta sekä GPU-käytöstä
seff <jobid>

# Yksityiskohtaiset kirjanpitotiedot
sacct -j <jobid> --long
```

## Videolla esitettävä

1.  Selitä lyhyesti omin sanoin (käyttäen mielellään kuvaajia apuna):
    - Mitä tarkoittavat **dataparallelismi**, **tensor-parallelismi** ja **pipeline-parallelismi**, ja mitä kussakin niistä jaetaan GPU:iden kesken.
    - Mitä eroa on prosessitason (MPI) ja säietason (OpenMP) rinnakkaisuudella.

2.  Käynnistä `sbatch tester.sh` (`roihu-cpu`) ja näytä sen tuottama `slurm-<jobid>.out`-tuloste.

3.  Selitä, mitä edellisessä vaiheessa tapahtui:
    - Näytä `tester.py` ja `tester.sh`.
    - Kerro, mitä `MPI.COMM_WORLD`, `rank`, `size` sekä `#SBATCH --ntasks=3` tarkoittavat, ja miksi tulostusrivien järjestys voi vaihdella.

4.  Käynnistä `sbatch openmp.sh` (`roihu-cpu`) ja näytä sen tuloste.

5.  Selitä, mitä edellisessä vaiheessa tapahtui:
    - Näytä `openmp.py` ja `openmp.sh`.
    - Kerro, miten `#SBATCH --ntasks=1` ja `#SBATCH --cpus-per-task=4` eroavat MPI-ajosta, mitä ympäristömuuttujat `OMP_NUM_THREADS` ja `NUMBA_THREADING_LAYER=omp` tekevät, ja miten Numban `prange` jakaa silmukan säikeille. (_Vinkki: katso Numban dokumentaatio_)

6.  Käynnistä `sbatch hybrid.sh` (`roihu-cpu`) tai `sbatch hybrid_gpu.sh` (`roihu-gpu`) ja näytä tuloste.

7.  Selitä, mitä edellisessä vaiheessa tapahtui:
    - Näytä `hybrid.py` ja `hybrid(_gpu).sh`.
    - Kerro, miten MPI-prosessit (`--ntasks=2`) ja OpenMP-säikeet (`--cpus-per-task=4`) toimivat yhdessä, mitä `comm.bcast` tekee, ja miten `hybrid_gpu.sh` eroaa CPU-puolen skriptistä.

8.  Käynnistä `sbatch run-ddp-gpu4.sh` (`roihu-gpu`), näytä valmistuneen koulutuksen `slurm-<jobid>.out`-tuloste ja aja työlle komento `seff <jobid>`.

9.  Selitä, mitä edellisessä vaiheessa tapahtui:
    - Näytä `run-ddp-gpu4.sh` ja esittele `mnist_ddp.py`-tiedostosta keskeiset kohdat (miksi vain rank 0 lataa datan, mitä `nccl`, `LOCAL_RANK`, `DistributedSampler` ja `DistributedDataParallel` tekevät).
    - Tulkkaa `seff <jobid>` -komennon tulosteesta, kuinka kauan koulutus kesti ja miten tehokkaasti varatut resurssit olivat käytössä.

Kokonaisuutena videosta tulee ilmetä, että ymmärrät prosessi- ja säietason rinnakkaisuuden sekä syväoppimisen GPU-rinnakkaistamisstrategioiden peruskäsitteet, ja osaat suorittaa ja tulkita monen prosessin, säikeen ja GPU:n eräajoja Roihussa. Tulevaisuuden projekteissa tiedät, että resurssien varaaminen ilman rinnakkaistamista on turhaa yhteisten resurssien haaskausta.

[^viehmannDLwithPytorch]: Viehmann, T. _Deep Learning with PyTorch 2nd edition_. Manning. 2026.
[^huyenAIEngineering]: Huyen, C. _AI Engineering_. O'Reilly. 2024.
[^freglyAISystems]: Fregly, C. _AI Systems Performance Engineering_. O'Reilly. 2025.
[^fuhrerSciComp]: Führer, C., Solem, J. E. & Verdier, O. _Scientific Computing with Python 2nd edition_. Packt. 2021.
