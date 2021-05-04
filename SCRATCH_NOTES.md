Kubernetes (aka k8s)
==================

Installation:
- Each k8s cluster machine requires a unique MAC address and product_uuid (/sys/class/dmi/id/product_uuid); this should be the norm, except for some virtual machine setups.
- K8s (or only `kubeadm`?) only supports version skew of one minor version (e.g. 1.3 to 1.4 is one minor version)

### Required Open TCP Ports
**Control-plane node(s)**  
```
6443
2379-2380
10250
10251
10252
```

**Worker nodes**  
```
10250
30000-32767
```

**Installing Packages**  
K8s requires containerd, which is included automatically with current versions of Docker. This (containerd) is called the *container runtime*.  
```
apt install docker.io
```

The Kubernetes commands are not part of the package manager and can be installed by adding a third party
repository.  
* `kubelet` runs on each node (non-master) and is in charge of making sure any containers that are supposed to be running on that node are running.
* `kubeadm` creates a cluster by setting up all the services where and how they are needed
* `kube-proxy` runs on each node (non-master) to ensure rules are in place to allow each node to communicate with each of the other nodes
* `kubectl` The command line program that you use to issue k8s commands
```
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
```

Upgrading `kubeadm` requires specific following of upgrade documentation and should only be done when following the documentation. Mark the packages to NOT auto-update by running.
```
sudo apt-mark hold kubelet kubeadm kubectl
```

Must disabled swap for kubelet to run properly (why?); `swapoff-a` and then disable swap from `/etc/fstab`  

Either set all node in each `/etc/hosts` file or setup CoreDNS to provide DNS for cluster.  


## Install Network


## Create Cluster
```
kubeadm init ...
```
This will output a bootstrap token which you'll need to join nodes to the cluster (tokens last 24 hours).
If you need to see the current token again, run the following on the master node:  
```
kubeadm token list
```

If the token expires and you need a new token, run the following on the master node:  
```
kubeadm token create
```

And then, to join up worker nodes, run the following on the worker nodes themselves:  
```
kubeadm join --token <token> <control-plane-host>:<control-plane-port> --discovery-token-ca-cert-hash sha256:<hash>
```

The discovery token `<hash>` can be retrieveed by running the following on the master node:  
```
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | \
  openssl rsa -pubin -outform der 2>/dev/null | \
  openssl dgst -sha256 -hex | sed 's/^.* //'
```


### Deployments


### Replica Sets


### Volumes

Volumes are temporary on a pod.
PersistantVolumes are peristant even across pod restarts.

PersistantVolumes can be based off many different techs: NFS, Gluster, Ceph, AWS, Azure, etc
PersistantVolume is a resource that K8s can use, but K8s does NOT offer to manage storage for you. You have to setup the storage yourself.
With that, PVs are outside of any namespace and can even be shared between clusters if you want to.

Pods must specify a PersistantVolumeClaim, and then k8s will go find an appropriate PV to use.
 - Once found, PV is mounted into Pod
 - And then container in Pod may mound the PC as well

Special volumes managed by k8s (must be created before use):
 - ConfigMap - for importing a config into your pod
 - secret - for getting security file (e.g. ssl/ssh key) into your pod

### Other info
`etd` key-value store used master node(s) to keep info  
`kube-scheduler` decides where/which nodes pods get sent to  
"Control page node" is where control page components run, like `etcd` and the API server (what `kubectl` communicates with)  


bootstrap tokens live 24 hours, have `public.secretsecretseceret` format

`kubectl get nodes`
`kubectl get pods`

`kubectl` taints the control nodes to prevent it from running pods by default (can be configured otherwise though)
