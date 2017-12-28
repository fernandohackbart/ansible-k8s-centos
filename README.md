# Installation of a Kubernetes cluster over Centos7 

### Create a Kubernetes master guest
```
virsh destroy k8s-master1;virsh undefine k8s-master1;rm -rf /opt/dcos/guests/k8s-master1/
```
 
```
mkdir -p /opt/dcos/guests/k8s-master1

virt-install \
 -n k8s-master1 \
 --description="DCOS Kubernetes master 1 machine" \
 --os-type=Linux \
 --os-variant=generic \
 --ram=1500 \
 --vcpus=1 \
 --graphics=vnc \
 --noautoconsole \
 --disk path=/opt/dcos/guests/k8s-master1/k8s-master1.img,bus=virtio,size=7 \
 --pxe \
 --network=bridge:dcos-br0,model=virtio,mac=52:54:00:e2:87:60
 ``` 

### Create a Kubernetes node guest
```
virsh destroy k8s-node1;virsh undefine k8s-node1;rm -rf /opt/dcos/guests/k8s-node1/
```
 
```
mkdir -p /opt/dcos/guests/k8s-node1

virt-install \
 -n k8s-node1 \
 --description="DCOS Kubernetes node 1 machine" \
 --os-type=Linux \
 --os-variant=generic \
 --ram=6000 \
 --vcpus=1 \
 --graphics=vnc \
 --noautoconsole \
 --disk path=/opt/dcos/guests/k8s-node1/k8s-node1.img,bus=virtio,size=7 \
 --pxe \
 --network=bridge:dcos-br0,model=virtio,mac=52:54:00:e2:87:61
 ``` 




## Based on the Kubernetes documentation

https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/

https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init/#config-file

https://kubernetes.io/docs/admin/authentication/

## To access the dashboard

https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/#deploying-the-dashboard-ui

https://github.com/kubernetes/dashboard/wiki/Accessing-Dashboard---1.7.X-and-above

```
kubectl proxy
```

http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy


```
kubectl get pods --all-namespaces
kubectl drain k8s-node1.prototype.local --delete-local-data --force --ignore-daemonsets
kubectl delete node k8s-node1.prototype.local
kubeadm reset
kubectl delete deployment kubernetes-dashboard -n kube-system
```

