# TensorFlow MNIST Training using Kubeflow TFJob
TensorFlow training can be done in a single node or distributed setup. Even though we can run TensorFlow directly on Kubernetes (as shown in the TF Serving example), the TFJob abstraction makes it easy to define TensorFlow deployments. The standard way of performing multi-node training in TensorFlow is using TF_CONFIG environment variable. This environment variable is a JSON string which provides cluster and task information, and is set for each binary running on the cluster. The nodes in the cluster can be worker and ps nodes. In multi-worker training, there is usually one worker that takes on a little more responsibility like saving checkpoint and writing summary file for TensorBoard in addition to what a regular worker does. Such worker is referred to as the chief worker, and it is customary that the worker with index 0 is appointed as the chief worker. Each node in the cluster also has one of the roles worker, ps, or chief.

The setting of TF_CONFIG environment variable can be a manual process if done outside Kubeflow, and this is the part which TFJob Controller automatically manages.
 
Follow the steps below to create a Kubeflow based ABM TensorFlow Training cluster.

1. Install the Juju client

```
snap install juju --classic
```

* Get storage details

```
kubectl get pvc
```

Anthos clusters on bare metal cluster uses the local volume provisioner (LVP) to manage local persistent volumes. There are three types of storage classes for local PVs in an Anthos clusters on bare metal cluster. 
* LVP share
* LVP node mounts
* Anthos system

https://cloud.google.com/anthos/clusters/docs/bare-metal/1.6/installing/storage

*Discuss node mounts*

2. Connect Juju to k8s cluster
juju add-k8s tfjobk8s --cluster-name=abm-tensorflow-tfjob-05132256 --storage=standard

3. Create a controller
juju bootstrap tfjobk8s

4.
tfadmin@abm-tensorflow-tfjob-05132256-ws0-001:~$ juju controllers
Use --refresh option with this command to see the latest information.

Controller  Model       User   Access     Cloud/Region  Models  Nodes  HA  Version
tfjobk8s*   controller  admin  superuser  tfjobk8s           1      1   -  2.9.0  

5. juju add-model kubeflow

6. juju deploy kubeflow

7. watch -c juju status --color
When everything is green

8. git clone https://github.com/kubeflow/tf-operator
cd tf-operator/examples/v1/mnist_with_summaries
# Deploy the event volume
kubectl apply -f tfevent-volume
# Submit the TFJob
kubectl apply -f tf_job_mnist.yaml
