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
* Specified as a Global unit running on ```MachineMetadata=role=leader```
* Assumes Etcd2 is running on same node
* Uses port 9090
* Binds to all addresses
* Sets service-cluster-ip-range to 172.22.0.0/16. 

I was thrown for a loop with service-cluster-ip-range. This is different from
your Flannel network. It is used by Kubernetes so service objects you specify
can be routed alongside your pods. The kube-proxy service will dynamically configure
your host's routing tables so services are accessible. I opted for the 172.22/16 private
address space as 10.100/16 may already be used if you work in a large organization.
If you have no idea what [10.100/16 or 172.22/16 means, I wrote a quick post on CIDR notation.](http://blog.michaelhamrah.com/2015/05/networking-basics-understanding-cidr-notation-and-subnets-whats-up-with-16-and-24/)

## kube-controller-manager

* Runs the Controller Manager service
* Runs as a single instance with ```MachineOf=kube-apiserver.service```
* Assumes kube-apiserver is running on the same node on port 9090

## kube-scheduler

* Runs the Scheduler service
* Runs as a single instance with ```MachineOf=kube-apiserver.service```
* Assumes kube-apiserver is running on the same node on port 9090

## kube-kubelet

The kube-kubelet service assumes its not running on a node with
an api server (as the api server is a "core" service while most
pods will be generic worker nodes). It assumes there is an
environment file at ```/etc/leader``` with a ```LEADER_ENDPOINT```
environment variable set. You can write to ```/etc/leader``` in
your cloud-config when you bring up a node.

* Runs the kubelet service
* Runs as a global unit (no restrictions, you may want to change this)
* Assumes there is a DEFAULT_IPV4 environment variable, set from
  a ```setup-network-environment.service``` unit from CoreOS's
  cloud-config.

## kube-proxy

* Like kubelet, assumes the presence of an ```/etc/leader``` environment
  file with a ```LEADER_ENDPOINT``` set.
* Runs as a global unit (you may want to change this)

