---
layout: post
title:  "Kubernetes Jobs on GCP "
date:   2020-01-24 23:02:47 -0800
categories: Programming
author: 'Anna Shcherbina'
---

Disclaimer: these instructions were tested for ubuntu 16.04 and ubuntu 18.04. Specific commands might not generalize for different operating systems. 

This tutorial explains how to run a batch of compute jobs on the Google Compute Platform (GCP) cloud using kubernetes, helm, docker, and gcsfuse. 

For the purposes of this tutorial, we will perform the task of running the SPAdes genomic assembler on a number of genomic (fastq) source files. 

# Install the needed prerequistes: 

## Install gcloud and gsutil
```
curl https://sdk.cloud.google.com | bash
exec -l $SHELL
gcloud init 
```

Authenticate with gcloud 
```
gcloud auth login 
```
You might need to configure your default gcp project. First check the configuration settings: 

```
gcloud config list 
```

If the proper project is not selected, you can change it: 

```
gcloud cnofig projects list
gcloud config set project "my-project"
```
## install kube 

It is recommended to install kube through gcloud if you will use it with GCP: 

```
gcloud components install kubectl
```
A stand-alone installer is also available, but not recommended for GCP clusters:
https://kubernetes.io/docs/tasks/tools/install-kubectl/

## install gcsfuse for mounting gcp buckets 
```
RUN apt-get update && apt-get install --yes --no-install-recommends \
    ca-certificates \
    curl \
    gnupg \
    && echo "deb http://packages.cloud.google.com/apt $GCSFUSE_REPO main" | tee /etc/apt/sources.list.d/gcsfuse.list \
    && curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add - \
    && apt-get update \
    && apt-get install gcsfuse -y \
    && apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/* 
```

Create a service account for mounting the bucket: 
https://console.cloud.google.com/iam-admin/serviceaccounts?project=gbsc-gcp-lab-kundaje
(replace 'gbsc-gcp-lab-kundaje' with the relevant project id) 

Create a service account, and download a key in json form. 

![go_to_service_account](https://github.com/kundajelab/kundajelab.github.io/blob/master/images/2020-01-24-kubernetes-jobs-on-gcp/service_account_gcp.png?raw=true)


![create_key](https://github.com/kundajelab/kundajelab.github.io/blob/master/images/2020-01-24-kubernetes-jobs-on-gcp/create_key.png?raw=true)


![create_key2](https://github.com/kundajelab/kundajelab.github.io/blob/master/images/2020-01-24-kubernetes-jobs-on-gcp/create_key2.png?raw=true)



## install docker 


```
sudo apt install docker.io
sudo systemctl start docker
sudo systemctl enable docker

```


# Create your kubernetes cluster
You have now installed all of the tools we will use in this tutorial -- on to kuberenetes-ing. 

```
gcloud container clusters create \
       --machine-type n1-highmem-32 \
       --num-nodes 1 \
       --enable-autoscaling \
       --min-nodes=0 \
       --max-nodes=60 \
       --zone us-central1-b \
       --cluster-version latest \
       --disk-size=20Gi \
       spades
```
Let's examine what each of these flags are: 
* machine-type: determines the available cores and RAM. The available machine types are listed here: https://cloud.google.com/compute/docs/machine-types
* num-nodes: the initial number of nodes to create in our cluster (within the default node pool). 
* enable-autoscaling: allows the cluster to grow and shrink in number of nodes based on compute demands 
* min-nodes: the minimum number of nodes the cluster can reach when autoscaling is enabled (I like to set this to 0 to avoid wasting resources). 
* max-nodes: the maximum number of compute nodes that can be created in the cluster. 
* zone: geographic zone in which the cluster is create (generally this only matters for compatibilitly with other resources such as storage buckets) 
* disk-size: the HDD size of each node in the cluster. Since we will be running our jobs inside docker images and mounting data from buckets, this value can be relatively small. 

The name of the cluster we created is "spades". 

set your GCP account as an admin account on the cluster: 
```
kubectl create clusterrolebinding cluster-admin-binding \
   --clusterrole=cluster-admin \
   --user=annashch@stanford.edu
```

Make sure kubectl is pointing to the cluster you just created: 

```
gcloud container clusters get-credentials spades --zone us-central1-b
```

# Create and populate GCP storage bucket 

```
gsutil mb -l us-central1-b gs://keratinocytes/
```
This creates a bucket called "keratinocytes" in the compute zone "us-central1-b". Make sure the zone name matches what you set in the cluster specification above. 

Copy local files into your bucket: 

```
gsutil cp -r [local_files_or_folder] gs://keratinocytes/
```
make sure the files were copied 

```
gsutil ls gs://kerationcytes/
```

## Verify that the bucket can be mounted locally with gcsfuse 
First, obtain a json key file for mounting the bucket (see above)


```
mkdir /mnt/data
gcsfuse --key-file key.json --implicit-dirs  -o allow_other keratinocytes /mnt/data
ls /mnt/data
```
Note: it is not intuitive that the bucket name does not need to be prefixed by "gs://" when the mount command is executed. 
To unmount the bucket, run: 

```
fusermount -u /mnt/data
```

# Create an account on dockerhub 
Refer to instructions here: https://hub.docker.com/signup
Dockerhub accounts are free 


# Generate a dockerfile with your compute image 

This is an example for our specific task. In this dockerfile we: 

1) start with a base ubuntu bioinic image 
2) Install the gcsfuse software for mounting gcp buckets. (We already did this on our local machine, the same series of commands can be added to the Dockerfile to install gcsfuse within the docker image). 
3) copy the key.json file for mounting gcfuse buckets to the image
4) copy additional scripts for mounting the gs://keratinocytes bucket, unmounting the bucket, and running our script of interest (the SPAdes assembler in this case). 
5) Install the spades assembler. 

Dockerfile:
```
FROM ubuntu:bionic

ENV GCSFUSE_REPO=gcsfuse-bionic

RUN apt-get update && apt-get install --yes --no-install-recommends \
    ca-certificates \
    curl \
    gnupg \
    wget \
    python \
    python-dev \
    && echo "deb http://packages.cloud.google.com/apt $GCSFUSE_REPO main" | tee /etc/apt/sources.list.d/gcsfuse.list \
    && curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add - \
    && apt-get update \
    && apt-get install gcsfuse -y \
    && apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
RUN mkdir /mnt/data


COPY key.json /etc/key.json
COPY mount.sh /opt/mount.sh
COPY umount.sh /opt/umount.sh


#install spades

WORKDIR /opt
RUN wget http://cab.spbu.ru/files/release3.14.0/SPAdes-3.14.0-Linux.tar.gz
RUN tar -xzvf SPAdes-3.14.0-Linux.tar.gz
ENV PATH="/opt/SPAdes-3.14.0-Linux/bin:${PATH}"
COPY script.sh /opt/script.sh

RUN chmod 777 /opt/*sh
CMD ["sleep", "360"]
```

In this Dockerfile, we copy "key.json", "mount.sh", "umoount.sh", and "script.sh" to the docker image. 
The key file is the same as you obtained in step XX. 

The contents of the other files are as follows: 

mount.sh: 
```
#!/bin/bash
gcsfuse --key-file /etc/key.json --implicit-dirs  -o allow_other keratinocytes /mnt/data
```
umount.sh: 
```
#!/bin/bash
fusermount -u /mnt/data
```

script.sh: 
```
#!/bin/bash
echo "running mount" 
bash /opt/mount.sh
echo "mount successful"
echo "running spades"
spades.py --pe1-1 /mnt/data/$1.end1.trimmed.fastq.gz --pe1-2 /mnt/data/$1.end2.trimmed.fastq.gz -t 20 -o /mnt/data/SPAdes-3.14-$1
echo "ran ls" 
bash /opt/umount.sh
echo "unmounting"
```

Now, build and push your image to dockerhub 

```
cd <directory-where-Dockerfile-is-stored>
docker build -t spades_gcp . 
```

Use the `docker images` command to view your newly created image: 

```
docker images 
```
The output will look similar to this: 

```
REPOSITORY                                           TAG                             IMAGE ID            CREATED             SIZE
kundajelab/spades_gcp                                latest                          eb64bbfe1edb        16 hours ago        288MB
```


Tag your docker image and push it to dockerhub 

```
docker tag eb64bbfe1edb kundajelab/spades_gcp:eb64bbfe1edb
```
The syntax above indicates that image with id eb64bbfe1edb will be pushed to the kundajelab dockerhub organization (you can use your own user account in place of kundajelab), to a repository called spades_gcp, with unique tag eb64bbfe1edb. The tag can be whatever you wish, a common practice is to use the word "latest" -- but my preference is to have each remote tag be a unique identifier. 

Now, login to dockerhub and push your built image: 

```
docker login
````
You will be prompted to provide the username and password you created for dockerhub above. 

```
docker push kundajelab/spades_gcp:eb64bbfe1edb 
```

# Create kubernetes yaml files to submit your job -- If you only need to run a handful of jobs


spades_job.yaml:
```
apiVersion: batch/v1
kind: Job
metadata:
  name: spades1
spec:
  ttlSecondsAfterFinished: 30
  template:
    spec:
      containers:
      - name: spades
        image: kundajelab/spades_gcp:latest
        resources:
          requests:
            memory: 50Gi
            cpu: 15
          limits:
            memory: 100Gi
            cpu: 15
        securityContext:
          privileged: true
          capabilities:
            add:
              - SYS_ADMIN
        command: ["/opt/script.sh"]
        args: ["keratinocytes-0.5d-rep1"]
      restartPolicy: OnFailure
  backoffLimit: 1
```
Let's break down what we have specified in the yaml file: 


Submit the job as follows: 

```
kubectl apply -f spades_job.yaml
```


# monitor job execution 

```
kubectl get all 
```
gives: 
```
NAME               READY   STATUS    RESTARTS   AGE
pod/spades-hsrx5   1/1     Running   0          14h

NAME                 TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.35.240.1   <none>        443/TCP   15h

NAME               COMPLETIONS   DURATION   AGE
job.batch/spades   0/1           14h        14h
```

Use the "describe" command to check the status of a job or pod. 

```
kubectl describe job.batch/spades
```
tells us the job is running: 
```
Name:           spades
Namespace:      default
Selector:       controller-uid=059cc91e-0e02-4ecd-89aa-3d347c55579e
Labels:         controller-uid=059cc91e-0e02-4ecd-89aa-3d347c55579e
                job-name=spades
Annotations:    kubectl.kubernetes.io/last-applied-configuration:
                  {"apiVersion":"batch/v1","kind":"Job","metadata":{"annotations":{},"name":"spades","namespace":"default"},"spec":{"backoffLimit":1,"templa...
Parallelism:    1
Completions:    1
Start Time:     Sat, 25 Jan 2020 03:16:44 -0800
Pods Statuses:  1 Running / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  controller-uid=059cc91e-0e02-4ecd-89aa-3d347c55579e
           job-name=spades
  Containers:
   spades:
    Image:      kundajelab/spades_gcp:latest
    Port:       <none>
    Host Port:  <none>
    Command:
      /opt/script.sh
      keratinocytes-0.5d-rep1
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Events:           <none>
```
And same for a pod: 
```
kubectl describe pod/spades-hsrx5
```

```
Name:           spades-hsrx5
Namespace:      default
Priority:       0
Node:           gke-spades-default-pool-eeea0ebb-3144/10.128.0.42
Start Time:     Sat, 25 Jan 2020 03:16:44 -0800
Labels:         controller-uid=059cc91e-0e02-4ecd-89aa-3d347c55579e
                job-name=spades
Annotations:    kubernetes.io/limit-ranger: LimitRanger plugin set: cpu request for container spades
Status:         Running
IP:             10.32.0.16
IPs:            <none>
Controlled By:  Job/spades
Containers:
  spades:
    Container ID:  docker://3fcd4d0ba692d6d5ab0bb73ef173af094deb2901fc9e619ef6d2e74f65f4a8f4
    Image:         kundajelab/spades_gcp:latest
    Image ID:      docker-pullable://kundajelab/spades_gcp@sha256:a2f214a93ffeebc65d40e1edfcaff552f5b42d12b71ec4d5abb59adb51176adf
    Port:          <none>
    Host Port:     <none>
    Command:
      /opt/script.sh
      keratinocytes-0.5d-rep1
    State:          Running
      Started:      Sat, 25 Jan 2020 03:16:52 -0800
    Ready:          True
    Restart Count:  0
    Requests:
      cpu:        100m
    Environment:  <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-hn2rr (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  default-token-hn2rr:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-hn2rr
    Optional:    false
QoS Class:       Burstable
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:          <none>

```
Use the "logs" command to view a pod's logs. 

```
kubectl logs pod/spades-hsrx5
```
```
running mount
Using mount point: /mnt/data
Opening GCS connection...
Opening bucket...
Mounting file system...
File system has been successfully mounted.
mount successful
running spades


== Warning ==  output dir is not empty! Please, clean output directory before run.




== Warning ==  No assembly mode was sepcified! If you intend to assemble high-coverage multi-cell/isolate data, use '--isolate' option.


Command line: /opt/SPAdes-3.14.0-Linux/bin/spades.py	--pe1-1	/mnt/data/keratinocytes-0.5d-rep1.end1.trimmed.fastq.gz	--pe1-2	/mnt/data/keratinocytes-0.5d-rep1.end2.trimmed.fastq.gz	-t	20	-o	/mnt/data/SPAdes-3.14-keratinocytes-0.5d-rep1	

System information:
  SPAdes version: 3.14.0
  Python version: 2.7.17
  OS: Linux-4.19.76+-x86_64-with-Ubuntu-18.04-bionic

Output dir: /mnt/data/SPAdes-3.14-keratinocytes-0.5d-rep1
Mode: read error correction and assembling
Debug mode is turned OFF

Dataset parameters:
  Standard mode
  For multi-cell/isolate data we recommend to use '--isolate' option; for single-cell MDA data use '--sc'; for metagenomic data use '--meta'; for RNA-Seq use '--rna'.
  Reads:
    Library number: 1, library type: paired-end
      orientation: fr
      left reads: ['/mnt/data/keratinocytes-0.5d-rep1.end1.trimmed.fastq.gz']
      right reads: ['/mnt/data/keratinocytes-0.5d-rep1.end2.trimmed.fastq.gz']
      interlaced reads: not specified
      single reads: not specified
      merged reads: not specified
Read error correction parameters:
  Iterations: 1
  PHRED offset will be auto-detected
  Corrected reads will be compressed
Assembly parameters:
  k: automatic selection based on read length
  Repeat resolution is enabled
  Mismatch careful mode is turned OFF
  MismatchCorrector will be SKIPPED
  Coverage cutoff is turned OFF
Other parameters:
  Dir for temp files: /mnt/data/SPAdes-3.14-keratinocytes-0.5d-rep1/tmp
  Threads: 20
  Memory limit (in Gb): 204


======= SPAdes pipeline started. Log can be found here: /mnt/data/SPAdes-3.14-keratinocytes-0.5d-rep1/spades.log

/mnt/data/keratinocytes-0.5d-rep1.end2.trimmed.fastq.gz: max reads length: 76
/mnt/data/keratinocytes-0.5d-rep1.end1.trimmed.fastq.gz: max reads length: 76

Reads length: 76


===== Read error correction started. 


===== Read error correction started. 


== Running: /opt/SPAdes-3.14.0-Linux/bin/spades-hammer /mnt/data/SPAdes-3.14-keratinocytes-0.5d-rep1/corrected/configs/config.info

  0:00:00.000     6M / 13M   INFO    General                 (main.cpp                  :  75)   Starting BayesHammer, built from refs/heads/spades_3.14.0, git revision c831f9be30a4364383cd4ebe5b78cdfcaad1acc9
  0:00:00.000     6M / 13M   INFO    General                 (main.cpp                  :  76)   Loading config from /mnt/data/SPAdes-3.14-keratinocytes-0.5d-rep1/corrected/configs/config.info
  0:00:00.186     7M / 13M   INFO    General                 (main.cpp                  :  78)   Maximum # of threads to use (adjusted due to OMP capabilities): 20
  0:00:00.186     7M / 13M   INFO    General                 (memory_limit.cpp          :  49)   Memory limit set to 204 Gb
  0:00:00.186     7M / 13M   INFO    General                 (main.cpp                  :  86)   Trying to determine PHRED offset
  0:00:00.215     7M / 13M   INFO    General                 (main.cpp                  :  92)   Determined value is 33
  0:00:00.215     7M / 13M   INFO    General                 (hammer_tools.cpp          :  38)   Hamming graph threshold tau=1, k=21, subkmer positions = [ 0 10 ]
  0:00:00.215     7M / 13M   INFO    General                 (main.cpp                  : 113)   Size of aux. kmer data 24 bytes
     === ITERATION 0 begins ===
  0:00:01.378     7M / 13M   INFO   K-mer Index Building     (kmer_index_builder.hpp    : 301)   Building kmer index
  0:00:01.378     7M / 13M   INFO    General                 (kmer_index_builder.hpp    : 117)   Splitting kmer instances into 320 files using 20 threads. This might take a while.
  0:00:01.821     7M / 13M   INFO    General                 (file_limit.hpp            :  32)   Open file limit set to 1048576
  0:00:01.821     7M / 13M   INFO    General                 (kmer_splitters.hpp        :  89)   Memory available for splitting buffers: 3.39999 Gb
  0:00:01.821     7M / 13M   INFO    General                 (kmer_splitters.hpp        :  97)   Using cell size of 209715
  0:00:01.843    13G / 13G   INFO   K-mer Splitting          (kmer_data.cpp             :  97)   Processing /mnt/data/keratinocytes-0.5d-rep1.end1.trimmed.fastq.gz
  0:17:53.388    13G / 13G   INFO   K-mer Splitting          (kmer_data.cpp             : 107)   Processed 13160002 reads
  0:37:05.984    13G / 13G   INFO   K-mer Splitting          (kmer_data.cpp             : 107)   Processed 26221405 reads
  0:56:19.286    13G / 13G   INFO   K-mer Splitting          (kmer_data.cpp             : 107)   Processed 38443123 reads
  1:15:33.612    13G / 13G   INFO   K-mer Splitting          (kmer_data.cpp             : 107)   Processed 49051279 reads
  1:36:03.143    13G / 13G   INFO   K-mer Splitting          (kmer_data.cpp             : 107)   Processed 59734618 reads
  1:58:01.606    13G / 13G   INFO   K-mer Splitting          (kmer_data.cpp             : 107)   Processed 70469203 reads
  2:23:03.385    13G / 13G   INFO   K-mer Splitting          (kmer_data.cpp             : 107)   Processed 83566956 reads
  2:49:17.334    13G / 13G   INFO   K-mer Splitting          (kmer_data.cpp             : 107)   Processed 96591029 reads
  3:17:56.190    13G / 13G   INFO   K-mer Splitting          (kmer_data.cpp             : 107)   Processed 109437559 reads
  3:48:39.303    13G / 13G   INFO   K-mer Splitting          (kmer_data.cpp             : 107)   Processed 120712614 reads
  4:22:22.591    13G / 13G   INFO   K-mer Splitting          (kmer_data.cpp             : 107)   Processed 131454394 reads
  4:58:23.120    13G / 13G   INFO   K-mer Splitting          (kmer_data.cpp             : 107)   Processed 142132818 reads
  5:36:46.765    13G / 13G   INFO   K-mer Splitting          (kmer_data.cpp             : 107)   Processed 152937013 reads
  5:36:46.766    13G / 13G   INFO   K-mer Splitting          (kmer_data.cpp             :  97)   Processing /mnt/data/keratinocytes-0.5d-rep1.end2.trimmed.fastq.gz
 14:16:05.379    13G / 13G   INFO   K-mer Splitting          (kmer_data.cpp             : 107)   Processed 274599076 reads
```

# Submitting a large number of jobs 

This gets tricky... GCP and AWS functionality is largely analogus with the exception of batch job submission, as documented in this service comparison released by GCP in November 2018 https://cloud.google.com/docs/compare/aws#service_comparisons 

Some tools that may help achieve the desired SLURM-like functionality are: 

* helm charts https://helm.sh/
* GKE Batch (currently in beta): https://cloud.google.com/batch/

These are highly useful tools for production-quality kubernetes clusters and worth learning -- but the learning curve can be a bit steep. (Comment below if you've had success running GCP batch jobs with these tools). 

If you're a grad student trying to process some data or train some machine learning models, a "hack" is as follows: 

1) Create a template yaml file with placeholders for the job name and the script arguments, as follows: 

```

```
