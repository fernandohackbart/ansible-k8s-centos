# Install keepalovede cluster manager


```bash
kubectl get nodes |grep -v k8s-master-0|grep Ready|awk '{print $1}' |while read line
do 
  kubectl label node $line node-role.kubernetes.io/node=worker --overwrite=true
done
```

```bash
ansible-playbook -i hosts k8s-keepalived-configure.yml
```

```bash
ansible-playbook -i conf/hosts-prod-flannel k8s-keepalived-clean-local.yml
ansible-playbook -i conf/hosts-prod-flannel k8s-keepalived-configure-local.yml
```
```bash
echo "apiVersion: v1
kind: Service
metadata:
  labels:
    app: instalador
  name: ops-instalador-dr
  annotations:
    k8s.co/keepalived-forward-method: DR
  namespace: ops
spec:
  ports:
  - name: http
    port: 8080
    protocol: TCP
    targetPort: http
  selector:
    app: instalador
    release: ops-instalador
  type: LoadBalancer" |kubectl create -f -
```
