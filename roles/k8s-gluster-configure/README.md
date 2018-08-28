

# Installing GlusterFS and Heketi on Kubernetes



## Dependencies

k8s-gluster-defaults

```
k8s_gluster_adm_dir: "/opt/k8s-gluster"
k8s_gluster_serviceaccount: "heketi-service-account"
k8s_gluster_namespace: "kube-system"
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
export HEKETI_CLUSTER_IP=`kubectl -n kube-system get service |grep heketi-cluster |awk '{print $3}'`
export HEKETI_CLUSTER_PORT=`kubectl -n kube-system get service |grep heketi-cluster |awk '{print $5}' |awk '{split($0,a,"/"); print a[1]}'`
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

To expand a volume
```
heketi-cli volume expand --volume=1bc4f5d7972dadef31c774376fe89cb1 -expand-size=8  --secret <password>
```

## Cleaning the resources

```
ansible-playbook -i hosts k8s-gluster-clean.yml
```

Probably after running this the devices used will have VGs left behind, reusing them may require manual intervention to remove the VGs and PVs
```
vgdisplay
vgremove <VG name>     ex: vg_8d93f425b9caec347a7c3cfd99686466
pvdisplay
pvremove /dev/vdb
```

## Testing (check if   volumetype: "replicate:2 or 3")

Create Kubernetes descriptors
```
export HEKETI_URL=http://`kubectl -n kube-system get services|grep heketi-cluster|awk '{print $3}'`:`kubectl -n kube-system get service |grep heketi-cluster |awk '{print $5}' |awk '{split($0,a,"/"); print a[1]}'`


cat > /tmp/test-storageclass-gluster.yml <<EOF
---  
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: test-storageclass-gluster
provisioner: kubernetes.io/glusterfs
parameters:
  resturl: "${HEKETI_URL}"
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
kubectl create -f /tmp/test-storageclass-gluster.yml
kubectl create -f /tmp/test-claim.yml
```

```
kubectl get sc
kubectl get pvc
kubectl get pv
```

```
kubectl delete -f /tmp/test-claim.yml 
kubectl delete -f /tmp/test-storageclass-gluster.yml
```

