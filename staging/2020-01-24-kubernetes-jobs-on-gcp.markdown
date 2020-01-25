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

## install helm to create job templates 

## install docker 

You have now installed all of the tools we will use in this tutorial -- on to kuberenetes-ing. 

# Create your kubernetes cluster

```
gcloud container clusters create \
       --machine-type n1-highmem-32 \
       --num-nodes 1 \
       --enable-autoscaling \
       --min-nodes=0 \
       --max-nodes=20 \
       --zone us-central1-b \
       --cluster-version latest \
       --disk-size=20Gi \
       spades
```
Let's examine what each of these flags are: 
* machine-type: determines the available cores and RAM. The available machine types are listed here: 


```
kubectl create clusterrolebinding cluster-admin-binding \
   --clusterrole=cluster-admin \
   --user=annashch@stanford.edu
```

Make sure kubectl is pointing to the cluster you just created: 

```
gcloud container clusters get-credentials spades --zone us-central1-b
```

# Create a helm chart for the cluster 


# Create and populate GCP storage bucket 

```
gsutil mb -l us-central1-b gs://keratinocytes/
```
Copy local files into your bucket: 


## Verify that the bucket can be mounted locally with gcsfuse 
First, obtain a json key file for mounting the bucket 
```
gcsfuse --key-file /etc/key.json --implicit-dirs  -o allow_other keratinocytes /mnt/data
```

# Create an account on dockerhub 

# Generate a dockerfile with your compute image 

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
docker login 
```

Tag your docker image and push it to dockerhub 

```
docker tag 
docker push kundajelab/spades_gcp:latest 
```
# Create kubernetes yaml files to submit your job 

```
apiVersion: batch/v1
kind: Job
metadata:
  name: spades0
spec:
  ttlSecondsAfterFinished: 30
  template:
    spec:
      containers:
      - name: spades
        image: kundajelab/spades_gcp:latest
        securityContext:
          privileged: true
          capabilities:
            add:
              - SYS_ADMIN
        command: ["/opt/script.sh","keratinocytes-0.5d-rep1"]
      restartPolicy: OnFailure
  backoffLimit: 1

```

# monitor job execution 

```
kubectl get all 
```

You can also visualize the job's logs via the GCP web console: 
