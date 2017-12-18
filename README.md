# Installation of a Kubernetes cluster over Centos7 


## Based on the Kubernetes documentation


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

