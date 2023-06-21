# Kubernetes Documentation

It is assumed the reader has an understanding and familiarity with containerization.  

**Written for Kubernetes v1.27**

*A Forewarning when learning Kubernetes:* Kubernetes changes often. If you are using a
different version of Kubernetes, this documentation _may not apply to you_. Likewise,
when searching for answers online, _many_ tutorials, guides, and answers will be
*wrong*, as they were written for older versions.

There are also a multitude of
Kubernetes novices creating simple guides about Kubernetes which are lacking in
context and understanding of the material. For the most part, assume all tutorials
are made by someone who copy-pasted various code snippets on the internet until
something appeared to work. _Be suspect of any answers you find._ (I guess that includes
this doc as well!)

[[_TOC_]]

## What is Kubernetes

**Kubernetes** is a container orchestration system. It manages deployment,
failover, scaling, replacement, and other services for containers (such
as Docker). It is often abbreviated using the numeronym **k8s**. While
Kubernetes can run on a single node (a single machine), one of its
primary features to create a cluster of nodes across which containers
can run.

## Terminology Reference

These definitions may be covered in more depths further on in this guide, but there are provided
here for quick reference.

* **Node** - a machine in the Kubernetes cluster.
* **Control plane** - A master node in the cluster. It runs essential cluster services and can receive API calls.
* **Worker** - An unprivileged node (not a control plane) on which pods can be scheduled to run.
* **Object** - Something that the Kubernetes cluster is managing (e.g. Pod, Deployment, etc).
* **Pod** - One or more containers that are deployed onto a node as a group. Containers in a single pod cannot be split across node and will always be kept together.
* **Manifest** - A definition of an object, usually presented in Yaml format, but can be JSON also.
* **Namespace** - A scope of where objects (such as Pods) exists. The default namespace is `default`. Cluster related pods are in the namespace `kube-system`. You can put all the pods for your specific project into a single namespace to keep it separate from other projects in the same cluster. This also allows for you to query info from just a specific namespace rather than the entire cluster.
* **Context** - A client side set of settings, useful for setting preferences and connection info. In each context you create, you can set things like a different default namespace, a different user to connect as, or a different cluster to use.
* **Label** - A key/value pair which is associated with an object in Kubernetes. They can be used for convenience, but also as settings or flags which help the cluster manage objects. They can be queried and filtered and are generally used to identify objects in the cluster.
* **Selector** - A query for objects using label keys/values. 
* **Annotation** - A key/value pair for arbitrary data which is _not_ used to identify objects. Annotation cannot be queried or filtered, and are generally used for client libraries or tools. These can be some setting than an object requires, or a configuration change that alters how that object operates.
* **Taint** - A key/value pair setting on a node which prevents pods from starting on that node, or can even evict a pod from the node, depending on the taint value. By default, there is a taint on control planes which prevents pods from running on them.
* **Toleration** - A definition in a pod that allows the pod to be scheduled on a node with specific taint(s).
* **Object** - A persistent entity within Kubernetes (e.g. pod, deployments, events, etc.) 
* **Role** - Refers to Role-Based Access Controls (RBAC). Roles are useful for defining groups and limiting permissions within your cluster. Alternatively, can refer to the `node-role` taint which is used to identify a control plane.
* **Resource** - Either used as a _resource type_ when used in an API call. Alternatively, it can refer to computing resources, such as CPU, RAM, or disk.
* **ReplicaSet** - Ensures a certain number of pods are running at once. Generally, it is recommended to use _Deployment_ instead of _ReplicaSets_.
* **Deployment** - A means of handling _ReplicaSets_ which include the ability to do rolling updates. A deployment is version aware of the pods, and can scale newer pods up to replace older pods.
* **Service** - Defines a way of exposing an application to the network.
* **LoadBalancer** - Both a way to load balance traffic, but also how you allow external traffic into your cluster.
* **Ingress** - Definition of routing rules for what services to route traffic to.
* **IngressController** - The service that reads rules from Ingress resource and routes traffic.
* **DaemonSet** - Ensures a single pod is running on each node in the cluster.
* **Probe** - A check as to the status of a pod, used by Kubernetes to determine pod health. There are several types of probes available.
* **Job** - Run a pod, specifically its command, to successful completion. Can run it a given number of times and in parallel.
* **CronJob** - Run a pod, specifically its command, on a given schedule.
* **ConfigMap** - A key and value stored in Kubernetes. Used to store data across the cluster, such as environment variables or config files.
* **Secret** - A key and value, similar to a ConfigMap, but used for secured data, such as security keys, passwords, or other sensitive data.
* **Volume** - Provides storage to a pod which may last beyond a container's lifetime. May, or may not, refer to actual persistent storage.
* **StatefulSet** - Similar to a Deployment, but with additional guarantees about ordering and consistency. Helpful in creating non-stateless services.

## Basics of Kubernetes

At its heart, Kubernetes schedules containers to run and keeps them running
according to a given set of specifications. These containers are run on nodes
that exist in the Kubernetes cluster, which can be small or quite large.

A typical kubernetes task is to "apply" a manifest file. Basically, a
manifest is a Yaml file defining something, like a service. The Yaml file
can be applied (telling K8s to "make it happen"), edited in place if it already
exists (same effect as apply), or deleted ("make it go away").

A simple example manifest file (details of manifests will be covered later).
```yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: my-web-pod
  labels:
    app: my-web
spec:
  containers:
  - name: my-nginx
    image: nginx
    ports:
    - containerPort: 80
      protocol: TCP
```

Kubernetes typically operates in a _declarative_ manner. Rather than specify
the steps needed to get to an end state, you declare what you want the end state
to be. It is then up to Kubernetes to decide what steps are needed.

The alternative is to be _imperative_. While Kubernetes supports some imperative
actions, it is recommended to be declarative whenever possible.

For example, Kubernetes can `apply` on a given manifest (declarative). This means
Kubernetes will work to make that manifest happen. If that manifest was
previously made to happen, Kubernetes will verify that is the case. The `apply` succeeds.

However, Kubernetes can also `create` a given manifest (imperative). This tells
Kubernetes to create something defined by the manifest. If what is defined in the manifest
already exists, even in only in part, then the `create` will fail. Kubernetes cannot
create something that already exists.

Another example, once you have containers running, you will likely want to open
up a port to allow connections into your containers. You could declaratively create
a service manifest file and `apply` it. Or you could imperatively use the
`expose` command to create the service.

## How Kubernetes Operates

Everything in Kubernetes is an object which is presented via an REST API interface
which returns JSON data responses. While the API uses JSON, Kubernetes configs are
almost exclusively Yaml.

We will be using CLI tools to managed our Kubernetes cluster, but these are merely
making a series of API call behind the scenes. The same actions could be completed
via `curl` commands, if you knew specifically what API calls were needed.

Of course, being that Kubernetes is API driven, there are a number of options available
to manage your cluster and the contents within it. However, within this guide, we'll
be using these official tools/services:

* `kubeadm`: To provision and build your cluster.
* `kubectl`: To issue commands regarding what is in your cluster.
* `kubelet`: The component on each cluster node which manages that specific node.

When creating objects in Kubernetes, they are often represented by a subdomain
of your cluster or a path in your API. As such, there are some [restrictions](https://kubernetes.io/docs/concepts/overview/working-with-objects/names/)
on naming. A general rule-of-thumb, avoid any names or labels which might conflict
with being embedded in a URL.

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
Kubernetes will run on a number of Container Runtimes which implement
the _Container Runtime Interface_ (CRI). The two most well known are:

* containerd
* CRI-O

The containerd CRI is actually already installed if you have Docker
installed, as Docker Engine also makes use of containerd. Kubernetes used
to be able to use Docker as its runtime, which allowed for use of `docker`
commands to interact with Kubernetes managed containers. This feature
has been removed in more modern versions of Kubernetes.

For our guide, we can use `containerd`. On Ubuntu, this can be installed via:
```
apt install -y containerd
```

#### Bug Fix
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

#### Interacting with containerd
Direct management of `containerd` is not covered here, except to say you can use
the `crictl` command to interact with `containerd`. You may have to specify the
socket in order to do so.

Example `crictl` commands:
```sh
# List pods (specifying containerd as runtime)
crictl -r unix:///run/containerd/containerd.sock pods
# Stop a pod
crictl stopp POD_ID
# Remove a pod (after it has been stopped)
crictl rmp POD_ID
```

### Disable Swap
While modern K8s does have some support for swap memory, older versions will
not work at all with any swap space. Swap space causes non-trivial problems
for Kubernetes when it wants to provide resource guarantees (e.g. RAM), so by
default it refuses to run when swap is available.

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

### Debian Install

Let's install `kubelet`, `kubeadm`, and `kubectl`. First, add the signing key need for the package repository.
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

All three tools can be also installed using `snap` on Ubuntu:
```
snap install --classic kubeadm
snap install --classic kubectl
snap install --classic kubelet
```

Author note: My limited testing with snap these packages resulted in a sub-optimal experience. You're on your own here.

### Manual Install

Installation can also be done independently. Instructions are available on:

* https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
* https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/


### Versions and Upgrading
**BEWARE!** Kubernetes and other associated services which interact with it can be _very_
sensitive to version changes. Large, breaking changes are frequent when going from one
version to another. You should always install a specific version of a software rather
than the `latest`, as an unexpected change will likely break compatability.

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

While designed to support interoperability within one minor version
(e.g. 1.12 compatible with 1.13), but always read the upgrade notes.

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
`kubeadm init` command.

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
# Setup a user's config so their kubectl will connect to the cluster
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

By default, control plane nodes will not schedule pods. Control
planes come with a [taint](#taints-and-tolerations) which prevents
scheduling of pods to run.

Running this:
```sh
kubectl get nodes -o json | jq '.items[].spec.taints'
```

Will output the taints on your nodes.
```json
[
  {
    "effect": "NoSchedule",
    "key": "node-role.kubernetes.io/control-plane"
  }
]
```

If you want control plane nodes to schedule and run pods, you'll need to
remove the taint.
```
# Remove control plane taint from specific node
kubectl taint nodes kube1.test.lib.msu.edu node-role.kubernetes.io/control-plane-

# Remote control plane taint from all nodes 
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

Even after this, your cluster won't work as networking has not
been setup. You might have noticed the status of the node
is `NotReady`. You'll need to select and deploy the networking
of your choice before things will finally be working.

For now, we'll setup networking using _Flannel_ (we'll answer "Why Flannel?" shortly,
and also go more into the `kube apply` command later in this document).
```
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

One the networking is started, the internal core DNS service should start as well.
To see pods running, including the `kube-system` namespace:
```
kubectl get pods --all-namespaces
```

If all pods looks okay, your cluster should be online.

You can also see the control plane and core DNS urls via:
```
kubectl cluster-info
```

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
documentation, use of Flannel for the CNI choice in our examples keep us
from introducing extra complexity while learning.

* https://github.com/flannel-io/flannel

#### Calico
Considered a more flexible and advanced option for networking, claiming
to be a performance option with support for security policies.

* https://github.com/projectcalico/calico

#### Cillium
Designed to handle large and complex networks more efficiently.

* https://github.com/cilium/cilium

#### Additional Options
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

#### Load Balancing Control Planes

Just because you added extra control plane nodes, it doesn't mean your cluster
will use them. You then have to setup a load balancer of some sort for API
access. How you setup the load balancer [is up to you](https://github.com/kubernetes/kubeadm/blob/main/docs/ha-considerations.md#options-for-software-load-balancing).

If you are setting up a new cluster, you should try creating a
[high-availability cluster from scratch](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/). The instructions below are for attempting to add a control plane to an
existing cluster.

Once your balancer is ready, you'll
need to update the following ([much](https://blog.scottlowe.org/2019/08/12/converting-kubernetes-to-ha-control-plane/)
[credit](https://blog.scottlowe.org/2019/07/30/adding-a-name-to-kubernetes-api-server-certificate/) to Scott Lowe):

* For all nodes (worker and control plane): Update `/etc/kubernetes/kubelet.conf` to change `server:` to your load balancer.
* For control plane nodes: Update `/etc/kubernetes/admin.conf` to change `server:` to your load balancer.
* For control plane nodes: Update `/etc/kubernetes/controller-manager.conf` to change `server:` to your load balancer.
* For control plane nodes: Update `/etc/kubernetes/scheduler.conf` to change `server:` to your load balancer.
* On a control plane node: Update `kube-proxy` ConfigMap by running `kubectl -n kube-system edit cm kube-proxy` and change the `server:` to your load balancer.
* On a control plane node: Update `kube-public` ConfigMap by running `kubectl -n kube-public edit cm cluster-info` and change the `server:` to your load balancer.
* Update the `kube-system` ConfigMap by:
  * Downloading the current config: `kubectl -n kube-system get configmap kubeadm-config -o jsonpath='{.data.ClusterConfiguration}' > /tmp/kubeadm.yaml`
  * Change the `controlPlaneEndpoint:` to your load balancer.
  * Add/update the `certSANs:` key under `apiServer:` to add any additional Subject Alternate Names (SANs) to the cluster's TLS certificate. \
    ```
    apiServer:
      certSANs:
      - "api.cluster.test.lib.msu.edu"
      - "lb.mycluster.example.edu"
      - "192.168.2.254"
    ```
  * If changing SANs, remove the existing certificates: `mv /etc/kubernetes/pki/apiserver.{crt,key} /tmp/`
  * If changing SANs, regenerate new certificates: `kubeadm init phase certs apiserver --config /tmp/kubeadm.yaml`
  * If changing SANS, restart the `kube-apiserver` pod(s) [by stopping/removing them](#interacting-with-containerd). (Kubelet will restart them.)
  * Re-upload your config to the cluster: `kubeadm config upload from-file --config /tmp/kubeadm.yaml`
* Restart `kubelet` and the cluster pods on each node. You can do via `systemd restart kubelet` and [by stopping/removing pods](#interacting-with-containerd) for each one in the `kube-system` namespace, or alternatively just restart the node.

### Cloud Hosted Kubernetes

As you can probably tell by now, setting up and running Kubernetes cluster is
not a trivial task. Many people recommend NOT configuring your own Kubernetes
clusters and just leave that task to cloud providers. These are often
referred to as Kubernetes-as-a-service (KaaS).

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
in our example case). But it does receive specifications for pods (loaded
from our manifests) and makes certain they are running as expected.

Then each nodes makes pods depending on its role. Each of these will be within
the namespace **kube-system**.

**kube-proxy**  
The proxy handles internal network traffic routing for services. In order to
route for all cluster traffic, it must run on every node.

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

**kube-scheduler**  
Determines where pods should run, according to available resources and contraints.
There should be a _controller_manager_ on every control plane node.

**etcd**  
A distributed key-value store used to store Kubernetes objects. Essentially,
this is the datastore for the cluster.
There should be an _etcd_ on every control plane node. However, you can also
setup Kubernetes to use an external _etcd_ store as well, which is not covered
by this documentation.

## Namespaces

Namespace provide a place where names of objects must be unique. However, any given
name could be reused in an alternate namespace.

Cluster pods are in the `kube-system` namespace, where the default namespace is
aptly called `default`.

To create a namespace, first create a namespace manifest file. E.g. `~/namespace-best-app.yaml`:
```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: best-app
```

Use `kubectl` to make the namespace:
```sh
kubectl apply -f ~/namespace-best-app.yaml
```

When specifying a `kubectl` command, you can use the `-n` flag to specify the namespace to use:
```sh
# Get pods from 'default' namespace
kubectl get pods

# Get pods from 'best-app' namespace
kubectl -n best-app get pods
```

To not filter or limit by namespace, you can use the `--all-namespace` or `-A` flag:
```sh
# Get all pods across all namespaces
kubectl get pods -A
```

To see all current namespaces:
```sh
kubectl get namespaces
```

## Using `kubectl`
The main command for interacting with your running cluster is `kubectl`. As you will be
using this command so often, you will be greatly helped by [setting up Bash autocompletion](#autocomplete-for-kubectl)
as soon as possible.

We'll cover many `kubectl` commands in this document, but one of the most used is `kubectl get` which is used
to query data, settings, and more from the cluster.

By default, `kubectl get` outputs a table-like output, typically with limited set of columns. 
```sh
$ kubectl get pods
NAME            READY   STATUS    RESTARTS   AGE
my-double-web   2/2     Running   0          27h
my-single-web   1/1     Running   0          27h
```

In querying objects, you will need to specify what to query. While you could query for `all`, that can be
quite broad once you have many objects.

Typically you will query with the _kind_ and _name_ of the object. Kubernetes supports multiple ways of formatting
this information, including `/` delimited or just using a space. There are also
[abbreviations](#resource-abbreviations) available.

Some examples commands querying for a Pod:
```sh
kubectl get pods/my-single-web
kubectl get pods my-single-web
kubectl get pod my-single-web   # singular form is supported
kubectl get po my-single-web    # po is the supported abbreviation for pods
```

You can modify this output several ways.

* `-o wide` Show additional columns of data in the table format.
* `--no-headers` Hide the header row when outputing in a table format.
* `--show-labels` Add a column when outputing in a table format to dispay labels for the objects.
* `--sort-by` Sort table data by specified data.
* `-w` Monitor for changes and update output when something changes (similar to Unix `watch` command)
* `-o yaml` Show _all_ data in a full dump in Yaml format.
* `-o json` Show _all_ data in a full dump in JSON format.
* `-o custom-columns` Create a customized table format output.

When using `-o custom-columns`, the format is a comma delimited list where each column is `COLUMN_NAME:data_query`, where the data_query is a JSONPath query.
```sh
kubectl get pods -o custom-columns=NAMESPACE:.metadata.namespace,NAME:.metadata.name,NODE:.spec.nodeName,HOSTIP:.status.hostIP,PHASE:.status.phase,START_TIME:.metadata.creationTimestamp
```

When using `--sort-by `, pass a JSONPath query.
```sh
kubectl get pods -sort-by=.status.startTime

You can also perform some filtering on output by use of the `--field-selector` flag. 
```sh
kubectl get pods --field-selector status.phase=Running
```

This is less helpful than it would sound, as there is no documentation as to what fields are available
and no command to be able to query them from Kubernetes itself. You can find examples which mentions some
fields, but those examples are mostly worthless copy-pasta and don't mention how to find the fields.

To actually find fields available, you currently have to delve into the source code (regards to one
individual who has [done just that](https://hoelz.ro/blog/which-fields-can-you-use-with-kubernetes-field-selectors)).

## Labels and Selectors

Labels are used extensively throughout Kubernetes. Labels are key-value pairs which you can assign
to objects. You'll also see Kubernetes itself applying labels in some circumstances. Labels are
useful for categorizing objects and can be queried against to find specific objects that match
certain labels.

Lets look at our namespace example from before, only we'll add some labels. Labels are placed
into the `metadata:` section of manifests.
```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: best-app
  labels:
    app: best
    group: public-apps
```

Here we've added two labels. The key and values are defined by us and could have been anything.

To see all labels in `kubectl get` output, use the `--show-labels` flag.
```sh
kubectl get namespaces --show-labels
```

To see a specific label (or labels), pass in the `-L` flag with the label key.
```sh
kubectl get pods -L app -L group
```

Kubernetes also support querying against labels. This is known as a **selector**. You will often
see `selector:` fields in manifests to filter which objects the manifest is referencing.

To use a selector with `kubectl get`, use the `-l` flag.
```sh
# Exact matching, multiple matches
kubectl get namespaces -l app=best-app,group=public-apps

# Negative match
kubectl get namespaces -l group!=public-apps

# Label key is set
kubectl get namespaces -l group

# Label key is not set
kubectl get namespaces -l !group

# Key value is one of value list
kubectl get namespaces -l 'group in (backend-apps,public-apps)'

# Key value is not one of value list
kubectl get namespaces -l 'group notin (backend-apps,utility-apps)'
```

Best practice is to use manifests to set/update labels, but the `kubectl label`
command can be use to make ad hoc label changes. Be forewarned, if you are trying
to change a label that was set in the object manifest, Kubernetes will just change it back
to match the manifest the instant it changes.
```sh
# Find the pods with label app=my-app and then set lable tier=second
kubectl label pods -l app=my-app tier=second
```

To remove a label manually, set the pass the key followed by a `-`.
```sh
kubectl label pods -l app=my-app tier-
```

While not a requirement, Kubernetes does have a set of
[recommended labels](https://kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/)
for common use. These include frequent needs to describe the application and it's purpose, such as:

* `app.kubernetes.io/name`
* `app.kubernetes.io/version`
* `app.kubernetes.io/part-of`

## Writing Manifests

Objects have _manifests_, which define the desired state of that object, also referred to
as the _specification_, or just _spec_. It is up to the control plane to take actions to
ensure the _status_, or current state, is changed to match the manifest.

If you know your object kind, you can get all the info about a manifest by using
the `kubectl explain` command.

To list manifest fields and description about those fields for a Deployment Spec:
```
kubectl explain deployment
```

This only lists the top level fields of a deployment manifest. To delve down
into sub-fields, you can `.` delimit the fields you are interested in.
```
kubectl explain deployment.spec
kubectl explain deployment.spec.strategy
```

To get a full tree of a manifest layout in a single output, you can use the `--recurse` flag:
```
kubectl explain service --recursive
```

While the underlying API uses JSON, manifest files are typically written in Yaml
(though you could use JSON). As Yaml files support multiple Yaml structures per
file, you can combine objects together when appropriate in a single file.

Here's an example Yaml config with a Deployment and Service manifest.
The service defined provides web access to the `httpd` containers
from the deployment.
```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-web-deployment
spec:
  selector:
    matchLabels:
      my-app: my-website
  replicas: 3
  template:
    metadata:
      labels:
        my-app: my-website
    spec:
      containers:
      - name: httpd
        image: httpd:2.4
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    my-app: my-website
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

Manifests can become quite large and complex. While putting multiple objects
into a single file is possible, keeping manifest definitions in separate files
can help with keeping a more organized Kubernetes config.

### Ad Hoc Objects

Objects can also be created from the command line with commands such
as `kubectl create` and `kubectl expose` (for Services). Rather than creating a
manifest, you pass configurations as flags.

Unless you are testing something small or experimenting, creaing a manifest file
then applying it is the better way to create and manage objects. (Also, keep your manifest
file in version control and within a CI process while you're at it!)

### Create Boilerplate Manifests Quickly

That said, using commands like `kubectl create` can be very handy for populating
a new manifest. By outputing the manifest with flags `-o yaml` and passing
`--dry-run`, you can quickly boilerplace a full manifest in seconds.
```
kubectl create deployment my-website --image=nginx -o yaml --dry-run > my-new-manifest.yaml
```

## Objects and Manifests
While there are quite a different kinds of Kubernetes objects, here are some key ones you
may want to use.

Each object can be defined using manifest file, or alternatively you could use imperative
commands. We'll continue to assume you are using manifest files.

Each manifest will have certain fields required.

__`apiversion:`__  
Each object will have an API version associated with it. APIs can change over
time and this value may also change for each object.

To see all available API versions, run: `kubectl api-versions`
To see all available resources and their associated API version value, run: `kubectl api-resources`

To look up info on a specific version of a resource, pass the `--api-version` flag  to `kubectl explain`:
```
kubectl explain --api-version=apps/v1 deployments
```

__`kind:`__  
Specifies the type of resource this manifest is defining. See `kubectl api-resources` for a complete list.

__`metadata:`__  
Defines metadata about the resource, such as a `name:` (required), `namespace:`, `labels:`, or `annotations:`.

__`spec:`__  
The definition of the resource. To see specifics about the definition, use `kubectl explain`.

### Pods
A [Pod](https://kubernetes.io/docs/concepts/workloads/pods/) is the smallest
deployable container object. It defines what container(s) exists in the Pod
and any attributes about each container. _Note: many values in running pod container cannot be updated._
The Pod has to be replaced in order for these changes to take effect.

_Common fields for Pod spec definitions:_

* `containers.name:` The name of the container within the pod.
* `containers.image:` The registry (optional) and image to use for this container.
* `ports:` Definitions of ports on the container. Used by Services.
* `env:` Define environment variables to be set in the container.
* `envFrom:` Define environment variables to be loaded from ConfigMaps or Secrets.
* `command:` Set the entrypoint for the container (note, this value is passed as a list).
* `args:` The command and arguments for the container.
* `terminationGracePeriodSeconds:` When killing a Pod, how long after sending SIGTERM until a SIGKILL is sent. Default: `30`

For full listing of fields, see `kubectl explain pod.spec`.

Example single container Pod:
```yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: my-single-web
  # namespace: default
  labels:
    app: my-web
spec:
  containers:
  - image: nginx
    name: web
    env:
    - name: MY_VAR
      value: "123"
    ports:
    - containerPort: 80
      protocol: TCP
```

Additional containers beyond the first in a Pod are often referred to as a __sidecar__. There
can be valid reason to have sidecars, but a single Pod shouldn't contain a whole stack of services.
It is better to separate major servies out into their own deployments and facilitate their communication
by using Services.

### Services
A Service make Pods available on a network. While a Pod may define `ports:`, those ports
do not become accessible until a Service is defined to make it so. Creating a Service
will create a ClusterIP to service as a load balancer. The Service will then load balance
to all matching Pods.

Service matches Pods it should be service via a `selector:`, meaning Pods must have appropriate
labels create on them before a Service can connect to them. Of special note, the Pods _only_ need
to match the Service's `selector:` to be service. If the label(s) in question are used by multiple
objects, then the Service will happy load balance for all them simultaneously.

Example Service manifest:
```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  labels:
    app: public-service
spec:
  ports:
  - port: 80
  selector:
    app: nginx-pod
```

By default, Services are of the `type` is `ClusterIP`, which makes the port available
to an IP only within the cluster.

You can make a service available on a host using the `type` of `NodePort`. With `NodePort`
the port on each node is proxied into the service from the node's host IPs. However,
services can only be exposed on a limited port range using this, by default 30000-32767.
You *can* change the `NodePort` range to include reserved ports such as 80 and 443, but it
is strongly discouraged in favor of using LoadBalancers and IngressControllers.

#### Headless Service
By default, Services provide a new ClusterIP which serves as an internal load balancer IP
for accessing the connected Pods. A headless service is one without this load balancing.

To make a service headless, set `ClusterIP: None` in the manifest. The result will be
the service name being a round-robin DNS to the pods. Additionally, with StatefulSets
the DNS name for each replica pod will be available as sub-domains of the service DNS.

With headless service name of `my-service` and StatefulSet name of `my-state` with 3 replicas,
DNS will names would be:

* `my-service` Round robin DNS to the 3 Pod replicas IPs.
* `my-state-0.my-service` The IP address to replica 0.
* `my-state-1.my-service` The IP address to replica 1.
* `my-state-2.my-service` The IP address to replica 2.

### ReplicaSet
ReplicaSets define a way to deploying a set of pod replicas. This can provide redundancy
and load balancing versus a single pod.

ReplicaSets define a numerical `replicas:` field to indicate how many replicas Kubernetes should try
to schedule at a time. It also has a `template:` field, which holds the template for how
the ReplicaSet should create Pods. Just like a Pod definition, the `template:` needs
a `metadata:` and `spec:` section to define the Pod object.

_However_, defining and using ReplicaSets should be avoided in general. While useful to know about,
using other objects like Deployments, DaemonSets, and StatefulSet is strongly encouraged, as they
have additional features beyond ReplicaSets. In fact, they use ReplicaSets behind the scenes.

Example ReplicaSet manifest.
```yaml
---
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: my-web
  labels:
    app: my-web
spec:
  replicas: 3
  selector:
    # Selector here matches labels in the template section
    matchLabels:
      app: my-web
  template:
    # Template of the Pod manifest for replicas
    metadata:
      labels:
        app: my-web
    spec:
      containers:
      - name: nginx
        image: nginx
```

### Deployments
A [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
is a way of managing updates to ReplicaSets. It can handle making
rollouts (deployment of changes) for the Pods in the ReplicaSet. Note that
_you do not define a ReplicaSet for a Deployment_; the Deployment is an object
that contains a ReplicaSet automatically.

If you recall, many things defined within a running Pod cannot be updated. For
example, values in `env:` will not be updated in a Pod. However, Deployments
will detect a change and begin a rollout of the new changes. Tracking
rollouts can be done via `kubectl rollout`.

* `kubectl rollout status deployment my-deployment` The current status a rollout for the given deployment.
* `kubectl rollout pause deployment my-deployment` Pause a rollout for the given deployment.
* `kubectl rollout resume deployment my-deployment` Resume a rollout for the given deployment.
* `kubectl rollout history deployment my-deployment` See a history of rollouts for the given deployment.
* `kubectl rollout undo deployment my-deployment` Rollback the given deployment to the previous revision.
* `kubectl rollout undo deployment my-deployment --to-revision 3` Rollback the given deployment to the specified revision.

Rollouts can be customized quite extensively.

**`.spec.strategy`** values:  

 * `Recreate`: All pods are stopped prior to creating new pods to replace them.
 * `RollingUpdate`: Pods are stopped and replaced in a more gradual manner. (Default)

Additional Deployment options:  

* `.spec.minReadySeconds`: How many seconds a Pod has to be ready before being considered available. Default: `0`
* `.spec.revisionHistoryLimit`: How many previous deployment revisions to allow rolling back to. Default: `10`
* `.spec.strategy.rollingUpdate.maxUnavailable`: Max number of pods that can become unavailable during a RollingUpdate. Can be a numer or percentage. Default: `25%`
* `.spec.strategy.rollingUpdate.maxSurge`: Max number of pod that can be created in excess of the `replicas:` count. These are new pods being prepared before the old pods are removed. Can be a number or percentage. Default: `25%`

An example Deployment manifest:
```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
  labels:
    app: my-web
spec:
  replicas: 3
  minReadySeconds: 15
  revisionHistoryLimit: 4
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 1
  selector:
    matchLabels:
      app: my-web-replicas
  template:
    metadata:
      labels:
        app: my-web-replicas
    spec:
      containers:
      - name: my-web-nginx
        image: nginx
        ports:
        - containerPort: 80
```

### DaemonSets
A [DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/)
is similar to a Deployment, except instead of replicas, a DaemonSet will always run a
single Pod on each of the cluster nodes. DaemonSets will not run on a control plane node if that node
has it's default taint, unless you set the DaemonSet to have tolerance for that taint.

DaemonSets are otherwise very similar to Deployments. There is one notable difference, however. While
`RollingUpdate` is still the default `strategy:` for updates, the DaemonSet does not have a `Recreate`
strategy, but rather a `OnDelete` strategy.

The `OnDelete` strategy updates the manifest, but leaves the old Pods running as they were. Only when
a Pod is manually removed will a new Pod be created, using the new manifest.

Example DaemonSet manifest:
```yaml
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: my-daemon
  namespace: my-global-pods
spec:
  selector:
    matchLabels:
      name: my-daemon-pod
  updateStrategy:
    rollingUpdate:
      maxUnavailable: 1
  template:
    metadata:
      labels:
        name: my-daemon-pod
    spec:
      # Toleration to have the DaemonSet runon control plane nodes; only needed if default taint was left in place
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      containers:
      - name: my-global
        image: my-global-image
```

### StatefulSets
[StatefulSet](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/) are also similar
to Deployments, only they provide additional guarantees. A StatefulSet
is designed to help with creating applications that must maintain a state. Things like datastores (SQL, NoSQL),
indexers (Solr), and the like.

To help in deployment of stateful applications, the StatefulSet offers the following:

* Stable persistant network identifiers, starting from `0` (default). E.g. `my-sset-0`, `my-sset-1`, `my-sset-2`, etc.
* Ordering of creation and startup of new Pods, from lowest number to highest.
* Ordered scaling down of Pods, always removing the highest numbered first.
* On scaling up Pods, it is always done from low identifier first. Pods must be alive and ready before the next Pod is scheduled.
* Volumes (persistant ones, that is) associated with a StatefulSet are not deleted the StatefulSet is scaled down or removed.

Similar to DaemonSets, StatefulSets support `RollingUpdate` and `OnDelete` strategies only.

Additionally, StatefulSets are strongly encouraged (official documentation says required) to
have a [headless service](#headless-service) defined, that is a service _without_ a ClusterIP assigned. By default,
services get a new ClusterIP that does load balancing internal to the cluster. By setting
`ClusterIP: None`, you make the service lack the load balancer IP. Instead the service name will
provide round-robin DNS to each replica in the StatefulSet. Additionally, each individual pod can
be reached at a sub-domain address of the service name.

For the example manifests below, DNS would be:
* `nginx-svc` Round robin for all 3 replicas.
* `my-sset-0.nginx-svc` Access the first replica.
* `my-sset-1.nginx-svc` Access the second replica.
* `my-sset-2.nginx-svc` Access the third replica.

Example StatefulSet and Headless Service manifest:
```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
  labels:
    app: nginx-svc
spec:
  ports:
  - port: 80
  clusterIP: None
  selector:
    app: nginx-sset
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: my-sset
  labels:
    app: nginx-sset
spec:
  replicas: 3
  serviceName: nginx-svc
  selector:
    matchLabels:
      app: nginx-sset
  template:
    metadata:
      labels:
        app: nginx-sset
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
```

### ConfigMaps
A [ConfigMap](https://kubernetes.io/docs/concepts/configuration/configmap/) is a way to
have key-value storage within Kubernetes. As ConfigMap data is stored within Kubernetes,
there are no additional steps to sync the data across nodes in the cluster. All nodes
accessing a ConfigMap will always read the same data.

Updating a ConfigMap results in immediate an immediate update to running containers.
If the running app re-reads the location where the data is presented (e.g. an enviroment variable),
you should have no need to restart anything to make the updated data go live.

ConfigMaps can store both UTF-8 strings (`data:` section) or base64 encoded binary data (`binaryData:`
section). Keys must be unique across both sections (you cannot `data.my-key-name` and `binaryData.my-key-name`
in the same ConfigMap). Each ConfigMap is limited to a max 1MB in total size.

Example ConfigMap manifest:
```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: alumni-settings
data:
  # property-like keys
  versionNum: "7.0"
  accountGreeting: "Welcome XYZ Alumni!"
  # file-like keys
  app.settings: |
    [theming]
    color.primary=white
    color.secondary=green
    layout=left-column
binaryData:
  # binary file-like data in stored in base64
  datFile: |
    QWhveSB0aGVyZSEgSGVyZSBiZSB0aGUgYmluYXJ5
    IGRhdGEsIG1hdGV5ISBZYWFycnJyIQo=
```

ConfigMaps can be referenced within manifests, such as setting environment variables
or mapping files into containers via volume mounts. ConfigMap data set into a comtainter
is read-only; you cannot update a ConfigMap value my modifying the data from within the
container.

Example manifest using ConfigMap data:
```yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: my-configmap-pod
spec:
  containers:
  - name: my-configmap-web
    image: nginx
    env:
    - name: VERSION_NUM
      valueFrom:
        configMapKeyRef:
          name: alumni-settings     # ConfigMap name
          key: versionNum           # data key name
    - name: GREETING_MSG
      valueFrom:
        configMapKeyRef:
          name: alumni-settings
          key: accountGreeting
    volumeMounts:
    - name: my-configmap-vols
      mountPath: "/etc/alumni"
      readOnly: true
  volumes:
  - name: my-configmap-vols
    configMap:
      name: alumni-settings
      # list of file mappings from ConfigMap key to filename in container
      items:
      - key: "app.settings"
        path: "settings.ini"
      - key: "datFile"
        path: "bin.dat"
```

### Secrets
[Secrets](https://kubernetes.io/docs/concepts/configuration/configmap/) are very similar to ConfigMaps,
and have mostly the same features and limitations, but they are
intended for sensitive or confidential data such as keys or passwords. Secrets are _not_ stored in
an encrypted manner by default. Refer to the documentation if you want to secure your Secrets storage.

Secrets are handled differently within Kubernetes, with additional protections on how/if the data is
presenting to Pods.

Of Historical note, ConfigMaps are newer than Secrets, as there were _only_ Secrets in past versions
of Kubernetes.

Secrets manifets are slightly different from ConfigMaps. Secrets can still store both UTF-8 strings
(`stringData:` section) or base64 encoded binary data (`data:` section), but the section key names
are different. Additionally, Secrets can specify a `type:` which will help Kubernetes know more
context about the data. For example, setting a type can change what fields are required to be
defined in the Secret. The default `type:` is `Opaque` which does not have any restrictions or special
handling.

When referencing Secret in a manifest, you can set it to be `optional: true`. When set and the
referenced value does not exist, Kubernetes will ignore it. By default, Secrets referenced
in a manifest are considered to be required.

Example Secret manifest mounting two `stringValues` as files in `/etc/secrets/`:
```yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: my-secrets
type: Opaque
stringData:
  authuser.txt: my-auth-user
  authpass.txt: greatPasswordHere
```

Example manifest using ConfigMap data:
```yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: my-secret-needing-pod
spec:
  containers:
  - name: my-secret-pod
    image: nginx
    volumeMounts:
    - name: etc-secret
      mountPath: "/etc/secrets"
      readOnly: true
  volumes:
  - name: etc-secret
    secret:
      secretName: my-secrets
```

### LoadBalancers
To permit traffic into your cluster from the outside on ports other than `NodePort`, you can
use a LoadBalancer. Kubernetes does not include any LoadBalancer implementations, rather,
these are often provided by the cloud service Kubernetes is running on.

The LoadBalancer object is often implemented differently based on the cloud provider offering
Kubernetes-as-a-Service (KaaS). When not using KaaS, you could use an external load balancer,
but there are options (although limited) for a cluster hosted LoadBalancer.

A LoadBalancer can be linked directly with a Service; that is, a single Service. Also, a LoadBalancer
does not do routing (at least from a Kubernetes perspective, KaaS providers might offer something).
To handle more complex routing or multiple Services, see Ingress and IngressController.

A popular LoadBalancer for self-managed clusters is **[MetalLB](https://metallb.universe.tf/)**
([GitHub](https://github.com/metallb/metallb)).

Note: If you are using Docker Swarm on a node, you may run into a port conflict with MetalLB as
both make use of port 7946.

#### Preparing
MetalLB requires a couple changes to your `kube-proxy` before proceeding. Both IP Virtual Server (IPVS)
and change how ARP requests are handled. By default, `kube-proxy` uses IPTables. IPVS is designed to
for load balancing, whereas IPTables is firewall software.

The `ip_vs` module must be enabled (check via `lsmod`; it's likely already there).

To check if the kube-proxy config needs to be updated, setting `strictARP: true` and `mode: "ipvs"`.

We'll pass it through `kubectl diff` to preview the change:
```sh
kubectl get configmap kube-proxy -n kube-system -o yaml | \
    sed -e "s/strictARP: \w\+/strictARP: true/" | \
    sed -e "s/mode: \"\w*\"/mode: \"ipvs\"/" | \
    kubectl diff -f - -n kube-system
```

If the diff that a change is needed, update it:
```sh
kubectl get configmap kube-proxy -n kube-system -o yaml | \
    sed -e "s/strictARP: \w\+/strictARP: true/" | \
    sed -e "s/mode: \"\w*\"/mode: \"ipvs\"/" | \
    kubectl apply -f - -n kube-system
```

#### Starting
Find a stable version of MetalLB to use (don't apply the `master` branch). Note in this example, we are
using version `v0.13.10`.
```sh
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.10/config/manifests/metallb-native.yaml
```

You will see numerous resources created. Once completed, there will be a new namespace `metallb-system` where MetalLB
exists. Nothing will happen until you start configuring resources with required configurations into the `metallb-system`
namespace.

#### Configuring

There are various way to [configure MetalLB](https://metallb.universe.tf/configuration/). Here we'll do a simple
layer 2 setup. We'll need an `IPAddressPool` and `L2Advertisement`. Note the IP address you specify must be a public
IP address which the LoadBalancer will assign to itself.

Example configuration manifest:
```yaml
---
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: mlb-pool
  namespace: metallb-system
spec:
  addresses:
  - 35.8.223.121/32
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: mlb-ad
  namespace: metallb-system
spec:
  ipAddressPools:
  - mlb-pool
```

#### Using
Once the LoadBalancer is ready, you can connect your Service to it
by setting `type: LoadBalancer`.
```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: my-web
  type: LoadBalancer
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 80
```

For each Service you assign to the LoadBalancer, you will need to ensure there is also an IP address available.
So if you have three Services, the LoadBalancer will need at least three IPs set under `addresses:`.

MetalLB does support a way of sharing IPs. It also supports assigning specific IPs to specfic Services, however
IP assignment is currently reliant upon a Kubernetes feature that is deprecated and will be removed in
future releases.

### Ingress Controller as DaemonSet and HostPort
An IngressController acts as as sort of reverse proxy to make a Service available external
to the cluster. It works with Ingress resources to do this.

There are 3 officially supported IngressControllers, among many other alternatives: https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/

We'll use `ingress-nginx` (one of the 3 official IngressControllers) in this example.

```sh
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/baremetal/deploy.yaml
```

By default, the IngressCluster still only makes use of NodePort. To allow external traffic, we then need to modify it's manifest.

#### Bind to Host Nodes

The first option to allow external traffic is to change settings on the `ingress-nginx-controller` to bind
to the host nodes directly and run as a DaemonSet, meaning each node will have one controller. If you have
the DaemonSet's port bind to the host port rather than a cluster port, the ingress controller will be  able
to listed to the publicly accessible ports (80,443) and then route that traffic appropriately.

Open the `ingress-nginx` manifest Yaml file and find the Service with `name: ingress-nginx-controller`.

TODO ensure DaemonSet
TODO hostNetwork: true
TODO hostPort:

### Ingress Controller with LoadBalancer

Alternatively, assuming you've setup a LoadBalancer to already receive external traffic, we can modify
the `ingress-nginx-controller` service to make use of the LoadBalancer.

We must set `type: "LoadBalancer"`, and then can optionally also set `externalTrafficPolicy: "Local"`.
This second setting will prevent Kubernetes from re-routing the external traffic away from the node
the LoadBalancer decided to route to.

```sh
kubectl get service/ingress-nginx-controller -n ingress-nginx -o yaml | \
    sed -e "s/mode: \"\w\+\"/mode: \"LoadBalancer\"/" | \
    kubectl apply -f - -n ingress-nginx
```

### Ingress
Ingress resources define routing. These do nothing by themselves, rather, they are read by an IngressController
in order to manage traffic.

An Ingress provides rules to accessing a service via the IngressController.
```yaml
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-web-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  # No need to specify ingressClassName if we setup one as the default
  ingressClassName: nginx
  rules:
  - host: "kube.test.lib.msu.edu"
    http:
      paths:
      - pathType: Prefix
        path: "/my-web"
        backend:
          service:
            name: my-web-service
            port:
              number: 80
```

### Job
[Jobs](https://kubernetes.io/docs/concepts/workloads/controllers/job/) are
tasks that you can schedule to have Kubernetes run. Once a Job has completed,
that's all there is. It doesn't do anything else.

Jobs can be run repeatedly and in parallel across the cluster if desired.
By setting `ttlSecondsAfterFinished:`, your jobs can automatically remove themselves
after a specified period once completed.

Example Job manifest:
```yaml
---
apiVersion: batch/v1
kind: Job
metadata:
  name: not-a-dos-attack
spec:
  ttlSecondsAfterFinished: 120
  completions: 30
  parallelism: 6
  template:
    spec:
      restartPolicy: "OnFailure"
      containers:
        - name: minion
          image: curlimages/curl
          args: ["-s", "https://www.cloudflare.com/"]
```

### CronJob
[CronJobs](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/) are
based on Jobs, but run on a repeating schedule. Setting `schedule:` for a CronJob
is in the same format you'd expect in an `/etc/crontab` file.

Example `schedule:` values:

* `*    * * * *` Run every minute of every day
* `*/30 * * * *` Run every half hour
* `15   2 5 * *` Run at 2:15am on the 5th of each month
* `0   17 * * 0` Run at 5pm every Sunday

CronJobs do have additional features availble not normally seen with most cron services.

* `concurrencyPolicy:` Can a CronJob run concurrently with a previous instance of the same job. Values: `Allow` (default), `Forbid`, `Replace`
* `startingDeadlineSeconds:` If a CronJob could not run on schedule, allow for a late start up this this many seconds.
* `successfulJobsHistoryLimit:` Keep this many successful CronJobs in the job history. Default: `3`
* `failedJobsHistoryLimit:` Keep this many failed CronJobs in the job history. Default: `1`

Example CronJob manifest:
```yaml
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: my-cron
spec:
  schedule: "0 * * * *"
  successfulJobsHistoryLimit: 5
  failedJobsHistoryLimit: 5
  concurrencyPolicy: Forbid
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: my-cron-job
            image: busybox
            command:
            - /bin/sh
            - -c
            - echo "It's $( date ) and all's well."
          restartPolicy: Never
```

Note that CronJobs create regular Jobs. For example, when looking for your CronJob jobs
they will be under the `jobs` type.
```sh
kubectl logs jobs/my-cron-28121574
```

## Volumes

Volumes provide a means of storage beyond the lifespan of a container.

### Type: emptyDir
Creates an empty directory when first pod is deployed, which persists on the node where
is was created so long as the pod exists. Even if the pod crashes, the volume
will persist. The volume is removed if that that pod is intentionally stopped, restarted,
updated, or the pod is removed from that specific node.
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: my-containter
    volumeMounts:
    - mountPath: /cache
      name: mycache
  volumes:
  - name: mycache
    emptyDir:
      sizeLimit: 512Mi
```

### Type: hostPath
Mount storage from the host node at from the given path.
Can be dangerous on multi-node cluster as there is no guarantee the pod will always be
scheduled to the same node.
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
spec:
  template:
    metadata:
      # -- snip --
    spec:
      containers:
      - name: my-deploys
        # -- snip --
        volumeMounts:
        - name: mymount
          mountPath: /opt/path
      volumes:
      - name: mymount
        hostPath:
          path: /mnt/node_path
```

### Type: Local PersistentVolume
Works similar to `hostPath` volumes, except Kubernetes will always reschedule the same pod
onto the same node. This means the data will always be persitent to that Pod. Of course when the
node is unavailable, the pod will be offline.
```yaml
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv
spec:
  capacity:
    storage: 2Gi
  volumeMode: Filesystem
  storageClassName: local-storage
  local:
    path: /mnt/my-data
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - kube1.test.lib.msu.edu
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: local-pvc
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: local-storage
  resources:
    requests:
      storage: 2Gi
---
apiVersion: v1
kind: Pod
metadata:
  # -- snip --
spec:
  containers:
  - name: my-app
    # -- snip --
    volumeMounts:
      - name: my-local-storage
        mountPath: /mnt/storage
  volumes:
    - name: my-local-storage
      persistentVolumeClaim:
        claimName: local-pcv
```

### Type: ConfigMap
[ConfigMaps](#configmaps) and [Secrets](#secrets) can also be leveraged as a mount type.
Data set in these can be mapped onto the Pod, albeit read-only. Note that there are
slight difference between ConfigMaps and Secrets. See the relevant documentation for details.

Example mounting `data.files.my-data` from a ConfigMap into a Pod at `/etc/basepath/my-data.txt`:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-config-map
data:
  files.my-data: |
    This is my file!
    Which contains my data!
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
    - name: my-container
      # -- snip --
      volumeMounts:
      - name: cm-volume
        mountPath: /etc/basepath
  volumes:
    - name: cm-volume
      configMap:
        name: my-config-map
        items:
        - key: files.my-data
          path: my-data.txt
```

### Other Volume Options
Additional options for volumes include using external storage, such as NFS mounts. These can
be used in similar manner to the local PersistentVolume, only they aren't local.

There are also cluster storage solutions for creating distributed internal storage within
the cluster. These include:

* [Longhorn](https://longhorn.io/) ([GitHub](https://github.com/longhorn/longhorn)): Block storage hosted in cluster with replicas across nodes.
* [Rook](https://rook.io/) ([GitHub](https://github.com/rook/rook)): Ceph based distributed storage for Kubernetes.
* [OpenEBS](https://openebs.io/) ([GitHub](https://github.com/openebs/openebs)): Distributed storage in Kubernetes with a number of different backends to choose from.

## Checking Pod Health
Kubernetes uses [probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
to determine if a pod containers are up, alive, and ready to
serve requests. By default, each probe always succeeds, so they must be
defined if you want your pods to be monitored.

Beside the three types of probes, there are also multiple methods of probes:
* `exec` Runs a command in the container
* `httpGet` Perform an HTTP request to the specified URL path
* `tcpSocket` Probe which succeeds if it can open a socket to the port specified
* `grpc` Probe using the gRPC Health Checking Protocol

Example Pod manifest with probes defined using `httpGet`:
```yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: my-healthy-pod
  labels:
    app: healthy 
spec:
  containers:
  - name: healthy
    image: healthy-image
    ports:
    - containerPort: 80
      protocol: TCP
      name: http
    - containerPort: 8081
      protocol: TCP
      name: health-port
    # Allow 200 seconds to startup
    startupProbe:
      httpGet:
        path: /health-stat
        port: health-port
      failureThreshold: 20
      periodSeconds: 10
    # Alive so long as can respond in 10 seconds (tolerate 1 failure)
    livenessProbe:
      httpGet:
        path: /health-stat
        port: health-port
      failureThreshold: 2
      periodSeconds: 10
    # Not ready unless it can respond within 3 seconds (no failures allowed)
    readinessProbe:
      httpGet:
        path: /health-stat
        port: health-port
      failureThreshold: 1
      periodSeconds: 3
```

### Liveness Probe
A `livenessProbe:` determines if a container is alive. A conatiner is determines to be alive once it's first
`livenessProbe:` succeeds, after which if it fails a specified number of times, the Pod will be considered
failed and restart the container.

Liveness probes can be independent of how other probes are executed, but a common tactic is to have a liveness
probe be the same command as a readiness probe and give the liveness probe a greater timout value. The result
is that if a container becomes overburdened and become no longer ready, but still condiered alive. A non-ready
container will not be given new requests and this can give the container to recover and hopefully return to a
ready state once the high-load has passed.

### Startup Probe
While a liveness probe might be reasonable during normal container operations, some applications might need
additional time when first coming online or bootstrapping. Rather than loosening the `livenessProbe:` fields
to accomodate this, there exists a `startupProbe:` which runs while a container is starting.
When the first `startupProbe:` succeeds, the container is considered alive and the `startupProbe:` is no
longer used and the liveness probes begin.

Container startup probe commands are often set to be the same as their liveness node commands. Only difference
being the `startupProbe:` will have a higher `failureThreshold` set. This grants the container a longer
period to come alive initially, but then a keeps a closer watch against the container becoming unhealthy once
running.

### Readiness Probe
A `readinessProbe:` is use to determine whether Services should route traffic to the container. If the
readiness probe goes into a failure state, the container non-ready container is no longer sent requests.

Containers with readiness probes may go between ready and not ready for many reasons. High load,
performing large tasks, etc. These are not situations where the container should be restarted. Instead,
Kubernetes can back off of the container to let it complete what it was doing, only sending it further
requests once the container becomes ready again.

### Probe Fields
Probe can each have the following fields set to modify their behavior.

* `initialDelaySeconds:` Do not perform any probes for this many seconds after the container as started. Default: `0`
* `periodSeconds:` Perform the probe every so many seconds. Default: `10`
* `timeoutSeconds:` Probe times out after so many seconds. Default: `1`
* `failureThreshold:` If individual probe attempts fail/timeout this many times, the overall probe has failed. Default: `3`
* `successThreshold:` If a probe has reached the `failureThreshold`, this is how many consecutive successes are needed before the 
probe is considered no longer in a failed state. Only applicable to readiness probes. Default: `1`

## Metrics
Kubernetes makes use of a metrics server, deployed within the `kube-system` namespace to monitor
resource usage. This is not setup by default and is a requirement for use of
the `kube top` command, having resource limits, and use of theautoscaling feature.

By default, the metrics server requires the Kubelet certificates be signed. See
[Kubernetes TLS Certificate](https://kubernetes.io/docs/tasks/tls/managing-tls-in-a-cluster/)
documentation for more details on setting this up.

Alternatively, you may add the `--kubelet-insecure-tls` flag in the `metrics-server` `spec.containers.args`,
availble at https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
(link is to `latest`; beware version compatability). There is also a configuration for High availability, see
the [official repository](https://github.com/kubernetes-sigs/metrics-server) for details.

Example commands:
```sh
# Get resource usage per node
kubectl top nodes

# Get resource usage for pods in all namespaces
kubectl top pods -A
```

## Pod Resource Constraints
Assuming you have a metrics server up and running, you can set resource constraints
on pods by use of requests and limits. ([Documentation](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/))

**Requests** define how much of a resource is required before a pod can be scheduled
on that node. If insufficient resource is available, the pod will not be scheduled on
that node.

**Limits** define the maximum amount of a resource that a pod can use. If the resource
if memory, then a process within the container will be killed. If the main process of a
container tries to exceed the memory limit, that contaner will be restarted.

_Resource Types_  

* `cpu:` CPU cores. Can be specified numerically (`2`, `0.5`) or as millicpu, where `1000m` is the same as `1` CPU.
* `memory:` RAM use. Example allocations: `512Ki`, `256Mi`, `2Gi`, `1Ti`, or `10240` (bytes, aka 10 Kilobytes)

Example manifest with resource constraints:
```yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: my-web-app
    image: my-frontend
    resources:
      requests:
        memory: "32Mi"
        cpu: "0.5"
      limits:
        memory: "256Mi"
        cpu: "1"
  - name: my-data-app
    image: my-backend
    resources:
      requests:
        memory: "512Mi"
        cpu: "750m"
      limits:
        memory: "1Gi"
        cpu: "1500m"
```

## Horizonal Pod Autoscaling
Kubernetes can autoscale Deployments and StatefulSets using [Horizonal Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) (HPA). This allows for Kubernetes to add or remove pods
per the desired specifications. Additional pods can be scheduled under high load, and then
removed once the load has reduced.

Autoscaling is typically based off data provided by the metrics server (CPU and memory), but
Kubernetes does support defining your own custom metrics which can be used instead.

## Events
An [Event](https://kubernetes.io/docs/reference/kubernetes-api/cluster-resources/event-v1/) is a
record of report with the cluster, typically of a state change that has happened. Examples of common
Events are:

* A Pod being scheduled
* Pulling of an image
* A Container being created or started
* Failure to create a volume mount

Most anything that happens within a Kubernetes cluster has an Event associated with it. Events
can be of two types `Normal` and `Warning`.

```sh
# List only Event of type Warning from all namespaces
kubectl get events -A --field-selector type=Warning

# List events for given Pod and watch for new events
kubectl events --for pod/my-web --watch

# List events in Yaml format
kubectl events -o yaml
```

By default, Events are kept for only 1 hour before being removed, but can be modified by setting
the `--event-ttl` flag to the API server. First use `kubeadm config print init-defaults` to find
the appropriate `apiVersion` and `kubernetesVersion`, then create your custom manifest with
the `apiServer.extraArgs.event-ttl` set to your custom value, and apply it.

Example manifest updating Event TTL to 4 hours and 30 mintes.
```yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: 1.27.0
apiServer:
  extraArgs:
    event-ttl: "4h30m0s"
```

## Taints and Tolerations

Taints flag a node such that it restricts scheduling of pods onto that node.
We first saw this with our control plane node, which by default didn't allow
pods to be scheduled there. Crucial `kube-system` pods are exempted from taints.

Taints can have three possible effects:

* `NoSchedule`: Prevent the node from scheduling new pods.
* `PreferNoSchedule`: Avoid scheduling on this node, but not outright prevent it.
* `NoExecute`: Not only prevent pods from being scheduled on this node, but evict any already running pods off this node.

The other side of taints are tolerations. Tolerations are defined in the manifest and allow that
manifest to tolerate the taint, and thus be scheduled on the tainted node.

One example of this might be to handle hardware differences of nodes. Or certain containers ought
to be scheduled on specific nodes. Say one node has powerful GPU installed
and you prefer to only run GPU heavy pods there. You can set a taint to handle this.

```sh
#                   NodeName     Key=Value:Effect
kubectl taint nodes my-gpu-node1 gpu=available:NoSchedule
```

This taint will prevent pods from being scheduled on `my-gpu-node1`. Then in our deployment
we could add a toleration for our GPU using app.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  # -- snip --
spec:
  # -- snip --
  template:
    metadata:
      # -- snip --
    spec:
      containers:
      - name: gpu-runner
        image: gpu-processor
      tolerations:
      - key: gpu
        operator: Equal
        value: available
        effect: NoSchedule
```

This deployment would now be schedulable onto our tainted node without issue.

To query for Taints using `jq`:
```sh
kubectl get nodes -o json | jq '.items[].spec.taints'
```

To remove Taints, pass the taint key followed by a `-` character.
```sh
# Remove a taint from all nodes
kubectl taint nodes --all gpu-

# Remove a taint from specific node
kubectl taint nodes my-gpu-node1 gpu-
```

## Pod Deployment Spread
If you have 3 nodes and want to deploy a set of 2 replica pods, you would likely not want
both replicas to on the same node. If that node went down, both replicas would go down at
the same time.

To manage how pods are deployed across nodes, Kubernetes provides topology constraints
as a means of managing this. ([Documentation](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/))

Example of spreading replicas at max 1 per node:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
  labels:
    app: my-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-deployment
  template:
    metadata:
      labels:
        app: my-deployment
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: my-deployment
      containers:
      - name: my-nginx
        image: nginx
```

## Commands Reference

Refer to [Using kubectl](#using-kubectl) earlier in this document for a quick intro on `kubectl`.

### [`kubectl get`](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#get)
Get information about a resource.
```sh
# Get a list of nodes in the cluster
kubectl get nodes
# Get a list of deployments in the default namespace
kubectl get deployments
# Get a list of deployments in the 'prod-web' namespace
kubectl get -n prod-web deployments
# Get a list of deployments in the all namespace
kubectl get deployments -A
# Get the full yaml dump for the specified Pod
kubectl get pod/my-web -o yaml
# Get info on all resources in the cluster in all namespace, output in wide table format
kubectl get all -A -o wide
```

### [`kubectl describe`](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#describe)
Show detailed information about a resource.
```sh
kubectl describe pod my-pod
kubectl describe -n kube-system deployment coredns
```

### [`kubectl edit`](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#edit)
Open an existing resource in an editor and edit it. Resource updated upon save and quit.
```sh
kubectl edit -n my-namespace deployment my-deployment-name
kubectl edit service my-service-name
kubectl edit -n my-namespace configmap my-settings-map
```

### [`kubectl logs`](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#logs)
To view logs (stdout, stderr) for Pods:
```sh
kubectl logs -n my-namespace pod/my-podname
# Get the logs for the previous instance of the given Pod
kubectl logs -n kube-system pod/my-podname --previous
# Follow log output with -f
kubectl logs -f pods my-nginx
# To get all logs for conainers in a Deployment/DaemonSet/StatefulSet, you must use label selectors
kubectl logs -n kube-system -f -l k8s-app=kube-dns
```

### [`kubectl exec`](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#exec)
Run a process (such as a shell) within an existing container. You can exec to any container
regardless of where your connecting from.
```sh
# Connect to primary container in a Pod
kubectl exec -i -t pods/my-shell-demo -- /bin/bash
# Connect to a sidecar container in a Pod
kubectl exec -it pods/my-double-pod --container logger-container -- /bin/bash
```

### [`kubectl cp`](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#cp)
Copy a file or directory either to or from a container.
```sh
# Copy a file from /tmp/ on host into /root/ dir in container for the given namespace
kubectl copy -n project-namespace /tmp/myfile.tgz my-pod:/root/
# Copy a file out from the container onto the host node for default namespace
kubectl copy my-pod:/etc/my-config.cfg /tmp/
```

### [`kubectl apply`](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#apply)
Apply a manifest configuration to create or update a resource. Only the parts of the resource
which are being modified need to be specified within the manifest.
```sh
# Local manifest
kubectl apply -n my-namespace -f my-manifest.yaml
# Remote manifest
kubectl apply -f https://k8s.io/examples/application/nginx/nginx-deployment.yaml
```

### [`kubectl diff`](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#diff)
See the difference between the provided manifest file and resource within Kubernetes.
Will also display if there are any change would not be valid.
`kubectl diff -f manifest.yaml`

### [`kubectl replace`](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#replace)
Replace existing resources with ones defined in the provided manifest. Will _not_ replace objects
which have immutable values which cannot be changed.
To force a resources to be fully removed first, allowing for updating immutable values,
you can provide the `--force` flag. This will ensure objects are deleted and recreated.
```sh
kubectl replace -f my-deployment.yaml --force
```

### [`kubectl rollout`](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#rollout)
TODO monitor and manage a rollout

### [`kubectl delete`](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#delete)
Completely delete resourced defined by the provided manifest.
```sh
kubectl delete -f my-manifest.yaml
```

### [`kubectl create`](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#create)
TODO create most resources imperatively
TODO imperative create calls will fail if already exists

### [`kubectl patch`](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#patch)
TODO update API objects in place

### [`kubectl expose`](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#expose)
TODO create a service imperatively

### [`kubectl run`](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#run)
TODO run a container imperatively

### [`kubectl attach`](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#attach)
TODO attach to a containers already running process (versus `run` which creates a new process)

### [`kubectl label`](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#label)
TODO set/modify labels

### [`kubectl annotate`](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#annotate)
TODO set/modify annotations

### [`kubectl scale`](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#scale)
TODO beware, as deployments will auto-correct your replica count
`kubectl scale deployments.apps -n kube-system coredns --replicas=3 deployment.apps/coredns scaled`

### [`kubectl autoscale`](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#autoscale)
TODO create HPA imperatively

## Helm: The Kubernetes Package Manager

Helm is an extra tool which can be used to quickly and easily deploy complete applications,
including all the object required. As most installtions need some changes, it allows for you
to customize parts of the application as well.

Installtion of Helm: https://v2.helm.sh/docs/using_helm/#installing-helm

Helm is similar to other package managers, just with slightly different terminology because it's
not like Kubernetes doesn't already have a hundred other bespoke terms you need to keep track of.

* Repository: A URL location where definitions for Helm are hosted.
* Chart: The name Helm uses to define package.
* Release: An instanced of a deployed chart.
* Artifact Hub: A central listing of repositories.

### Finding a Chart
Charts can be found be found on the Artifact Hub (Hub) by searching:
```sh
helm search hub TERM
```

Repositories not on the Hub can be added manually and require a name to be associated with it.
```sh
#             RepoName RepoUrl
helm repo add traefik  https://traefik.github.io/charts
```

You can then search for Charts in your added repositories by:
```sh
helm repo add traefik https://traefik.github.io/charts
```

### Installing a Package as a Release
The most common way of installing a Chart is via `helm install` and
providing a name for the release and the Chart name:
```sh
# Name the Release yourself
#            Release    Chart
helm install my-traefik traefik/traefik

# Or let Helm create a name for you
helm install traefik/traefik --generate-name
```

Alternative way of installing Chart include via a tarball, an
extracted tarball, or a URL to a tarball.
```sh
helm install my-app my-app.tar.gz
helm install my-app /path/to/my-app/
helm install my-app https://my-app.example.edu/charts/my-app.tgz
```

To see the state of a release:
```sh
helm status my-app
```

To see a list of all current releases on your cluster:
```sh
helm list
```

#### Customizing a Release
Stock versions of Charts often need customization to become useful to
your specific circumstances. To customize a release, you can provide
a set of values in a Yaml to modify the default install values.

TODO

### Upgrading a Release
TODO

### Rollback of a Release
TODO

### Removing a Release
Uninstalling a release can be done by simply:
```sh
helm uninstall my-app
```

## Kustomize: Templating for Kubernetes

The Kustomize tool allows for creating manifest templates. Previously, this was a standalone tool,
but has since been integrated into Kubernetes natively via `kubectl kustomize`.

Kustomize allows you to create a `kustomization.yaml` to list all your resources and apply it in
one go via `kubeadm apply -k <dir_containing_kuztomization_yaml>` (note the `-k` flag instead of
the typical `-f`).

The `kustomization.yaml` file could be something like (where the other yaml files are in the same directory):
```yaml
resources:
- configmap.yaml
- deployment.yaml
- service.yaml
```

You can then create patches that apply to certain environments (e.g. `dev`, `stage`, `prod`).
Say the above file was a template of shared manifests located in a `myproject/base/`.

You could create another `kustomize.yaml` file in `myproject/dev/` which includes the `base` resources
and then builds upon them, adding resources, applying label, and patching the base manifests.
```yaml
namePrefix: dev-
resources:
- ../base
- mailcatcher.yaml
commonLabels:
  appEnv: dev
patchesStrategicMerge:
- set_replicas_to_1.yaml
```

Where set `set_replicas_to_1.yml` is a partial manfiest, such as:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-web
spec:
  replicas: 1
```

## Other Helpful Tips

### Host Service Logs
Logs for containerd and kubelet go into `/var/log/syslog` by default. If you'd like them configured
with separate log files, you must add a configuration in `rsyslog`.

TODO rsyslog config

### Resource Abbreviations
You may have noticed that when referencing resources, both plural singular are allowed.
Additionally, many resource kinds have abbreviations you can use, also known as short names.
For example, these three commands are equivalent:
```
kubectl get services -A
kubectl get service -A
kubectl get svc -A
```

Here is a partial list of short names, but you can always see the full list
by running `kubectl api-resources`.

| Full name | Short name |
| --------- | ---------- |
| `configmaps` | `cm` |
| `cronjobs` | `cj` |
| `daemonsets` | `ds` |
| `deployments` | `deploy` |
| `events` | `ev` |
| `horizonalpodautoscalers` | `hpa` |
| `ingresses` | `ing` |
| `namespaces` | `ns` |
| `nodes` | `no` |
| `persistentvolumeclaims` | `pvc` |
| `persistentvolumes` | `pv` |
| `pods` | `po` |
| `replicasets` | `rs` |
| `services` | `svc` |
| `statefulsets` | `sts` |

### Autocomplete for kubectl
Enabling tab autocomplete can make it considerably easier to use `kubectl`. To
enable it for Bash:
```
# Enable kubectl autocomplete in bash for current shell
source <(kubectl completion bash)

# Enable kubectl autocomplete system wide
echo "source <(kubectl completion bash)" > /etc/bash_completion.d/kubectl
```

### Other Implementations
As Kubernetes is just a set of APIs, you don't technically need `kubeadm` or `kubectl`. You could
just make API calls, or even make your own alternate commands, or recreate the official tools
in your own way. And people have done just that. Here are a few.

* Minikube: Mainly used for learning Kubernetes. Not for production. Many tutorials use this, but it can cause confusion later on as some practices learned in Minikube does not transfer to real Kubernetes.
* KinD: Kubernetes in Docker. Run a Kubernetes cluster in Docker. Not for production.
* MicroK8s: Easier Kubernetes with Sane Defaults.
* K3s: A "Lightweight" Kubernetes, using less memory and having smaller binaries.

### From Docker Compose to K8s using Kompose

A tool to try to convert docker-compose files to Kubernetes configs: https://kompose.io/

Unlikely that you would want to use the output, but it might help in the process.

### Docker Swarm Comparisons

If you are coming from Docker Swarm, here is a table of closest equivalent features in Kubernetes.

| Docker Swarm | Kubernetes | Notes |
| ------------ | ---------- | ----- |
| Service (non-public) | Deployment + Service | |
| Service (public) | DaemonSet + Service + IngressController + Ingress | |
| Service (public) | LoadBalancer + Deployment + Service + IngressController + Ingress | Not quite 1-to-1 conversion, but a better configuration. |
| Replicated Service | Deployment |  |
| Global Service | DaemonSet |  |
| Health Check | Liveness Probe | Also similar to Readiness Probe |

And here is a table if you are looking for a rough equivalent command from `docker` using `kubectl`.

| Docker and Swarm (`docker`) | Kubernetes (`kubectl`) | Notes |
| ------------ | ---------- | ----- |
| `stack deploy` | `apply` |  |
| `exec` | `exec` |  |
| `logs` | `logs` |  |
| `service logs <SERVICE>` | `logs deployment/<DEPLOYMENT>` |  |
| `<TYPE> inspect` | `<TYPE> describe` |  |

