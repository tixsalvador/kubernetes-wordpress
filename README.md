# Wordpress on kubernetes cluster 

Setting up wordpress on k8s  running on vagrant centos/7 vm's. 
### Requirements
 - Create vm's using [Vagrantfile]
 - Run ansible [playbook] to prepare the vm's.

### Steps to install kubernetes cluster using kubeadm
```sh
sudo to root
kubeadm init --pod-network-cidr 10.244.0.0/16 --apiserver-advertise-address 10.10.10.10
```
 - 10.244.0.0/16 - default network of flannel 
 - 10.10.10.10 - master node ip

```sh
su <regular user>
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
 Deploy pod network to the cluster. I edit the network settings on my [flannel yaml]  to use the 2nd nic due to the 1st nic is NATed iface of vagrant. 
 ```sh
 kubectl apply -f kube-flannel.yml 
 ```
 
 Add nodes to the cluster
 ```sh
 #Run to get the join command
 kubeadm token create --print-join-command 
 #Connect to each node of the cluster and run the join command
 kubeadm join 10.10.10.10:6443 --token <blah>     --discovery-token-ca-cert-hash <blah>
 #While on the master and nodes fix kublet to use the local ip.
 edit the file /etc/sysconfig/kubelet
 Add:
 KUBELET_EXTRA_ARGS=--node-ip=<the.node.ip> 
 #Reboot the Vm's
 ```
 Kubernetes cluster should be up by now
 ```sh
 #Check nodes
 kubectl get nodes -owide
 
 NAME        STATUS   ROLES    INTERNAL-IP   
master-k8   Ready    master   10.10.10.10   
node-k81    Ready    <none>   10.10.10.11   
node-k82    Ready    <none>   10.10.10.12   
node-k83    Ready    <none>   10.10.10.13  

#Check pods
kubectl get pods -owide

NAMESPACE     NAME                                  STATUS      IP            NODE        
kube-system   coredns-f9fd979d6-gx4qk               Running     10.244.0.5    master-k8   
kube-system   coredns-f9fd979d6-wbtll               Running     10.244.0.4    master-k8   
kube-system   etcd-master-k8                        Running     10.10.10.10   master-k8   
kube-system   kube-apiserver-master-k8              Running     10.10.10.10   master-k8   
kube-system   kube-controller-manager-master-k8     Running     10.10.10.10   master-k8   
kube-system   kube-flannel-ds-9jc4n                 Running     10.10.10.13   node-k83    
kube-system   kube-flannel-ds-bmhhr                 Running     10.10.10.10   master-k8   
kube-system   kube-flannel-ds-q8gz6                 Running     10.10.10.12   node-k82    
kube-system   kube-flannel-ds-qkspf                 Running     10.10.10.11   node-k81    
kube-system   kube-proxy-5vnkf                      Running     10.10.10.12   node-k82    
kube-system   kube-proxy-j7z5x                      Running     10.10.10.13   node-k83    
kube-system   kube-proxy-t8dbp                      Running     10.10.10.10   master-k8   
kube-system   kube-proxy-zfbgg                      Running     10.10.10.11   node-k81    
kube-system   kube-scheduler-master-k8              Running     10.10.10.10   master-k8 
```

#### Deploy Dashboard UI on external interface.
```sh
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0/aio/deploy/recommended.yaml
```
Make the dashboard available externally. (This is only for my testing environment)
Edit kubernetes-dashboard service
```sh
kubectl -n kubernetes-dashboard  edit service kubernetes-dashboard
# Change:
type: NodePort   
```
Get token
```sh
kubectl -n kube-system describe $(kubectl -n kube-system get secret -n kube-system -o name | grep namespace) | grep token:
```
Connect to dashboard UI
```sh
kubectl proxy
```
http://external-ip:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/


[Vagrantfile]: <https://github.com/tixsalvador/vagrant_docker/blob/master/Vagrantfile.k8>
[playbook]: <https://github.com/tixsalvador/ansible_vagrant>
[flannel yaml]: <https://github.com/tixsalvador/ansible_vagrant/blob/master/files/kube-flannel.yml>
