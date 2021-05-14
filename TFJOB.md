# TensorFlow Training using TFJob
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
