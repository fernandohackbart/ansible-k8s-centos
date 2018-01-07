# Installation of a Kubernetes cluster over Centos7 

### Create a Kubernetes master guest
```
virsh destroy k8s-master1;virsh undefine k8s-master1;rm -rf /opt/k8s/guests/k8s-master1/
virsh destroy k8s-node1;virsh undefine k8s-node1;rm -rf /opt/k8s/guests/k8s-node1/
virsh destroy k8s-node2;virsh undefine k8s-node2;rm -rf /opt/k8s/guests/k8s-node2/
```
 
```
mkdir -p /opt/k8s/guests/k8s-master1

virt-install \
 -n k8s-master1 \
 --description="DCOS Kubernetes master 1 machine" \
 --os-type=Linux \
 --os-variant=generic \
 --ram=1500 \
 --vcpus=1 \
 --graphics=vnc \
 --noautoconsole \
 --disk path=/opt/k8s/guests/k8s-master1/k8s-master1.img,bus=virtio,size=7 \
 --pxe \
 --boot=hd,network \
 --network=bridge:dcos-br0,model=virtio,mac=52:54:00:e2:87:60
``` 

### Create a Kubernetes node guest
```
mkdir -p /opt/k8s/guests/k8s-node1

virt-install \
 -n k8s-node1 \
 --description="DCOS Kubernetes node 1 machine" \
 --os-type=Linux \
 --os-variant=generic \
 --ram=6000 \
 --vcpus=4 \
 --graphics=vnc \
 --noautoconsole \
 --disk path=/opt/k8s/guests/k8s-node1/k8s-node1.img,bus=virtio,size=7 \
 --disk path=/opt/k8s/guests/k8s-node1/k8s-node1-data.img,bus=virtio,size=20 \
 --pxe \
 --boot=hd,network \
 --network=bridge:dcos-br0,model=virtio,mac=52:54:00:e2:87:61
```

```
mkdir -p /opt/k8s/guests/k8s-node2

virt-install \
 -n k8s-node2 \
 --description="DCOS Kubernetes node 2 machine" \
 --os-type=Linux \
 --os-variant=generic \
 --ram=6000 \
 --vcpus=4 \
 --graphics=vnc \
 --noautoconsole \
 --disk path=/opt/k8s/guests/k8s-node2/k8s-node2.img,bus=virtio,size=7 \
 --disk path=/opt/k8s/guests/k8s-node2/k8s-node2-data.img,bus=virtio,size=20 \
 --pxe \
 --boot=hd,network \
 --network=bridge:dcos-br0,model=virtio,mac=52:54:00:e2:87:66
```

## After theinstallation of the Linux boxes the same are shutdown, should start them
```
virsh start k8s-master1
virsh start k8s-node1
virsh start k8s-node2
```

## If this is a reinstallation then have to clean the old SSH fingerprints
```
ssh-keygen -R 192.168.40.42 -f /home/fernando.hackbart/.ssh/known_hosts
ssh-keygen -R 192.168.40.43 -f /home/fernando.hackbart/.ssh/known_hosts
ssh-keygen -R 192.168.40.48 -f /home/fernando.hackbart/.ssh/known_hosts
```

## If this is a reinstallation then have to clean the old SSH fingerprints
```
ssh root@192.168.40.42 date
ssh root@192.168.40.43 date
ssh root@192.168.40.48 date
```

## Run the Ansible deployment playbook
```
git clone https://github.com/fernandohackbart/ansible-k8s-centos.git
```

## Run the Ansible deployment playbook
```
cd ansible-k8s-centos
ansible-playbook -i hosts k8s.yml
```

## To configure GlusterFS and Heketi on Kubernetes
```
cd ansible-k8s-centos
ansible-playbook -i hosts k8s-gluster-configure.yml
```

## Configure kubectl in the desktop
```
scp root@192.168.40.42:/etc/kubernetes/admin.conf  ~/.kube/config
```

## To access the dashboard
https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/#deploying-the-dashboard-ui

https://github.com/kubernetes/dashboard/wiki/Accessing-Dashboard---1.7.X-and-above


```
kubectl proxy
```

http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy

You can skip the authentication as the kubernetes-dashboard roles was created as per https://github.com/kubernetes/dashboard/wiki/Access-control...

So, no security in the dashboard for this installation :P

## Setting up Helm

https://github.com/kubernetes/helm

https://docs.helm.sh/using_helm/#quickstart-guide

As I am using OpenSuSE Tumbleweed there is a package called helm to by installed with zypper
```
zypper install helm
```

You should have a Kubernetes master running on 192.168.40.42 and kubectl configured
```
kubectl config current-context
kubectl cluster-info
```

Be sure that the following role mapping and grants are run  (this is run in the k8s-master playbook, but in case the helm commands fail with some unauthorized error messages)
```
kubectl create serviceaccount --namespace kube-system tiller
kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
kubectl create clusterrolebinding default-cluster-rule --clusterrole=cluster-admin --serviceaccount=default:default
kubectl create clusterrolebinding kube-system-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:default

```
So, now we can init
```
helm init --service-account tiller --upgrade
```

Assign service account tiller to the tiller POD
```
kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'
```

and check if there is a POD called `tiller-deploy`
```
kubectl get pods --namespace kube-system
```

Let's play with it, first update the repo 
```
helm repo update
```

And add the incubator helm repo
```
helm repo add incubator https://kubernetes-charts-incubator.storage.googleapis.com/
```

## Some commands:
```
kubectl get pods --all-namespaces
kubectl drain k8s-node1.prototype.local --delete-local-data --force --ignore-daemonsets
kubectl delete node k8s-node1.prototype.local
kubeadm reset
kubectl delete deployment kubernetes-dashboard -n kube-system
```

## Based on the Kubernetes documentation

https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/

https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init/#config-file

https://kubernetes.io/docs/admin/authentication/



