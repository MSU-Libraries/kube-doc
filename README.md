# Kubernetes Documentation

_Written for Kubernetes v1.27_

Assumed understanding and familiarity with containerization.

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
* **Pod** - One or more containers that are deployed onto a node as a group. Containers in a single pod cannot be split across node and will always be kept together.
* **Manifest** - A definition of an object, usually presented in Yaml format, but can be JSON also.
* **Namespace** - A scope of where objects (e.g. pods) exists. The default namespace is `default`. Cluster related pods are in the namespace `kube-system`. You can put all the pods for your specific project into a single namespace to keep it separate from other projects in the same cluster. This also allows for you to query info from just a specific namespace rather than the entire cluster.
* **Context** - A client side set of settings, useful for setting preferences and connection info. In each context you create, you can set things like a different default namespace, a different user to connect as, or a different cluster to use.
* **Label** - A key/value pair which is associated with an object in Kubernetes. They can be used for convenience, but also as settings or flags which help the cluster manage objects. They can be queried and filtered and are generally used to identify objects in the cluster.
* **Annotation** - A key/value pair for arbitrary data which is _not_ used to identify objects. Annotation cannot be queried or filtered, and are generally used for client libraries or tools. These can be some setting than an object requires, or a configuration change that alters how that object operates.
* **Taint** - A key/value pair setting on a node which prevents pods from starting on that node, or can even evict a pod from the node, depending on the taint value. By default, there is a taint on control planes which prevents pods from running on them.
* **Toleration** - A definition in a pod that allows the pod to be scheduled on a node with specific taint(s).
* **Object** - A persistant entity within Kubernetes (e.g. pod, deployments, events, etc.) 
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
* **Volume** - Provides storage to a pod which may last beyond a container's lifetime. May, or may not, refer to actual persistant storage.
* **StatefulSet** - Similar to a Deployment, but with additional guarantees about ordering and consistency. Helpful in creating non-stateless services.

## Basics of Kubernetes

At its heart, Kubernetes schedules containers to run and keeps them running
according to a given set of specifications. These containers are run on nodes
that exist in the Kubernetes cluster, which can be small or quite large.

A typical kubernetes task is to "apply" a manifest file. Basically, a
manifest is a Yaml file defining something, like a service. The Yaml file
can be applied (telling K8s to "make it happen"), edited in place if it already
exists (same effect as apply), or deleted ("make it go away").

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
`kubectl expose` command to create the service.

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
* `kubectl`: To issue commands to your cluster.
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
planes come with a taint (more on taints later) which prevents
scheduling of pods to run.

Running this:
```sh
kubectl get nodes -o json | jq '.items[].spec.taints'
```

Will output the taints on your nodes
```json
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

### Namespaces

Namespace provide a place where names of objects must be unique. However, any given
name could be reused in an alternate namespace.

Cluster pods are in the `kube-system` namespace, where the default namespace is
aptly called `default`.

To create a namespace, first create a namespace manifest file. E.g. `~/namespace-best-app.yaml`:
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: best-app
  labels:
    name: best-app
```

Use `kubectl` to make the namespace:
```sh
kubectl apply -f ~/namespace-best-app.yaml
```

When specifying a `kubectl` command, you can use the `-n` flag to specify the namespace to use:
```
# Get pods from 'default' namespace
kubectl get pods

# Get pods from 'best-app' namespace
kubectl -n best-app get pods
```

To not filter or limit by namespace, you can use the `--all-namespace` or `-A` flag:
```
# Get all pods across all namespaces
kubectl get pods -A
```

To see all current namespaces:
```
kubectl get namespaces
```

## Volumes

Volumes provide storage beyond the operation of a pod.

### Type: emptyDir
Creates an empty directory when first pod is deployed, which persists on the node where
is was created so long as the pod exists. Note, even if the pod crashes, the volume
will persist. The volume is removed if that that pod is intensionally restarted or the pod
is removed from the node.
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

#### Type: hostPath
Mount storage from the host node at from the given path.
Can be dangerous on multi-node cluster as there is no guarantee the pod will always be
on the same node.
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
              mountPath: /opt/container_path
      volumes:
        - name: mymount
          hostPath:
            path: /mnt/node_path
```

#### Type: Local PersistantVolume
Works similar to `hostPath` volumes, except Kubernetes will always reschedule the same pod
onto the same node. This means the data will always be persitant to that pid. Of course when the
node is unavailable, the pod will be offline.
```yaml
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

#### Type: ConfigMap
TODO ConfigMap (can only be read-only)
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

#### Other options
TODO longhorn: block storage hosted in cluster with replicas across nodes
TODO also many other options, such as external NFS, but via external providers (mounted similar to local pv)

## kubeadm Commands

TODO common flags
 * --help
 * -n NAMESPACE
 * -o wide (more columns for `kubectl get`)
 * -o json (display json response; useful for passing into jq)
 * -o yaml
 * --show-labels (show labels for `get pods`)

TODO filtering
`kubectl get pods --field-selector spec.nodeName=node1`

TODO common and useful commands, links to docs

TODO customizing output
```
kubectl get nodes -o json
kubectl get pods -o custom-columns=NAMESPACE:.metadata.namespace,NAME:.metadata.name,NODE:.spec.nodeName,HOSTIP:.status.hostIP,PHASE:.status.phase,START_TIME:.metadata.creationTimestamp --sort-by=.metadata.creationTimestamp --no-header
```

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

## Labels

TODO

get specific label values
```
kubectl get pods -Lapp -Ltier -Lrole
```

filter by label values (i.e. selector)
```
kubectl get pods -lapp=guestbook,role=slave
# supports k=v,k!=v,k in (v1,v2), key notin (v1,v2)
           key, (meaning key is set)
           !key (meaning key is not set)
```

update labels (filter by app, update tier)
```
kubectl label pods -l app=nginx tier=fe
```

it's not just you who uses labels, kubernetes uses labels/selectors internally to manage things

While not a requirement, Kubernetes does have a set of
[recommended labels](https://kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/)
for common use. These include frequent needs to describe the application and it's purpose, such as:

* `app.kubernetes.io/name`
* `app.kubernetes.io/version`
* `app.kubernetes.io/part-of`

## Objects
While there are quite a different kinds of Kubernetes objects, here are some key ones you
may want to use.

### Deployment
TODO

`kubectl rollout` # status, pause, resume, history, undo

strategies: recreate, rollingupdate
 - `minReadySeconds`
 - `progressDeadlineSeconds`
 - `updateStrategy.rollingUpdate.maxUnavailable` define the number of pods wich can upgraded at once


### DaemonSet
TODO

### ConfigMap and Secrets
TODO

Updating either of these results in immediate update to running containers.
If the running app can re-read the location where the data is presented, it
would have no need to restart anything to get the updated data.

Can store either:
* utf-8 string
* binary data in base64

max size: 1 MB

https://kubernetes.io/docs/concepts/configuration/configmap/
https://kubernetes.io/docs/concepts/configuration/secret/

### Service
A service provides a way to export a port, either internally or externally. By default,
the `type` is `ClusterIP`, which makes the port available to an IP only within the cluster.

You can make a service available on a host using the `type` of `NodePort`. However,
services can only be exposed on a limited port range using this, by default 30000-32767.

With `NodePort`, the port on each node is proxied into the service from the node's host IPs.

You *can* change the `NodePort` range to include reserved ports such as 80 and 443, but it is discouraged.

### LoadBalancer
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

#### Preparation
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

#### Apply Manifest
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
TODO
- one shot
- parallel fixed completion
- work queue: parallel jobs

### CronJob
TODO

## Checking Pod Health
Kubernetes uses probes to determine if a pod is up, alive, and ready to
serve requests. By default, each probe always succeeds, so they must be
defined if you want your pods to be monitored.

### Startup Probe
TODO

### Liveness Probe
TODO

### Readiness Probe
TODO

### Metrics
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

# Get resource usage for pods
kubectl top pods -A
```

### Resource Requests and Limits
TODO
requests: resources needed to run
limits: max resources allowed while running

### Taints and Tolerations

Taints flag a node such that it restricts scheduling of pods onto that node.
We first saw this with our control plane node, which by default didn't allow
pods to be scheduled there. Crucial `kube-system` pods are exempted from taints.

Taints can have three possible effects:

* `NoSchedule`: Prevent the node from scheduling new pods.
* `PreferNoSchedule`: Avoid scheduling on this node, but not outright prevent it.
* `NoExecute`: Not only prevent pods from being scheduled on this node, but evict any already running pods off this node.

The other side of taints are tolerations. Tolerations are defined in the manifest and allow that
manifest to tolerate the taint, and thus be scheduled on the tainted node.

One example of this might be to handle hardware differences of nodes. Say one node has powerful GPU installed
and you prefer to only run GPU heavy pods there. You can set a taint to handle this.

```
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

## kubectl Commands

TODO common and useful commands, links to docs

`kubectl get all`

`kubectl describe deployment.apps -n kube-system coredns`

`kubectl scale deployments.apps -n kube-system coredns --replicas=3 deployment.apps/coredns scaled`

`kubectl edit KIND NAME`

`kubectl logs -n kube-system <podName>`
`kubectl logs -n kube-system <podName> --previous`

`kubectl -n kube-system edit cm <cmName>` Edit config map

`kubectl exec -i -t shell-demo -- /bin/bash` # exec in pod with single container (no need to specify)

`kubectl exec -i -t my-pod --container main-app -- /bin/bash` # selecting container from multi-container pod

`kubectl apply` create/update API; when updating, only need to specify parts changing

`kubectl create` create API; will fail if already exists

`kubectl replace` completely replace existing object; must provide complete manifest; `--force` flag will have it actually perform a `DELETE` before re-creating object with new manifest

to update fields that cannot be updated, delete and re-create the resource with `replace --force`
`kubectl replace -f https://k8s.io/examples/application/nginx/nginx-deployment.yaml --force`

TODO see the difference between file and loaded manifest (and display if the change would not be valid)
`kubectl diff -f manifest.yaml`

`kubectl patch` to update API objects in place

`kubectl delete`

`kubectl scale`

`kubectl autoscale`

## Helm: The Kubernetes Package Manager

TODO

## Helpful Tips

### List All the Things

List objects
```
kubectl api-resources
```

Available API versions
```
kubectl api-versions
```

List built-in roles
```
kubectl get clusterroles
```

Using the `APIGROUP`, you can look up the version from `api-versions` using something like:
```
kubectl explain --api-version=apps/v1 deployments
```

### Autocomplete for kubectl
Enabling tab autocomplete can make it considerably easier to use `kubectl`. To
enable it for Bash:
```
# Enable kubectl autocomplete in bash for current shell
source <(kubectl completion bash)

# Enable kubectl autocomplete system wide
echo "source <(kubectl completion bash)" > /etc/bash_completion.d/kubectl
```

### From Docker Compose to K8s using Kompose

A tool to try to convert docker-compose files to Kubernetes configs: https://kompose.io/

Unlikely that you would want to use the output, but it might help in the process.

### Other Implementations
As Kubernetes is just a set of APIs, you don't technically need `kubeadm` or `kubectl`. You could
just make API calls, or even make your own alternate commands, or recreate the official tools
in your own way. And people have done just that. Here are a few.

* Minikube: Mainly used for learning Kubernetes. Not for production. Many tutorials use this, but it can cause confusion later on as some practices learned in Minikube does not transfer to real Kubernetes.
* KinD: Kubernetes in Docker. Run a Kubernetes cluster in Docker. Not for production.
* MicroK8s: Easier Kubernetes with Sane Defaults.
* K3s: A "Lightweight" Kubernetes, using less memory and having smaller binaries.


## Docker Swarm Comparisons

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

