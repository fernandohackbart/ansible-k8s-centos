

# Installing GlusterFS and Heketi on Kubernetes



## Dependencies

k8s-gluster-defaults

```
k8s_gluster_adm_dir: "/opt/k8s-gluster"
k8s_gluster_serviceaccount: "heketi-service-account"
k8s_gluster_namespace: "default"
k8s_heketi_secret: "Welcome1"
k8s_heketi_executor: "kubernetes"
k8s_heketi_node_port: "30625"
k8s_heketi_cluster_port: "8080"
k8s_heketi_service_account: "heketi-service-account"
k8s_heketi_nodes_names: ["k8s-node1.prototype.local","k8s-node2.prototype.local"]
k8s_heketi_nodes_ips: ["192.168.40.43","192.168.40.48"]
k8s_heketi_nodes_devices: ["/dev/vdb","/dev/vdb"]

# Until https://github.com/fernandohackbart/ansible-k8s-centos/issues/2 is fixed
k8s_heketi_nodes_name1: "k8s-node1.prototype.local"
k8s_heketi_nodes_name2: "k8s-node2.prototype.local"
k8s_heketi_nodes_ip1: "192.168.40.43"
k8s_heketi_nodes_ip2: "192.168.40.48"
k8s_heketi_nodes_device1: "/dev/vdb"
k8s_heketi_nodes_device2: "/dev/vdb"
```


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

## Cleaning the resources

This can be tricky to reconfigure as the devices can be already in use..., but to clean can use this
```
ansible-playbook -i hosts k8s-gluster-clean.yml
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

