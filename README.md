# Fleet Unit Files for Kubernetes on CoreOS

These are a collection of fleet unit files that can be deployed to an existing CoreOS cluster.
They still require a minimum amount of configuration. They come from snippets 
scattered throughout the kubernetes project as well as work from Kelsey Hightower.

*_Why Fleet instead of Cloud-Config?_*

Most of the Kubernetes examples have unit files set within the cloud-config section of 
CoreOS. This doesn't work well in practice. It's difficult to update the cloud-config 
of an entire CoreOS cluster, especially where cloud-drive is not available (i.e AWS). 
Using Fleet for deploying Kubernetes  lets you focus on building a CoreOS cluster first 
(or using an existing one) and then  deploy Kubernetes on top of it. It also lets you 
easily mess around with Kubernetes without having to deal with underlying nodes.

On a side note, if you just want to get up and running to experiment with Kubernetes,
there are a variety of one-click solutions to launch a Kubernetes cluster. For people
experimenting with complex topologies, where you want a variety of heterogenous nodes,
manually deploying Kubernetes on top of an existing CoreOS cluster makes sense. Thus
these unit files.

## Assumptions

* Assumes you have a running CoreOS cluster with etcd 2.0+. 
* Your nodes define `COREOS_PRIVATE_IPV4` in `/etc/environment`. CoreOS should
  do this automatically upon startup.
* Your nodes have a file `/etc/kube-config.env` deployed, which contains the
  expected network configuration. See below for an example.
* Your nodes have a binary `/opt/bin/curl-update-binary` deployed, which
  downloads a file from an URL.

### `kube-config.env`

This defines the network settings for your Kubernetes cluster:

* Where will the master run (and on which port)?
* On which port will Kubelets (and HealthZ/cAdvisor) run?
* Which IP range is to be used by services?
* Where does the [cluster nameserver] run and which domain does it serve?

The `KUBE_MASTER_URLS` can be multiple, comma-separated URLs (beginning with
  http://, **not** ending with a slash). This means you can point it to a
  specific node running the apiserver or preferably to some load balanced
  endpoint of a set of master nodes. (Because of the transient nature of CoreOS
  nodes, minimizing hardcoded IP addresses is a plus.)

Do not confuse `service-cluster-ip-range` (`KUBE_SERVICE_RANGE` in
  `kube-config.env`) with your Flannel network segment! It **must** not be the
  same! The Flannel network segment is used by Kubernetes to give each of your
  PODs an own IP address and allow routing between them.
  `service-cluster-ip-range` on the other hand is used as an allocation pool
  for the Services you create.

Routing for both is setup dynamically by the Kube Proxy (using the information
  stored by Flannel and Kubernetes in your etcd cluster) using the host's
  routing tables and/or iptables NAT chain.

Both can point to arbitrary IP ranges that must not overlap. They should also
  not overlap with the IP range of your physical hosts! In the examples below
  we use `192.168.10.0/24` for the physical hosts, `10.1.1.0/24` for the
  Kubernetes services, while Flannel is setup to use `10.0.0.0/16`.

If you have no idea what [10.100.0.0/16 or 172.22.0.0/16 means, I wrote a quick post on CIDR notation](http://blog.michaelhamrah.com/2015/05/networking-basics-understanding-cidr-notation-and-subnets-whats-up-with-16-and-24/).

You can deploy this file using cloud-config:

```yaml
write-files:
  - path: /etc/kube-config.env
    permissions: '0644'
    content: |
      …
```

[cluster nameserver]: https://github.com/GoogleCloudPlatform/kubernetes/tree/master/cluster/addons/dns

#### Example:

```sh
KUBE_MASTER_URLS=http://192.168.10.24:8080
KUBE_MASTER_PORT=8080
KUBE_KUBELET_PORT=10250
KUBE_HEALTHZ_PORT=10248
KUBE_CADVISOR_PORT=4194
KUBE_SERVICE_RANGE=10.1.1.0/24
KUBE_CLUSTER_NAMESERVER=10.1.1.10
KUBE_CLUSTER_DOMAIN=cluster.local
```

### `curl-update-binary`

The purpose of this file is to download a file from a given URL, updating an
  existing file if the remote one is newer.

You can deploy this file using cloud-config:

```yaml
write-files:
  - path: /opt/bin/curl-update-binary
    permissions: '0755'
    content: |
      …
```

#### Example:

```bash
#!/bin/bash
set -e
file="$1" ; shift
[[ "${file}" ]]
url="$1" ; shift
[[ "${url}" ]]
dir="$(dirname "${file}")"
[ -d "${dir}" ] || mkdir -p "${dir}"
[ -s "${file}" ] || rm -f "${file}"
/usr/bin/curl --fail --location --remote-time --time-cond "${file}" --output "${file}".new "${url}"
if [ -s "${file}".new ] ; then
  mv "${file}".new "${file}"
else
  rm "${file}".new
fi
chmod +x "${file}"
```

## Template Files

Each Kubernetes service is broken out into its own unit file, prefixed with ```kube-``` to
make grouping in Fleet easier. Each unit file 
downloads its respective executable in ```ExecStartPre``` and executes it via
```ExecStart```. There are a few things you'll want to configure in these 
unit files before launching.

If you're running CoreOS in production you probably have nodes assigned 
certain roles. For instance, you probably have a set of "core service" nodes
running etcd while a set of "worker" nodes are running as proxys. The unit
files use ```X-Fleet``` machine metadata to assign certain kubernetes functionality
to specific nodes.

### kube-apiserver

* Runs the Kubernetes API. 
* Specified as a Global unit running on ```MachineMetadata=k8srole=master```
* Assumes Etcd2 is running on same node
* Binds to all addresses

## kube-controller-manager

* Runs the Controller Manager service
* Runs as a single instance with ```MachineOf=kube-apiserver.service```
* Assumes kube-apiserver is running on the same node

## kube-scheduler

* Runs the Scheduler service
* Runs as a single instance with ```MachineOf=kube-apiserver.service```
* Assumes kube-apiserver is running on the same node

## kube-kubelet

The kube-kubelet service assumes its not running on a node with
an api server (as the api server is a "core" service while most
pods will be generic worker nodes).

* Runs the Kubelet service
* Runs as a global unit with `MachineMetadata=k8srole=node`

## kube-proxy

* Runs the Kube Proxy service on top of Flannel
* Runs as a global unit with `MachineMetadata=k8srole=node`

