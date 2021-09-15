# Hybrid and Sovereign AI on Anthos Bare Metal

# Table of Contents

<!-- toc -->
* [Overview](#overview)
* [Terraform as IaC Substrate](#terraform-as-iac-substrate)
* [ABM Cluster on GCE using Terraform](#abm-gce-cluster-using-terraform)
* [TensorFlow ResNet Model Serving on ABM](#serving-tensorflow-resnet-model-on-abm)
* [TensorFlow Training using Kubeflow TFJob](training#tensorflow-mnist-training-on-abm-using-kubeflow-tfjob)
* Deploying TensorFlow ResNet Model on ABM with Anthos Config Management (Coming Soon!)
* Deploying TensorFlow ML GPU Model on ABM with ACM (Coming Soon!)

<!-- tocstop -->

## Overview
AI and Machine Learning workflows using TensorFlow on [Anthos Bare Metal](https://cloud.google.com/anthos/clusters/docs/bare-metal/1.6). TensorFlow is one of the most popular ML frameworks ([10M+ downloads](https://pypistats.org/packages/tensorflow) per month) in use today, but at the same time presents a lot of challenges when it comes to setup (GPUs, CUDA Drivers, TF Serving etc), performance tuning, cluster provisioning, maintenance, and model serving. This work will showcase the easy to use guides for ML model serving, training, infrastructure, ML Notebooks, and more on Anthos Bare Metal.

## Terraform as IaC Substrate
[Terraform](https://www.terraform.io/) is an open-source infrastructure as code software tool, and one of the ways in which Enterprise IT teams create, manage, and update infrastructure resources such as physical machines, VMs, switches, containers, and more. Provisioning the hardware or resources is always the first step in the process and these guides will be using Terraform as a common substrate to create the infrastructure for AI/ML apps. Checkout our upstream contribution to the Google Terraform Provider for [GPU support](https://github.com/terraform-google-modules/terraform-google-vm/pull/160) in the [instance_template module](https://github.com/terraform-google-modules/terraform-google-vm/pull/158). 

## Serving TensorFlow ResNet Model on ABM
In this installation you'll see how to create an end-to-end TensorFlow ML
serving ResNet installation on ABM using Google Compute Engine. Once the setup is completed, you'll be able to send image classification requests using GRPC client to ABM ML Serving
cluster.

### Requirements
 * Google Cloud Platform access and install `gcloud` SDK
 * Service Account JSON
 * Terraform, Git, Container Image

### ResNet SavedModel Image on GCR
Let's create a local directory and download the Deep residual network (ResNet)
model.

```
rm -rf /tmp/resnet
mkdir /tmp/resnet
curl -s http://download.tensorflow.org/models/official/20181001_resnet/savedmodels/resnet_v2_fp32_savedmodel_NHWC_jpg.tar.gz | tar --strip-components=2 -C /tmp/resnet -xvz
```

Verify the SavedModel

```
ls /tmp/resnet/*
saved_model.pb variables
```

Now we will commit the ResNet serving docker image:

```
docker run -d --name serving_base tensorflow/serving
docker cp /tmp/resnet serving_base:/models/resnet
docker commit --change "ResNet model" serving_base $USER/resnet_serving
docker kill serving_base
docker rm serving_base
```

Copy the local docker image to gcr.io

```
export GCR_IMAGE_PATH="gcr.io/$GCP_PROJECT/abm_serving/resnet"
docker tar $USER/resnet_serving $GCR_IMAGE_PATH
docker push $GCR_IMAGEPATH
```


### ABM GCE Cluster using Terraform
Create GCE demo host and perform few steps to setup the host:

```
export SERVICE_ACCOUNT_FILE=<FILE_LOCATION>

export DEMO_HOST="abm-demo-host-live"
gcloud compute instances create $DEMO_HOST --zone=us-central1-a
gcloud compute scp $SERVICE_ACCOUNT_FILE $USER@$DEMO_HOST:
```


Perform ssh login into the demo machine and follow steps below:

```
gcloud compute ssh $DEMO_HOST --zone=us-central1-a

# Activate Service Account
gcloud auth activate-service-account --key-file=$SERVICE_ACCOUNT_FILE

# Install Git
sudo apt-get install git

# Install Terraform
# v0.14.10
export TERRAFORM_VERSION="0.14.10"
```

List current Anthos/GKE clusters using hub membership. You can list existing
clusters and compare it with newly created clusters.

```
# List Anthos BM clusters
gcloud container hub memberships list
```

Install Terraform, and make few minor changes to configuration files:

```
# Remove any previous versions. You can skip if this is a new instance
sudo apt remove terraform

sudo apt-get install software-properties-common

curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -
sudo apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"
sudo apt-get update && sudo apt-get install terraform=$TERRAFORM_VERSION

terraform -version
```

Let's setup some ABM infrastructure on GCE using Terraform

```
# Git clone ABM Terraform setup
git clone https://github.com/GoogleCloudPlatform/anthos-samples.git
cd anthos-samples
git checkout abm-gcp-tf-demo
cd anthos-bm-gcp-terraform

# Make changes to cluster names and few edits
cp terraform.tfvars.sample terraform.tfvars
```

Make edits to the `variables.tf` and `terraform.tfvars` and also make sure the
`abm_cluster_id` is modified to a unique name

```
# Change abm_cluster_id and service account name in variables.tf
export CLUSTER_ID=`echo "abm-tensorflow-"$(date +"%m%d%H%M")`
echo $CLUSTER_ID
```

Create GCE resources using Terraform and verify

```
# Terraform init and apply
terraform init && terraform plan
terraform apply

# Verify resources using gcloud
gcloud compute instancs list

# Let's create cluster using bmctl and perform pre-flight checks and verify
export KUBECONFIG=$HOME/bmctl-workspace/$CLUSTER_ID/$CLUSTER_ID-kubeconfig

# List ABM clusters
gcloud container hub memberships list

# Listing the details of live-cluster
gcloud container hub memberships describe $LIVE_CLUSTER_NAME
```

Verify k8s cluster details and check few outputs
```
kubectl get nodes
kubectl get deployments
kubectl get pods
```

### TensorFlow ResNet model service on ABM Cluster

```
git clone https://github.com/GoogleCloudPlatform/anthos-ai
cd anthos-ai

kubectl create -f serving/resnet_k8s.yaml

# Let's view deployments and pods
kubectl get deployments
kubectl get pods

kubectl get services
kubectl describe service resnet-abm-service

# Let's send prediction request to ResNet service on ABM
git clone https://github.com/puneith/serving.git
sudo tools/run_in_docker.sh python tensorflow_serving/example/resnet_client_grpc.py $IMAGE_URL --server=10.200.0.51:8500
```

Return to the demo host and then destroy the demo host

```
# Destroy resources and demo host
terraform destroy

gcloud compute instances delete $DEMO_HOST
```
