

# Installing GlusterFS and Heketi on Kubernetes



## Dependencies


## Configuring

The nodes should be already configured and have a /dev/vdb disk available for Gluster 

```
ansible-playbook -i hosts k8s-gluster-configure.yml
```

## Configuring the cluster
This should be executed in the Kubernetes master 

Check the cluster IP of the `heketi-cluster` service
```
export HEKETI_CLUSTER_IP=`kubectl get service |grep heketi-cluster |awk '{print $3}'`
export HEKETI_CLUSTER_PORT=`kubectl get service |grep heketi-cluster |awk '{print $5}' |awk '{split($0,a,"/"); print a[1]}'`
```

Check the topology
```
export HEKETI_CLI_SERVER=http://${HEKETI_CLUSTER_IP}:${HEKETI_CLUSTER_PORT}
export HEKETI_CLI_USER=admin
heketi-cli topology info --secret Welcome1
```


Load the topology again...
```
heketi-cli --secret Welcome1 topology load -j /opt/k8s-gluster/topology.json
```

## Testing

Create Kubernetes descriptors
```
cat > /tmp/test-storageclass-gluster.yml <<EOF
---  
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: test-storageclass-gluster
provisioner: kubernetes.io/glusterfs
parameters:
  #resturl: "http://heketi-cluster.default.svc.cluster.local:8080"
  resturl: "http://10.96.198.52:8080"
  restauthenabled: "true"
  restuser: "admin"
  restuserkey: "Welcome1"
  volumetype: "replicate:2"
allowVolumeExpansion: true
EOF
```


```
cat > /tmp/test-claim.yml <<EOF
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
 name: test-claim
 annotations:
   volume.beta.kubernetes.io/storage-class: test-storageclass-gluster
spec:
 accessModes:
  - ReadWriteOnce
 resources:
   requests:
     storage: 2Gi
EOF
```

```
kubectl create -f /tmp/gluster/test-storageclass-gluster.yml
kubectl create -f /tmp/gluster/test-claim.yml
```

```
kubectl delete -f /tmp/gluster/test-claim.yml 
kubectl delete -f /tmp/gluster/test-storageclass-gluster.yml
```

