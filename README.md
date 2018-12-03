# Kubernetes local storage performance testing with Cassandra

This guide walks through the steps to test local storage performance of a 3-node Cassandra cluster running on Kubernetes on Oracle Cloud Infrastructure (OCI).

The Dense IO virtual and bare metal instances on OCI provides local storage powered by NVMe-based SSDs. You can find more info about the VM shapes available on OCI [here](https://cloud.oracle.com/compute/virtual-machine/features).

**NOTE:** Local Persistent Volumes for Kubernetes is beta. Check [this link](https://kubernetes.io/blog/2018/04/13/local-persistent-volumes-beta/) for more information.

I used Kubernetes 1.11.1 running on Oracle Linux 7.5 when writing the instructions. 


## Step 1: Creating a cluster with Oracle Cloud Infrastructure Container Engine for Kubernetes (OKE)
Follow the steps in [this link](https://www.oracle.com/webfolder/technetwork/tutorials/obe/oci/oke-full/index.html) to create a managed Kubernetes cluster running in OCI.

## Step 2: Creating a RAID array
After the cluster is created, we will SSH into the worker nodes and prepare the NVMe drives for Kubernetes. We will basically create a RAID0 array, format it as XFS and then mount it.

You can check [this link](https://docs.cloud.oracle.com/iaas/Content/GSG/Tasks/launchinginstance.htm#three) to learn how to get the public IPs of the worker nodes.

**NOTE:** The username for the worker nodes is **opc**.

You can find more info about protecting data on NVMe devices [here](https://docs.cloud.oracle.com/iaas/Content/Compute/References/nvmedeviceinformation.htm).

NOTE: The number of available NVMe devices changes based on the VM/Bare Metal instance shape. Make sure to change the command when creating the array accordingly. More info about shapes [here](https://cloud.oracle.com/compute/virtual-machine/features).

```
sudo yum install mdadm -y
```

```
sudo mdadm --create /dev/md0 --raid-devices=8 --level=0 /dev/nvme0n1 /dev/nvme1n1 /dev/nvme2n1 /dev/nvme3n1 /dev/nvme4n1 /dev/nvme5n1 /dev/nvme6n1 /dev/nvme7n1
```

```
sudo mdadm --detail --scan | sudo tee -a /etc/mdadm.conf >> /dev/null
```

```
sudo mkfs.xfs /dev/md0
```

```
sudo mkdir -p /mnt/disks/md0
```

```
sudo chmod -R 777 /mnt
```

```
sudo mount /dev/md0 /mnt/disks/md0
```

```
echo "/dev/md0   /mnt/disks/md0       xfs    defaults,nofail   0   2" > /tmp/mnt_entry
```

```
sudo sh -c 'cat /tmp/mnt_entry  >> /etc/fstab'
```

## Step 3: Creating a Storage Class

Commands below are taken from the [Local Persistent Storage User Guide](https://github.com/kubernetes-incubator/external-storage/tree/master/local-volume) in Kubernetes repo. So you need to clone the repo if you would want to follow these steps.


```
kubectl create -f provisioner/deployment/kubernetes/example/default_example_storageclass.yaml
```
## Step 4: Creating local persistent volumes

Again, the following commands assume that you have already cloned the [Local Persistent Storage User Guide](https://github.com/kubernetes-incubator/external-storage/tree/master/local-volume) repo.

**NOTE:** Dynamic provisioning is not supported in beta. All local PersistentVolumes must be statically created.


```
helm template ./helm/provisioner > ./provisioner/deployment/kubernetes/provisioner_generated.yaml
```

```
kubectl create -f ./provisioner/deployment/kubernetes/provisioner_generated.yaml
```


## Step 5: Deploying Cassandra with the Helm chart

This guide uses the chart in the incubator repository in [this link](https://github.com/helm/charts/tree/master/incubator/cassandra). 

I changed the values for settings like "max heap size, heap new size" in values.yaml according to the recommended Cassandra configuration parameters in values.yaml.

```
helm install --namespace "cassandra" -n "cassandra" incubator/cassandra
```

## Step 6: Running Cassandra performance tests

It takes about 6 minutes for all the Cassandra nodes up and running. You can check the status by running the following command:

```
kubectl exec -it --namespace cassandra cassandra-0 -- nodetool status
```

After the cluster is running, we can start testing the performance. We will use Cassandra's built-in testing tool called `cassandra-stress`.

We will be using the tests documented in Datastax's [page](https://docs.datastax.com/en/cassandra/3.0/cassandra/tools/toolsCStress.html) for `cassandra-stress`.

Firstly, we will initialize with 1 million writes:

```
kubectl exec -it --namespace cassandra cassandra-0 -- cassandra-stress write n=1000000 -rate threads=50
```

Then we will run a mixed test.

**NOTE:** I'm running the test on one of the Cassandra nodes for simplicity. Feel free to run the same test in the other nodes, too of you think running the test on one node would not saturate the cluster.

**NOTE:** I'm using a great tool called [Kubectx](https://github.com/ahmetb/kubectx) to manage Kubernetes contexts. I highly recommend using it if you're working on multiple Kubernetes clusters at the same time.

```
kubectl exec -it --namespace cassandra cassandra-0 -- cassandra-stress mixed ratio\(write=1,read=3\) n=100000 cl=ONE -pop dist=UNIFORM\(1..1000000\) -schema keyspace="keyspace1" -mode native cql3 -rate threads\>=16 threads\<=256 -log file=./$(kubectx)_mixed_autorate_50r50w_1M.log -graph file=$(kubectx).html title=$(kubectx)
```

```
kubectl cp cassandra/cassandra-0:$(kubectx)_mixed_autorate_50r50w_1M.log /Users/opastirm/Documents/test-results/$(kubectx)
```

```
kubectl cp cassandra/cassandra-0:$(kubectx).html /Users/opastirm/Documents/test-results/$(kubectx)
```
