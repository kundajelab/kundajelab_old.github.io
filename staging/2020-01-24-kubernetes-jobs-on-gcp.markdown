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

# Create a helm chart for the cluster 
