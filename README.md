# Kubernetes local storage performance testing with Cassandra

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

```
kubectl create -f provisioner/deployment/kubernetes/example/default_example_storageclass.yaml
```

```
helm template ./helm/provisioner > ./provisioner/deployment/kubernetes/provisioner_generated.yaml
```

```
kubectl create -f ./provisioner/deployment/kubernetes/provisioner_generated.yaml
```

```
helm install --namespace "cassandra" -n "cassandra" incubator/cassandra
```

```
kubectl exec -it --namespace cassandra cassandra-0 -- nodetool status
```

```
kubectl exec -it --namespace cassandra cassandra-0 -- cassandra-stress write n=1000000 -rate threads=50
```

```
kubectl exec -it --namespace cassandra cassandra-0 -- cassandra-stress mixed ratio\(write=1,read=3\) n=100000 cl=ONE -pop dist=UNIFORM\(1..1000000\) -schema keyspace="keyspace1" -mode native cql3 -rate threads\>=16 threads\<=256 -log file=./$(kubectx)mixed_autorate_50r50w_1M.log -graph file=$(kubectx).html title=$(kubectx)
```

```
kubectl cp cassandra/cassandra-0:$(kubectx)mixed_autorate_50r50w_1M.log /Users/opastirm/Documents/test-results/$(kubectx)
```

```
kubectl cp cassandra/cassandra-0:$(kubectx).html /Users/opastirm/Documents/test-results/$(kubectx)
```
