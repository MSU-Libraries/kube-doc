# Kubernetes Documentation

_Written for Kubernetes v1.27_

[[_TOC_]]

## What is Kubernetes

**Kubernetes** is a container orchestration system. It manages deployment,
failover, scaling, replacement, and other services for containers (such
as Docker). It is often abbreviated using the numeronym **k8s**. While
Kubernetes can run on a single node (a single machine), one of its
primary features to create a cluster of nodes across which containers
can run.

## Terminology Reference

* **Node** - a machine in the Kubernetes cluster.
* **Control plane** - A master node in the cluster. It runs essential cluster services and can receive API calls.
* **Worker** - An unprivileged node (not a control plane) on which pods can be scheduled to run.
* **Pod** - One or more containers that are deployed onto a node as a group. Containers in a single pod cannot be split across node and will always be kept together.

TODO
* label
* role
* namespace

## How Kubernetes Works

TODO
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

Kubernetes is known for making fairly significant changes in new versions.
Old features can deprecated and removed and new changes are introduced 
often. Note the version of Kubernetes this documentation was written for
(located at the top of this document). If you are working with newer or older
versions of Kubernetes, your experience will likely not follow this documentation
exactly.

As such, upgrading Kubernetes can require significant changes as well.
Be forewarned that upgrades may not be a trivial change. Research any
changes completely before proceeding.

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

By default, control-plane nodes will not schedule pods. They come
with a taint (more on taints later) which prevents scheduling of
pods to run.

Running this:
```sh
kubectl get nodes -o json | jq '.items[].spec.taints'
```

Will output the taints on your nodes
```
[
  {
    "effect": "NoSchedule",
    "key": "node-role.kubernetes.io/control-plane"
  }
]
```

If you want control-plane nodes to schedule and run pods, you'll need to
remove the taint.
```
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

Even after this, your cluster won't work as no networking has
been setup. You might have noticed the status of the node
is `NotReady`. You'll need to select and deploy the networking
of your choice before things will finally be working.

For now, we'll setup networking using _Flannel_ (we'll answer "Why Flannel?" shortly,
and also cover the `kube apply` command later in this document).
```
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

One the networking is started, the internal core DNS service should start as well.
To see pods running, including the `kube-system` namespace:
```
kubectl get pods --all-namespaces
```

If all pods looks okay, your cluster should be online.

### Networking

Kubernetes doesn't prescribe a set of networking tools to use, rather
if uses the _Container Network Interface_ (CNI) to to setup virtual
ethenet devices, routes, assigning IPs, etc. CNI is used by Kubernetes,
but also other solutions, such as Podman, rkt, and Mesos.

Why so many options? Well, why do we need Apache, Nginx, Lighttpd, and
the rest? Open source mean you have choices, whether you like it or not.

Popular options for networking are listed below:

#### Flannel
Flannel is a simple to setup networking option which works well for
many use cases. Unless you need something more complex or are missing
a needed feature, Flannel is a good place to start.

Due to its minimal setup, Flannel makes a great choice for guides such as
this one, as it reduces the number of extra things that need to be explained
and configured. Considering there is so much else being covered in this
documentation, use Flannel for the CNI choice in our examples keep us
from introducing extra complexity while learning.

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
`kubeadm join` command via a `--certificate-key` argument. For example:
```
kubeadm join kube1.test.lib.msu.edu:6443 --token 1up71d.lz7pxx6byine4tn0 \
	--discovery-token-ca-cert-hash sha256:76e3fd8935183ed3e4ce91be9d1ccd314af41426e1a3bde4f182bb6736c613dc \
	--control-plane \
    --certificate-key bd8a57edb8de39921dc747e9bb0dd2c34a82aa4a87ed5195fb4f705f9c66fa01
```

Before attempting to join a new code, ensure your [firewalls](#firewalls) are
configured correctly on all the nodes you'll be including in your cluster.

If you don't have the output from your initial `kubeadm init` command, you can create
new tokens for joining nodes. To create a new token and output the command needed to
join a new node, run the following from an existing control plane node:
```
kubeadm token create --print-join-command
```

It will output the command to join a node as a worker. Again, if you want the new
node to be a control plane, add the `--control-plane` flag and appropriate
`--certificate-key` (or have manually copied the certs up before). Remember, if
you want a control plane node to schedule and run pods, you will need to remove
the default taint they have:
```
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

If you plan on having multiple control planes, it is strongly recommended to
have them in odd numbers (i.e. 3, 5, 7). This should help ensure a majority
quorum can be maintained in the even of a control plane loss (avoiding a split
vote for new leaders).

### Cloud Hosted Kubernetes

As you can probably tell by now, setting up and running Kubernetes cluster is
not a trivial task. Many people recommend NOT configuring your own Kubernetes
clusters and just leave that task to cloud providers. These are often
referred to as Kubernetes-as-a-service.

Most cloud service providers have a fast way to provision and configure a
ready-to-use Kubernetes cluster, where you can start deploying
containers using `kubectl apply` almost immediately. Cloud service providers
makes have their own Kubernetes provisioning tools, so each one will
likely be slightly different.

Some of the larger providers are:

* Google Kubernetes Engine (GKE)
* Amazon Elastic Kubernetes Service (EKS)
* Microsoft Azure Kubernetes Service (AKS)

There are also many smaller providers with their own methods of provisioning,
but in all cases, they provide CLI tools and/or a web interface which will
create the cluster for you, meaning you can avoid firewalls, `kubeadm` commands,
and many of the other tasks we covered in creating our own cluster.

## What Makes up a Cluster

A cluster is made up of many parts, with nodes (both control plane and worker)
being the base on which everything else runs. And on each node runs a CRI
(`containerd` in our example case), as we covered earlier.

Also on each node, the _kubelet_ runs. This is not a container or pod, but
a service running on the host machine. Think of it as the client or agent of
Kubernetes cluster. It doesn't run the containers; the CRI does that (`containerd`
in our example case). But it does receive specifications for pods (aka PodSpecs)
and makes certain they are running as expected.

Then each nodes makes pods depending on its role. Each of these will be within
the namespace **kube-system**.

**kube-proxy**  
The proxy handle network traffic routing for services. To route traffic for
loadbalancing, it must run on every node.

**coredns**  
Internal DNS service to provide name resolution and service discovery within
the cluster. This is a replicated deployment running on the pod network, which
is why DNS won't start until you start a CNI (_Flannel_ in our example).

You should see multiple DNS pods running. This is to provide redundancy in the
DNS service. You can scale this number up or down if you'd like. (See the
`kube scale` command covered later in this document.)

**kube-apiserver**  
Provides the REST API server which `kubectl` and other services connect to.
There should be an _apiserver_ on every control plane node.

**kube-controller-manager**  
Monitors cluster state versus desired state, making changes when needed
to move the cluster toward the desired state.
There should be a _controller_manager_ on every control plane node.

**etcd**  
A distributed key-value store used to store Kubernetes objects. Essentially,
this is the datastore for the cluster.
There should be an _etcd_ on every control plane node. However, you can also
setup Kubernetes to use an external _etcd_ store as well, which is not covered
by this documentation.

### Roles

TODO A node role is just a label with the format node-role.kubernetes.io/<role>
kubectl label node cb2.4xyz.TODO.com node-role.kubernetes.io/worker=worker

TODO removing role
kubectl label node cb2.4xyz.TODO.com node-role.kubernetes.io/worker-

### Namespaces

* contexts?

## kubeadm Commands

TODO common flags
 * --help
 * --all-namespaces -A
 * -n NAMESPACE
 * -o wide (more columns for `kubectl get`)
 * -o json (display json response; useful for passing into jq)
 * --show-labels (show labels for `get pods`)

TODO filtering
`kubectl get pods --field-selector spec.nodeName=node1`


TODO common and useful commands, links to docs

TODO customizing output
```
kubectl get nodes -o json
kubectl get pods -o custom-columns=NAMESPACE:.metadata.namespace,NAME:.metadata.name,NODE:.spec.nodeName,HOSTIP:.status.hostIP,PHASE:.status.phase,START_TIME:.metadata.creationTimestamp --sort-by=.metadata.creationTimestamp --no-header
```

## kubectl Commands

TODO common and useful commands, links to docs

`kubectl describe deployment.apps -n kube-system coredns`

`kubectl scale deployments.apps -n kube-system coredns --replicas=3 deployment.apps/coredns scaled`

## Helm: The Kubernetes Package Manager

TODO

## Helpful Tips

### Autocomplete for kubectl
Enabling tab autocomplete can make it considerably easier to use `kubectl`. To
enable it for Bash:
```
# Enable kubectl autocomplete in bash for current shell
source <(kubectl completion bash)

# Enable kubectl autocomplete system wide
echo "source <(kubectl completion bash)" > /etc/bash_completion.d/kubectl
```
