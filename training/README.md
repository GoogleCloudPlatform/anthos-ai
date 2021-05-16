# TensorFlow MNIST Training on ABM using Kubeflow TFJob
TensorFlow training can be done on single node or in multi-node setup. Even though we can run TensorFlow directly on Kubernetes (as shown in the TF Serving example), the TFJob abstraction makes it easy to define TensorFlow *training deployments*. [TFJob](https://www.kubeflow.org/docs/components/training/tftraining/) is a Kubernetes [custom resource](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) that you can use to run TensorFlow training jobs on Kubernetes. The Kubeflow implementation of TFJob is in [tf-operator](https://github.com/kubeflow/tf-operator). The standard way of performing multi-node training in TensorFlow is using [TF_CONFIG](https://www.tensorflow.org/guide/distributed_training#TF_CONFIG) environment variable. This environment variable is a JSON string which provides `cluster` and `task` information, and is set for each binary running on the cluster. The nodes in the cluster can be `worker` and `ps` nodes. In multi-worker training, there is usually one worker that takes on a little more responsibility like saving checkpoint and writing summary file for TensorBoard in addition to what a regular worker does. Such worker is referred to as the `chief` worker, and it is customary that the worker with index 0 is appointed as the chief worker. Each node in the cluster also has one of the roles `worker`, `ps`, or `chief`.

The setting of TF_CONFIG environment variable can be a manual process if done outside Kubeflow, and this is the part which [TFJob Controller](https://github.com/kubeflow/tf-operator/blob/master/tf_job_design_doc.md#controller) automatically manages. 

## Prerequisites
* [Anthos Bare Metal](https://cloud.google.com/anthos/clusters/docs/bare-metal/1.7) Cluster on GCE (using [Terraform](https://learn.hashicorp.com/tutorials/terraform/install-cli))
* ABM GPU Cluster

Checkout the [Github](https://github.com/GoogleCloudPlatform/anthos-samples/tree/master/anthos-bm-gcp-terraform) repo for more details on how to automatically setup ABM on Google Compute Engine and Terraform. 

## Kubeflow install on ABM
Anthos Bare Metal already provides access to k8s cluster and GPUs. The next steps will show how to install Kubeflow (TFJob) on ABM using [Juju](https://juju.is/) followed by MNIST training. Juju is an Open Source Charmed Operator Framework, composed of a Charmed Operator Lifecycle Manager and the Charmed Operator SDK. It simplifies how you configure, scale and operate todays’ complex software. Juju deploys everywhere: to public or private clouds. [Charmed Kubeflow](https://charmed-kubeflow.io/) is the full set of K8s operators to deliver applications and services that make up the latest version of Kubeflow. Perform the steps below to get started:

* Install the Juju client via [snap](https://snapcraft.io/docs/installing-snapd) using command:

```
snap install juju --classic
```

* Get storage details

```
kubectl get pvc
```

Anthos clusters on bare metal cluster uses the local volume provisioner (LVP) to manage local persistent volumes. There are three types of storage classes for [local PVs](https://cloud.google.com/anthos/clusters/docs/bare-metal/1.6/installing/storage) in an Anthos clusters on bare metal cluster:

1. LVP share
2. LVP node mounts
3. Anthos system

Let's connect juju to the ABM k8s cluster. Because we already have an ABM cluster, we will use Juju to deploy the Kubeflow CRDs (like TFJob) in order to run TensorFlow training. We'll specify the ABM cluster name which was provided during [Terraform](https://www.terraform.io/) automation. 

* Connect Juju to k8s cluster
```
export ABM_CLUSTER_NAME=<YOUR_ABM_CLUSTER_NAME>
juju add-k8s tfjobk8s --cluster-name=$ABM_CLUSTER_NAME --storage=standard
```

* Create a controller
```
juju bootstrap tfjobk8s
```

* Check controllers

```
Controller  Model       User   Access     Cloud/Region  Models  Nodes  HA  Version
tfjobk8s*   controller  admin  superuser  tfjobk8s           1      1   -  2.9.0  
```

* Add juju model and deploy Kubeflow 

```
juju add-model kubeflow
juju deploy kubeflow
```

* Verify progress of the install

```
watch -c juju status --color
```

## TensorFlow MNIST Training using TFJob
* Let's git clone the MNIST TensorFlow training from Kubeflow

```
git clone https://github.com/kubeflow/tf-operator
cd tf-operator/examples/v1/mnist_with_summaries

# Deploy the event volume from anthos-ai repo
kubectl apply -f anthos-ai/training/tfevent-volume

# Submit the TFJob
kubectl apply -f tf_job_mnist.yaml
```

You can check the running TensorFlow Job:

```
kubectl -n kubeflow get tfjob mnist -o yaml

```
You should see the output as shown below. Notice, how the code is running an image `gcr.io/kubeflow-ci/tf-mnist-with-summaries:1.0` and python TensorFlow code `/var/tf_mnist/mnist_with_summaries.py`

```
apiVersion: kubeflow.org/v1
kind: TFJob
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"kubeflow.org/v1","kind":"TFJob","metadata":{"annotations":{},"name":"mnist","namespace":"kubeflow"},"spec":{"cleanPodPolicy":"None","tfReplicaSpecs":{"Worker":{"replicas":1,"restartPolicy":"Never","template":{"spec":{"containers":[{"command":["python","/var/tf_mnist/mnist_with_summaries.py","--log_dir=/train/logs","--learning_rate=0.01","--batch_size=150"],"image":"gcr.io/kubeflow-ci/tf-mnist-with-summaries:1.0","name":"tensorflow","volumeMounts":[{"mountPath":"/train","name":"training"}]}],"volumes":[{"name":"training","persistentVolumeClaim":{"claimName":"tfevent-volume"}}]}}}}}}
  creationTimestamp: "2021-05-14T16:40:23Z"
  generation: 1
  name: mnist
  namespace: kubeflow
  resourceVersion: "669424"
  selfLink: /apis/kubeflow.org/v1/namespaces/kubeflow/tfjobs/mnist
  uid: 11a8d9e5-82ce-488f-a06f-929d31dfbb62
spec:
  cleanPodPolicy: None
  tfReplicaSpecs:
    Worker:
      replicas: 1
      restartPolicy: Never
      template:
        spec:
          containers:
          - command:
            - python
            - /var/tf_mnist/mnist_with_summaries.py
            - --log_dir=/train/logs
            - --learning_rate=0.01
            - --batch_size=150
            image: gcr.io/kubeflow-ci/tf-mnist-with-summaries:1.0
            name: tensorflow
            volumeMounts:
            - mountPath: /train
              name: training
          volumes:
          - name: training
            persistentVolumeClaim:
              claimName: tfevent-volume
status:
  conditions:
  - lastTransitionTime: "2021-05-14T16:40:23Z"
    lastUpdateTime: "2021-05-14T16:40:23Z"
    message: TFJob mnist is created.
    reason: TFJobCreated
    status: "True"
    type: Created
  - lastTransitionTime: "2021-05-14T16:40:25Z"
    lastUpdateTime: "2021-05-14T16:40:25Z"
    message: TFJob kubeflow/mnist is running.
    reason: TFJobRunning
    status: "True"
    type: Running
  replicaStatuses:
    Worker:
      active: 1

```
