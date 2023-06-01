# Kubernetes Documentation

[[_TOC_]]

## What is Kubernetes

## Terminology Reference

* control plane / worker

## How Kubernetes Works

* API driven

## Runtime

Kubernetes will run on a number of Container Runtimes which implement
the _Container Runtime Interface_ (CRI). The two most well known are:

* containerd
* CRI-O

The containerd CRI is actually already installed if you have Docker
installed, as Docker Engine also makes use of containerd.

## Setup & Install

Provision your nodes and ensure `/etc/hosts` is set up with all node
hostnames and IPs on each of the other nodes in the cluster.

### Example Setup
For the examples in this readme, we'll be using the following nodes, all
setup with Ubuntu 22.04.

* `kube1.test.lib.msu.edu`, `35.8.223.111`
* `kube2.test.lib.msu.edu`, `35.8.223.112`
* `kube3.test.lib.msu.edu`, `35.8.223.113`

### Container Runtime
Kubernetes requires a CRI available, as noted above. We can use `containerd`
for this purpose. On Ubuntu, this can be installed via:
```
apt install -y containerd
```

**Bug Fix**  
At least [on Debian/Ubuntu](https://github.com/kubernetes/kubernetes/issues/110177),
there exists a bug that will break containerd and
force containers to periodically restart.

To fix this, we setup a manual config file and set `SystemdCgroup` to `true`.
```
mkdir -p /etc/containerd/
containerd config default > /etc/containerd/config.toml
sed -i -e 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
systemctl restart containerd
```

### Disable Swap
While modern K8s does have some support for swap memory, older versions will
not work at all with any swap space. Swap space causes non-trivial problems
for Kubernetes when it wants to provide resource guarantees, so by default it
refuses to run when swap is available.

To avoid issue, you can disable swap entirely.
```
swapoff -a
# Then comment out swap mounts from /etc/fstab
```

### Setup Networking
Kubernetes requires additional networking modules enabled
and must be allowed to forward traffic.

```
# Enable modules by default on startup
tee /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF

# Probe modules for current boot
modprobe overlay
modprobe br_netfilter

# Enable kernel parameteres
tee /etc/sysctl.d/kubernetes.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

# Restart sysctl to read in new parameters
systemctl restart systemd-sysctl
```

Tools you will need include:
* `kubeadm`: To provision and build your cluster.
* `kubectl`: To issue commands to your cluster.
* `kubelet`: The component on each cluster node which manages that specific node.

### Debian Install

Add the signing key need for the package repository.
```
curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-archive-keyring.gpg
```

Add the package repository (Note the `xenial` despite that Ubuntu distro being long expired. Way-to-go, Google.)
```
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

Then install the required packages.
```
apt update
apt install kubeadm kubectl kubelet
```

### Snap Install

All three tools can be installed using `snap` on Ubuntu:
```
snap install --classic kubeadm
snap install --classic kubectl
snap install --classic kubelet
```

Author note: My limited testing with snap these packages resulted in a sub-optimal experience.

### Manual Install

Installation can also be done independently. Instructions are available on:

* https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
* https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/


### Upgrading
Upgrading Kubernetes may require specific upgrade documentation and should only
be done when following the documentation. To avoid bad things from potentially
happening, you can mark the packages to not auto-update.
```
# For apt installs
apt-mark hold kubelet kubeadm kubectl

# For snap installs
snap refresh --hold kubelet
snap refresh --hold kubeadm
snap refresh --hold kubectl
```

## Initialize the Cluster
If everything else is ready, you can create a new cluster by
initializing a master node. On the master node, run the
`kube init` command.

There are many customizations you can set when creating your cluster,
so you may want to refer to the `kubeadm init` [documentation](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init/).
To see what the default values are for any given config option, run:
```
kubeadm config print init-defaults
```

Here is an example command to create a cluster.
```sh
kubeadm init --control-plane-endpoint=kube1.test.lib.msu.edu \
             # IP address to say API server is listening on
             --apiserver-advertise-address 35.8.223.111 \
             # CIDR address range to use (a specific CIDR may be required for certain networks)
             --pod-network-cidr 10.244.0.0/16
```

This will precheck and ensure your node is configured correctly
before proceeding. If your node isn't ready, it will display
errors indicating what you still need to do, but it won't fix
the problems. That's up to you!

*Warning:* While there is a `kubeadm reset` command that is supposed
to reset a node back to a pre-configured, pre-cluster state, it does
a partial job at best. Make certain you get all you configurations
correct for your first `kubeadm init` command and it may save you
some headaches.

### Firewalls
If you have firewalls enabled, such as `ufw`, you will need to allow
[certain traffic through](https://kubernetes.io/docs/reference/networking/ports-and-protocols/).

The Kubernetes API runs on port `6443` by default and is used by both
Kubernetes clients (like `kubectl`) and the nodes themselves.

_However_, you may need additional ports open depending on your
particular cluster. Some networking configurations require
additional ports and you will need to allow those thorugh.

One quick-and-dirty solution is to allow all traffic from all cluster
nodes to all other cluster nodes, then only allow external
traffic to the API port for external IPs which would be issuing
commands to the cluster.

_Example `ufw` rules for node at `35.8.223.111`_:
```
ufw allow from 10.244.0.0/16 to 35.8.223.111
ufw allow from 35.8.223.112 to 35.8.223.111
ufw allow from 35.8.223.113 to 35.8.223.111
```

_Example `ufw` rule to allow API access from external IP `1.2.3.999` to control plane node at 35.8.223.111_:
```
ufw allow from 1.2.3.999 to 35.8.223.111 port 6443 proto tcp
```

### Controlling the Cluster
Where configuring the cluster is done via `kubeadm`, controlling the cluster
uses `kubectl`. The `kubectl` command does not have to be run on a node
in the cluster as Kubernetes is API driven, so you can configure it to run
anywhere (network restriction permitting).

#### Configuration
The configuration for `kubectl` is located in `$HOME/.kube/config` and
needs to be setup before using `kubectl`.

If you are using `kubectl` (as we are in this example), the config for the
newly created cluster can be found at `/etc/kubernetes/admin.conf`. You can
grab this file for use as your config.

```sh
# Setup a user's config
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Or temporarily for root user
export KUBECONFIG=/etc/kubernetes/admin.conf
```

If all worked well, you should be able to see your node via:
```
kubectl get nodes
```

By default, control-plane nodes will not schedule pods. If you want
control-plane nodes, you'll need to:
```
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

Your cluster won't work, as no networking has
been setup. The status of the node is `NotReady`. You'll need to
select and deploy the networking of your choice.

For now, we'll setup networking using _Flannel_.
```
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

To see pods running, including the `kube-system` namespace:
```
kubectl get pods --all-namespaces
```

### Networking

Kubernetes doesn't prescribe a set of networking tools to use either, rather
if uses the _Container Network Interface_ (CNI) to to setup virtual
ethenet devices, routes, assigning IPs, etc. CNI is used by Kubernetes,
but also other solutions, such as Podman, rkt, and Mesos.

Why so many options? Well, why do we need Apache, Nginx, Lighttpd, and
the rest? Open source mean you have options!

Popular options for networking are listed below:

#### Flannel
Flannel is a simple to setup networking option which works well for
many use cases.

* https://github.com/flannel-io/flannel

#### Calico
Considered a more flexible and advanced option for networking, claiming
to be a performance option with support for security policies.

* https://github.com/projectcalico/calico

#### Cillium
Designed to handle large and complex networks more efficiently.

* https://github.com/cilium/cilium

#### Some Additional Options
More networking solutions for those interested in researching them.

* Weave Net: https://github.com/weaveworks/weave
* Antrea: https://github.com/antrea-io/antrea
* Kube-router: https://github.com/cloudnativelabs/kube-router
* Kube-OVN: https://github.com/kubeovn/kube-ovn

For an even more extensive list, check out the readme on the
[CNI GitHub page](https://github.com/containernetworking/cni).

Note that when using a cloud networking provider, you may not have
any choice in what your networking option will be.

### Additional Cluster Nodes
Upon success of first running `kubeadm init`, you may have noticed two messages
display about adding new nodes to the cluster. Something like the following:
```
You can now join any number of control-plane nodes by copying certificate authorities
and service account keys on each node and then running the following as root:

  kubeadm join kube1.test.lib.msu.edu:6443 --token 1up71d.lz7pxx6byine4tn0 \
	--discovery-token-ca-cert-hash sha256:76e3fd8935183ed3e4ce91be9d1ccd314af41426e1a3bde4f182bb6736c613dc \
	--control-plane

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join kube1.test.lib.msu.edu:6443 --token 1up71d.lz7pxx6byine4tn0 \
	--discovery-token-ca-cert-hash sha256:76e3fd8935183ed3e4ce91be9d1ccd314af41426e1a3bde4f182bb6736c613dc
```

These provided instructions on adding nodes to the cluster. Quite simply, run
one of the commands on the new node to add that node to the cluster.
Note the difference between the two commands is one has `--control-plane` and
the other does not. If the `--control-plane` flag is used, the new
node will be an additional control plane; without that flag, the
new node will be a worker.

One additional caveat. Control plane nodes also require the control plane certificates
to be able to join. This can be done [manually](#https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/#manual-certs), or via storing them in a Kubernetes
Secret (more on Secrets later). To use certificates in a Secret, you would need
to add the `--upload-certs` flag to the `kubeadm init` command above. Or if you
forgot, you can run the following afterwards:
```
kubeadm init phase upload-certs --upload-certs
```

This will output a certificate key, which you can then use as part of your
`kubeadm join` command. For example:
```
kubeadm join kube1.test.lib.msu.edu:6443 --token 1up71d.lz7pxx6byine4tn0 \
	--discovery-token-ca-cert-hash sha256:76e3fd8935183ed3e4ce91be9d1ccd314af41426e1a3bde4f182bb6736c613dc \
	--control-plane \
    --certificate-key bd8a57edb8de39921dc747e9bb0dd2c34a82aa4a87ed5195fb4f705f9c66fa01
```

Before attempting to join a new code, ensure your [firewalls](#firewalls) are
configured correctly.

To create a new token and output the command needed to join a new node, run the
following from an existing control plane node:
```
kubeadm token create --print-join-command
```

It will output the command to join a node as a worker. Again, if you want the new
node to be a control plane, add the `--control-plane` flag.

### Cloud Hosted Kubernetes

## What is in a Cluster

* kubelet service
* proxy, dns, apiserver, controller, etcd

### Namespaces

* contexts?

