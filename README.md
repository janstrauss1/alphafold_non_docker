# AlphaFold Non-Docker setup on NEC HPC system at RZ CAU Kiel

## **Setup and installation**

## Log onto cluster using your username and password
Note: Ensure you have a VPN connection (Cisco AnyConnect Client)

``` bash
ssh -X <username>@nesh-fe.rz.uni-kiel.de
```
change to working directory using:

``` bash
cd $WORK
```

### **Install miniconda**
make conda directory
``` bash
mkdir conda
```
Enter conda directory
``` bash
cd conda
```
Get conda installer and install
``` bash
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh && bash Miniconda3-latest-Linux-x86_64.sh
```

### **Create a new conda environment and update**

``` bash
conda create --name alphafold python==3.8
conda update -n base conda
```

### **Activate conda environment**

``` bash
conda activate alphafold
```

### **Install dependencies**

- Change `cudnn==8.2.1.32` and `cudatoolkit==11.0.3` versions if they are not supported in your system (seems to be supported on NEC cluster)

``` bash
conda install -y -c conda-forge openmm==7.5.1 cudnn==8.2.1.32 cudatoolkit==11.0.3 pdbfixer==1.7
conda install -y -c bioconda hmmer==3.3.2 hhsuite==3.3.0 kalign2==2.04
```

### **Download alphafold [git repo](https://github.com/deepmind/alphafold.git)**

``` bash
git clone https://github.com/deepmind/alphafold.git
# alphafold_path="/path/to/alphafold/git/repo"
alphafold_path="/gxfs_work1/geomar/<username>/alphafold"
```

### **Download chemical properties to the common folder**

``` bash
wget -q -P alphafold/alphafold/common/ https://git.scicore.unibas.ch/schwede/openstructure/-/raw/7102c63615b64735c4941278d92b554ec94415f8/modules/mol/alg/src/stereo_chemical_props.txt
```

### **Install alphafold dependencies**

- Change `jaxlib==0.1.69+cuda<111>` version if this is not supported in your system (seems to be supported on NEC cluster)

_Note:_ jax updgrade: cuda111 supports cuda 11.3 - https://github.com/google/jax/issues/6628

``` bash
pip install absl-py==0.13.0 biopython==1.79 chex==0.0.7 dm-haiku==0.0.4 dm-tree==0.1.6 immutabledict==2.0.0 jax==0.2.14 ml-collections==0.1.0 numpy==1.19.5 scipy==1.7.0 tensorflow==2.5.0 pandas==1.3.4 tensorflow-cpu==2.5.0

pip install --upgrade jax jaxlib==0.1.69+cuda111 -f https://storage.googleapis.com/jax-releases/jax_releases.html
```

### **Apply OpenMM patch**

``` bash
# $alphafold_path variable is set to the alphafold git repo directory (absolute path)

# cd ~/anaconda3/envs/alphafold/lib/python3.8/site-packages/ && patch -p0 < $alphafold_path/docker/openmm.patch

# or

cd ~/miniconda3/envs/alphafold/lib/python3.8/site-packages/ && patch -p0 < $alphafold_path/docker/openmm.patch
```

### **Download all databases**

- Option 1: Use our [download_db.sh script](https://github.com/kalininalab/alphafold_non_docker/blob/main/download_db.sh) which uses wget, rsync, gunzip and tar instead of aria2c
    - Our script maintains the AF2 [download directory structure](https://github.com/deepmind/alphafold#genetic-databases)

To download download_db.sh script from github using wget
``` bash
alphafold_path="/gxfs_work1/geomar/<username>/alphafold"

wget -O download_db.sh https://raw.githubusercontent.com/kalininalab/alphafold_non_docker/main/download_db.sh -P alphafold_path
```

``` bash
# To use our download_db script (download the script first)
Usage: download_db.sh <OPTIONS>
Required Parameters:
-d <download_dir>     Absolute path to the AF2 download directory (example: /home/johndoe/alphafold_data)
Optional Parameters:
-m <download_mode>    full_dbs or reduced_dbs mode [default: full_dbs]

# To download all data (full_dbs mode)
# The script will create the folder </home/johndoe/alphafold_data> if it does not exist
bash download_db.sh -d </home/johndoe/alphafold_data>

# To download reduced version of the databases (reduced_dbs mode)
# The script will create the folder </home/johndoe/alphafold_data> if it does not exist
bash download_db.sh -d </home/johndoe/alphafold_data> -m reduced_dbs
```

``` bash
# alphafold_database="/gxfs_work1/geomar/<username>/alphafold/alphafold_database"
# bash download_db.sh -d /gxfs_work1/geomar/<username>/alphafold/alphafold_database -m full_dbs

bash download_db.sh -d /gxfs_work1/geomar/<username>/alphafold/alphafold_database -m reduced_dbs
```

- Option 2: Follow https://github.com/deepmind/alphafold#genetic-databases

### **Updating existing AlphaFold installation to include AlphaFold-Multimers (v2.1.1)**
- Please refer to [this section](https://github.com/deepmind/alphafold#updating-existing-alphafold-installation-to-include-alphafold-multimers)


## **Running alphafold (v2.1.1)**

First create some folders to store sequences and results
``` bash
cd /gxfs_work1/geomar/<username>/alphafold/
mkdir alphafold_runs
cd alphafold_runs
mkdir sequences
mkdir results
```

Download a sequence
``` bash
cd /gxfs_work1/geomar/<username>/alphafold/alphafold_runs/sequences
wget -O query.fasta https://www.uniprot.org/uniprot/A0A1E7EXA4.fasta
```

``` bash
alphafold_path="/gxfs_work1/geomar/<username>/alphafold"

wget -O run_alphafold.sh https://raw.githubusercontent.com/kalininalab/alphafold_non_docker/main/run_alphafold.sh -P alphafold_path
```


- Use this [bash script](https://github.com/kalininalab/alphafold_non_docker/blob/main/run_alphafold.sh)

``` bash
Usage: ./run_alphafold_v21.sh <OPTIONS>
Required Parameters:
-d <data_dir>         Path to directory of supporting data (e.g. /gxfs_work1/geomar/smomw453/ref_DBs/AlphaFold)
-o <output_dir>       Path to a directory that will store the results.
-f <fasta_path>       Path to a FASTA file containing sequence. If a FASTA file contains multiple sequences, then it will be folded as a multimer
-t <max_template_date> Maximum template release date to consider (ISO-8601 format - i.e. YYYY-MM-DD). Important if folding historical test sets
Optional Parameters:
-g <use_gpu>          Enable NVIDIA runtime to run with GPUs (default: true)
-n <openmm_threads>   OpenMM threads (default: all available cores)
-a <gpu_devices>      Comma separated list of devices to pass to 'CUDA_VISIBLE_DEVICES' (default: 0)
-m <model_preset>     Choose preset model configuration - the monomer model, the monomer model with extra ensembling, monomer model with pTM head, or multimer model (default: 'monomer')
-c <db_preset>        Choose preset MSA database configuration - smaller genetic database config (reduced_dbs) or full genetic database config (full_dbs) (default: 'full_dbs')
-p <use_precomputed_msas> Whether to read MSAs that have been written to disk. WARNING: This will not check if the sequence, database or configuration have changed (default: 'false')
-l <is_prokaryote>    Optional for multimer system, not used by the single chain system. A boolean specifying true where the target complex is from a prokaryote, and false where it is not, or where the origin is unknown. This value determine the pairing method for the MSA (default: 'None')
-b <benchmark>        Run multiple JAX model evaluations to obtain a timing that excludes the compilation time, which should be more indicative of the time required for inferencing many proteins (default: 'false')
```

- This script needs to be put into the top directory of the alphafold git repo that you have downloaded

```
# Directory structure
alphafold
├── alphafold
├── CONTRIBUTING.md
├── docker
├── example
├── imgs
├── LICENSE
├── README.md
├── requirements.txt
├── run_alphafold.py
├── run_alphafold.sh    <--- Copy the bash script and put it here
├── run_alphafold_test.py
├── scripts
└── setup.py
```

- Put your query sequence in a fasta file <filename.fasta>. 

    - In the below example query sequence was obtained from [here](https://colab.research.google.com/drive/1qWO6ArwDMeba1Nl57kk_cQ8aorJ76N6x)
- Running the script

```
# Example run (Uses the GPU with index id 0 as default)
# bash run_alphafold.sh -d ./alphafold_data/ -o ./dummy_test/ -f ./example/query.fasta -t 2020-05-14
bash run_alphafold.sh -d /gxfs_work1/geomar/smomw453/ref_DBs/AlphaFold -o ./dummy_test/ -f ./example/query.fasta -t 2020-05-14
bash run_alphafold.sh -d /gxfs_work1/geomar/smomw453/ref_DBs/AlphaFold -o ./alphafold_runs/results -f ./alphafold_runs/sequences/query.fasta -t 2020-05-14

# OR for CPU only run
# bash run_alphafold.sh -d ./alphafold_data/ -o ./dummy_test/ -f ./example/query.fasta -t 2020-05-14 -g False
bash run_alphafold.sh -d /gxfs_work1/geomar/smomw453/ref_DBs/AlphaFold -o ./dummy_test/ -f ./example/query.fasta -t 2020-05-14 -g False
bash run_alphafold.sh -d /gxfs_work1/geomar/smomw453/ref_DBs/AlphaFold -o ./alphafold_runs/results -f ./alphafold_runs/sequences/query.fasta -t 2020-05-14 -g False

```

- The results folder `dummy_test` can be found in this git repo along with the query (`example/query.fasta`) used
- The arguments to the script follows the original naming of the alphafold parameters, except for `fasta_paths`. This script can do only one fasta query at a time. So use a terminal multiplexer (example: tmux/screen) to do multiple runs.
- One can also control the number of cores used by OpenMM using the `-n` argument (dafult: uses all available cores)
- For further information refer [here](https://github.com/deepmind/alphafold)

### **Running AlphaFold-Multimer**
- All steps are the same as when running the monomer system, but you will have to 
    - provide an input fasta with multiple sequences,
    - set *-m multimer* option when running *run_alphafold.sh* script,
    - optionally set the *-l* option with true or false that determine whether all input sequences in the given fasta file are prokaryotic. If that is not the case or the origin is unknown then set to false.

    ```
    # Example run (Uses the GPU with index id 0 as default)
    bash run_alphafold.sh -d alphafold_data/ -o dummy_test/ -f multimer_query.fasta -t 2021-11-01 -m multimer -l true
    ```

### **Examples (Modified from [AF2](https://github.com/deepmind/alphafold#examples))**

Below are examples on how to use AlphaFold in different scenarios.

#### **Folding a monomer**

Say we have a monomer with the sequence `<SEQUENCE>`. The input fasta should be:

```fasta
>sequence_name
<SEQUENCE>
```

Then run the following command:

```bash
bash run_alphafold.sh -d alphafold_data/ -o dummy_test/ -f monomer.fasta -t 2021-11-01 -m monomer
```

#### **Folding a homomer**

Say we have a homomer from a prokaryote with 3 copies of the same sequence
`<SEQUENCE>`. The input fasta should be:

```fasta
>sequence_1
<SEQUENCE>
>sequence_2
<SEQUENCE>
>sequence_3
<SEQUENCE>
```

Then run the following command:

```bash
bash run_alphafold.sh -d alphafold_data/ -o dummy_test/ -f homomer.fasta -t 2021-11-01 -m multimer -l true
```

#### **Folding a heteromer**

Say we have a heteromer A2B3 of unknown origin, i.e. with 2 copies of
`<SEQUENCE A>` and 3 copies of `<SEQUENCE B>`. The input fasta should be:

```fasta
>sequence_1
<SEQUENCE A>
>sequence_2
<SEQUENCE A>
>sequence_3
<SEQUENCE B>
>sequence_4
<SEQUENCE B>
>sequence_5
<SEQUENCE B>
```

Then run the following command:

```bash
bash run_alphafold.sh -d alphafold_data/ -o dummy_test/ -f heteromer.fasta -t 2021-11-01 -m multimer -l false
```

## **API changes between v2.0.0 and v2.1.1**
- The preset flag *-p* was split into *-c* (db_preset) and *-m* (model_preset) in our *run_alphafold.sh*
    - Four model presets (for option *-m*) are now supported
        - monomer
        - monomer_casp14
        - monomer_ptm
        - multimer
    - Two db preset configurations (for option *-c*) are supported
        - full_dbs
        - reduced_dbs
- The model names to use are not specified using *-m* option anymore. If you want to customize model names you will have to modify the appropriate MODEL_PRESETS dictionary in *alphafold/model/config.py*

## **Disclaimer**

- We do not guarantee that this will work for everyone
- The non-docker version was tested with the following system configuration 
    - Dell server
        - CPU: AMD EPYC 7601 2.2 GHz
        - RAM: 1 TB
        - GPU: NVIDIA Tesla V100 16G
        - OS: CentOS 7 (kernel 3.10.0-1160.24.1.el7.x86_64)
        - Cuda: 11.3
        - NVIDIA driver version: 470.42.01
    - Storage
        - Downloaded database size: 2.2 TB (uncompressed)
