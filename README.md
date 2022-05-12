# helixer-docker

- Prerequisites (on host):
  - Nvidia GPU with CUDA capabilities >=3.5; installed driver version >= 450.80.02 

- Prepare, install nvidia docker runtime (on host), e.g. for ubuntu:
```
curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | sudo apt-key add -
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list |  sudo tee /etc/apt/sources.list.d/nvidia-docker.list
sudo apt update

sudo apt-get install nvidia-docker2
sudo pkill -SIGHUP dockerd 
```
Or follow instruction from https://github.com/NVIDIA/nvidia-docker



- Build:
```
mkdir SOME_DIR
cd SOME_DIR
wget https://raw.githubusercontent.com/gglyptodon/helixer-docker/main/Dockerfile
mkdir -p data/out
chmod o+w data/out # something the container can write to

docker build -t helixer_testing_tf11_2_cudnn8 --rm .
```


- Run:
```
docker run --runtime=nvidia -it --name helixer_testing --rm --mount type=bind,source="$(pwd)"/data,target=/home/helixer_user/shared helixer_testing_tf11_2_cudnn8:latest
```


- Try out:
```
helixer_user@03356047d15f:~$ cd shared/out/

helixer_user@03356047d15f:~/shared/out$ mkdir models
curl https://uni-duesseldorf.sciebo.de/s/4NqBSieS9Tue3J3/download --output models/land_plant.h5

helixer_user@03356047d15f:~/shared/out$ curl -L ftp://ftp.ensemblgenomes.org/pub/plants/release-47/fasta/arabidopsis_lyrata/dna/Arabidopsis_lyrata.v.1.0.dna.chromosome.8.fa.gz --output Arabidopsis_lyrata.v.1.0.dna.chromosome.8.fa.gz

helixer_user@03356047d15f:~/shared/out$ gunzip Arabidopsis_lyrata.v.1.0.dna.chromosome.8.fa.gz

helixer_user@03356047d15f:~/shared/out$ fasta2h5.py --species Arabidopsis_lyrata --h5-output-path Arabidopsis_lyrata.h5 --fasta-path Arabidopsis_lyrata.v.1.0.dna.chromosome.8.fa
# ->  Numerification of 0-22951293 of the sequence of 8 took 3.50 secs
# 1 Numerified Fasta only Coordinate (seqid: 8, len: 22951293)
# in 8.59 secs [...]

helixer_user@03356047d15f:~/shared/out$ ~/Helixer/helixer/prediction/HybridModel.py --load-model-path models/land_plant.h5 --test-data Arabidopsis_lyrata.h5 --overlap --val-test-batch-size 32 -v
# -->
# Total params: 2,105,672
# Trainable params: 2,105,096
# Non-trainable params: 576
# [...]

helixer_user@03356047d15f:~/shared/out$ helixer_post_bin Arabidopsis_lyrata.h5 predictions.h5 100 0.1 0.8 60 Arabidopsis_lyrata_chromosome8_helixer.gff3
# --> Total: 12727167bp across 2300 windows

```
-----------------------------------

Notes on running via Singularity 
---
For singularity install, see also: 
https://github.com/sylabs/singularity/blob/master/INSTALL.md

Install go:
```
export VERSION=1.18.1 OS=linux ARCH=amd64  # change this as you need

wget -O /tmp/go${VERSION}.${OS}-${ARCH}.tar.gz \
  https://dl.google.com/go/go${VERSION}.${OS}-${ARCH}.tar.gz
sudo tar -C /usr/local -xzf /tmp/go${VERSION}.${OS}-${ARCH}.tar.gz

echo 'export PATH=$PATH:/usr/local/go/bin' >> ~/.bashrc
source ~/.bashrc
```

```
mkdir  -p github.com/syslabs/
cd github.com/syslabs
git clone --recurse-submodules https://github.com/sylabs/singularity.git
cd singularity/
git checkout --recurse-submodules v3.9.9

./mconfig
make -C builddir
sudo make -C builddir install

```
```
singularity --version
# -->  singularity-ce version 3.9.9

```

```
# pull current docker image (properly versioned images shall be uploaded at some point in the near future 🤞 )
singularity pull  docker://gglyptodon/helixer-docker:helixer_tf11.2_cudnn8

# in this example, the directory "helixer_test" already contains downloaded data, models/land_plant.h5 is present etc 
singularity run helixer-docker_helixer_tf11.2_cudnn8.sif fasta2h5.py --species Arabidopsis_lyrata --h5-output-path Arabidopsis_lyrata.h5 --fasta-path helixer_test/Arabidopsis_lyrata.v.1.0.dna.chromosome.8.fa
# notice '--nv' for GPU support
singularity run --nv helixer-docker_helixer_tf11.2_cudnn8.sif /home/helixer_user/Helixer/helixer/prediction/HybridModel.py --load-model-path models/land_plant.h5 --test-data Arabidopsis_lyrata.h5 --overlap --val-test-batch-size 32 -v
singularity run helixer-docker_helixer_tf11.2_cudnn8.sif helixer_post_bin Arabidopsis_lyrata.h5 predictions.h5 100 0.1 0.8 60 Arabidopsis_lyrata_chromosome8_helixer.gff3
```
