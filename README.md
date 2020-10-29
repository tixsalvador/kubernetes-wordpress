# Wordpress on kubernetes cluster

Setting up wordpress on k8s running on vagrant centos/7 vm's.

### Requirements

- Create vm's using [Vagrantfile]
- Run ansible [playbook] to prepare the vm's.

### Steps to install kubernetes cluster using kubeadm

```sh
$ sudo to root
$ kubeadm init --pod-network-cidr 10.244.0.0/16 --apiserver-advertise-address 10.10.10.10
```

- 10.244.0.0/16 - default network of flannel
- 10.10.10.10 - master node ip

```sh
$ su <regular user>
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Deploy pod network to the cluster. I edit the network settings on my [flannel yaml] to use the 2nd nic due to the 1st nic is NATed iface of vagrant.

```sh
$ kubectl apply -f kube-flannel.yml
```

Add nodes to the cluster

```sh
#Run to get the join command
$ kubeadm token create --print-join-command
#Connect to each node of the cluster and run the join command
$ kubeadm join 10.10.10.10:6443 --token <blah>     --discovery-token-ca-cert-hash <blah>
#While on the master and nodes fix kublet to use the local ip.
edit the file /etc/sysconfig/kubelet
Add:
KUBELET_EXTRA_ARGS=--node-ip=<the.node.ip>
#Reboot the Vm's
```

Kubernetes cluster should be up by now

```sh
#Check nodes
$ kubectl get nodes -owide

NAME        STATUS   ROLES    INTERNAL-IP
master-k8   Ready    master   10.10.10.10
node-k81    Ready    <none>   10.10.10.11
node-k82    Ready    <none>   10.10.10.12
node-k83    Ready    <none>   10.10.10.13

#Check pods
$ kubectl get pods -owide

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
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0/aio/deploy/recommended.yaml
```

Make the dashboard available externally. (This is only for my testing environment)
Edit kubernetes-dashboard service

```sh
$ kubectl -n kubernetes-dashboard  edit service kubernetes-dashboard
# Change:
type: NodePort
```

Get token

```sh
$ kubectl -n kube-system describe $(kubectl -n kube-system get secret -n kube-system -o name | grep namespace) | grep token:
```

Connect to dashboard UI
external-ip:service port

#### Create secret file

```sh
db_password=echo -n “pass” | base64
wordpress_password=echo -n “pass” | base64
```

[wp_secret.yml]

```sh
apiVersion: v1
kind: Secret
metadata:
  name: wordpress-secret
  labels:
    app: wordpress
type: Opaque
data:
  db_root_password: $db_password
  db_wordpress_password: $wordpress_password
```

```sh
$ kubectl create -f wp_secret.yml
```

#### Create persistent volume and claim

[wp_volumes.yml]

```sh
apiVersion: v1
kind: PersistentVolume
metadata:
  name: etcd-pv0
  labels:
    pv-name: etcd-pv0
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Recycle
  nfs:
    path: /mnt/nfs/data/etcd0
    server: 10.10.10.10
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-etcd-0
  labels:
    app: wordpress
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  selector:
    matchLabels:
      pv-name: etcd-pv0
```

```sh
$ kubectl create -f https://raw.githubusercontent.com/tixsalvador/kubernetes-wordpress/main/wp_volumes.yml
```

#### Deploy etcd

Assign labels to nodes

```sh
$  kubectl label nodes node-k81 name=node-k81
$  kubectl label nodes node-k82 name=node-k82
$  kubectl label nodes node-k83 name=node-k83
```

Add pod anti affinity rule on etcd yaml so scheduler does not co-locate replicas on a single node.

```sh
spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - etcd
            topologyKey: "kubernetes.io/hostname"
      containers:
```

Deploy etcd pods and services [wp_etcd.yml]

```sh
$ kubectl create -f https://raw.githubusercontent.com/tixsalvador/kubernetes-wordpress/main/wp_etcd.yml
```

Add service port temporarily only to check if etcd is working.

```sh
$ vim wp_etcd.yml
Add:
apiVersion: v1
kind: Service
metadata:
   name: etcd-testport
   labels:
     app: etcd-testport
spec:
  type: NodePort
  ports:
  - port: 2379
    name: etcd-testport
    targetPort: 2379
    protocol: TCP
  selector:
    app: etcd
```

Check etcd if it's working

```sh
$ kubectl get pods -o wide | awk '{print $1,$3,$7}'
NAME STATUS NODE
etcd-0 Running node-k81
etcd-1 Running node-k83
etcd-2 Running node-k82
```

```sh
 $ curl -L -X PUT http://10.10.10.10:32488/v2/keys/message -d value="tixfirstetcd"
{"action":"set","node":{"key":"/message","value":"tixfirstetcd","modifiedIndex":29,"createdIndex":29},"prevNode":{"key":"/message","value":"tixfirstetcd","modifiedIndex":28,"createdIndex":28}}
$ curl -L http://10.10.10.10:32488/v2/keys/message
{"action":"get","node":{"key":"/message","value":"tixfirstetcd","modifiedIndex":29,"createdIndex":29}}
```

#### Mysql cluster

Deploy 3 mysql replicas [wp_mysql.yml]

```sh
$ kubectl create -f https://raw.githubusercontent.com/tixsalvador/kubernetes-wordpress/main/wp_mysql.yml
```

Check percona cluster if sync. (Values should be the same below)

```sh
mysql> show global status  where Variable_name='wsrep_cluster_status' or Variable_name like 'wsrep_local_state';
+----------------------+---------+
| Variable_name        | Value   |
+----------------------+---------+
| wsrep_local_state    | 4       |
| wsrep_cluster_status | Primary |
+----------------------+---------+

mysql> show variables like 'read_only';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| read_only     | OFF   |
+---------------+-------+
```

Check etcd if the nodes are added in the service directory. Enable etcd-testport to connect to etcd nodeport.

```sh
$ curl http://<ETCD IP:NODEPORT>/v2/keys/pxc-cluster/<CLUSTER_NAME>/?recursive=true | jq
{
  "action": "get",
  "node": {
    "key": "/pxc-cluster/percona",
    "dir": true,
    "nodes": [
      {
        "key": "/pxc-cluster/percona/10.244.2.83",
        "dir": true,
        "expiration": "2020-10-18T20:58:29.432858508Z",
        "ttl": 26,
        "nodes": [
          {
            "key": "/pxc-cluster/percona/10.244.2.83/ipaddr",
            "value": "10.244.2.83",
            "expiration": "2020-10-18T20:58:29.362794483Z",
            "ttl": 26,
            "modifiedIndex": 45297,
            "createdIndex": 45297
          },
          {
            "key": "/pxc-cluster/percona/10.244.2.83/hostname",
            "value": "mysql-2",
            "expiration": "2020-10-18T20:58:29.391964085Z",
            "ttl": 26,
            "modifiedIndex": 45298,
            "createdIndex": 45298
          }
        ],
        "modifiedIndex": 40653,
        "createdIndex": 40653
      },
      {
        "key": "/pxc-cluster/percona/10.244.3.63",
        "dir": true,
        "expiration": "2020-10-18T20:58:33.02749108Z",
        "ttl": 30,
        "nodes": [
          {
            "key": "/pxc-cluster/percona/10.244.3.63/ipaddr",
            "value": "10.244.3.63",
            "expiration": "2020-10-18T20:58:32.963542336Z",
            "ttl": 30,
            "modifiedIndex": 45303,
            "createdIndex": 45303
          },
          {
            "key": "/pxc-cluster/percona/10.244.3.63/hostname",
            "value": "mysql-1",
            "expiration": "2020-10-18T20:58:32.999217831Z",
            "ttl": 30,
            "modifiedIndex": 45304,
            "createdIndex": 45304
          }
        ],
        ],
        "modifiedIndex": 40683,
        ],
        "modifiedIndex": 40683,
        ],
        "modifiedIndex": 40683,
        "createdIndex": 40683
      },
      {
        "key": "/pxc-cluster/percona/10.244.1.86",
        "dir": true,
        "expiration": "2020-10-18T20:58:31.643178982Z",
        "ttl": 28,
        "nodes": [
          {
            "key": "/pxc-cluster/percona/10.244.1.86/ipaddr",
            "value": "10.244.1.86",
            "expiration": "2020-10-18T20:58:31.575521209Z",
            "ttl": 28,
            "modifiedIndex": 45300,
            "createdIndex": 45300
          },
          {
            "key": "/pxc-cluster/percona/10.244.1.86/hostname",
            "value": "mysql-0",
            "expiration": "2020-10-18T20:58:31.60966732Z",
            "ttl": 28,
            "modifiedIndex": 45301,
            "createdIndex": 45301
          }
        ],
        "modifiedIndex": 40635,
        "createdIndex": 40635
      }
    ],
    "modifiedIndex": 37,
    "createdIndex": 37
  }
}
```

#### Wordpress

Deploy wordpress in replicaset [wp_wordpress.yml]

```sh
$ kubectl create -f https://raw.githubusercontent.com/tixsalvador/kubernetes-wordpress/main/wp_wordpress.yml
```

Add HPA (Horizontal Pod Autoscaler) for wordpress

Requirements:

- Metrics server [metrics-server]
- HTTP/HTTPS stress tester [siege] - testing utility

Install metrics-server

Download components.yaml and edit start settings because of self-singed certs and vagrant vm's.

```sh
$ wget https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.3.7/components.yaml
```

Edit components.yaml

```sh
# Look for the container config arguents and add --kubelet-preferred-address-types and --kubelet-insecure-tls
$ vi components.yaml
 containers:
      - name: metrics-server
        image: k8s.gcr.io/metrics-server/metrics-server:v0.3.7
        imagePullPolicy: IfNotPresent
        args:
          - --cert-dir=/tmp
          - --secure-port=4443
          - --kubelet-preferred-address-types=InternalIP
          - --kubelet-insecure-tls
```

Start the metric server

```sh
$  kubectl apply -f components.yaml

$  kubectl top nodes
NAME        CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
master-k8   113m         5%     1141Mi          65%
node-k81    72m          7%     687Mi           77%
node-k82    66m          6%     648Mi           72%
node-k83    65m          6%     642Mi           72%

$  kubectl top pods
NAME              CPU(cores)   MEMORY(bytes)
etcd-0            15m          40Mi
etcd-1            13m          26Mi
etcd-2            12m          28Mi
mysql-0           12m          224Mi
mysql-1           15m          224Mi
mysql-2           15m          243Mi
wordpress-7nhqq   1m           41Mi
```

[vagrantfile]: https://github.com/tixsalvador/vagrant_docker/blob/master/Vagrantfile.k8
[playbook]: https://github.com/tixsalvador/ansible_vagrant
[flannel yaml]: https://github.com/tixsalvador/ansible_vagrant/blob/master/files/kube-flannel.yml
[wp_secret.yml]: https://github.com/tixsalvador/kubernetes-wordpress/blob/main/wp_secret.yml
[wp_volumes.yml]: https://github.com/tixsalvador/kubernetes-wordpress/blob/main/wp_volumes.yml
[wp_etcd.yml]: https://github.com/tixsalvador/kubernetes-wordpress/blob/main/wp_etcd.yml
[wp_mysql.yml]: https://raw.githubusercontent.com/tixsalvador/kubernetes-wordpress/main/wp_mysql.yml
[wp_wordpress.yml]: https://github.com/tixsalvador/kubernetes-wordpress/blob/main/wp_wordpress.yml
[metrics-server]: https://github.com/kubernetes-sigs/metrics-server
[siege]: https://www.joedog.org/siege-home/
