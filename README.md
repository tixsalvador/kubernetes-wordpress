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
```
 Deploy pod network to the cluster. I edit the network settings on my [flannel yaml]  to use the 2nd nic due to the 1st nic is the NATed iface of vagrant. 
 ```sh
 kubectl apply -f kube-flannel.yml 
 ```
 
 Add nodes to the cluster
 ```sh
 #Run to get the join command
 kubeadm token create --print-join-command 
 #Connect to each node of the cluster and run the join command
 kubeadm join 10.10.10.10:6443 --token <blah>     --discovery-token-ca-cert-hash <blah>
 ```
 
[Vagrantfile]: <https://github.com/tixsalvador/vagrant_docker/blob/master/Vagrantfile.k8>
[playbook]: <https://github.com/tixsalvador/ansible_vagrant>
[flannel yaml]: <https://github.com/tixsalvador/ansible_vagrant/blob/master/files/kube-flannel.yml>
