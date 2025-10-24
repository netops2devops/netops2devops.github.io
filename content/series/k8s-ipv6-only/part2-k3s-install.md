---
title: Simplest IPv6 only Kubernetes cluster using k3s and Cilium
date: 2025-10-20
tags: ["IPv6", "Cilium", "Kubernetes"]
authors: ["Kapil Agrawal"]
comments: false
series: ["Kubernetes in IPv6 only networks"]
series_order: 2
---

Assuming the underlying network and infrastructure setup is complete as described in Part 1 we are now ready to install [k3s](https://docs.k3s.io) as our Kubernetes distribution. Before starting the installation, we need to prepare the k3s configuration file and override a few default settings. From my experience, this is where most users get tripped up early on when setting up Kubernetes on an IPv6 only network. They assume all the default configurations will work seamlessly with IPv6, but that’s far from true. As mentioned in Part 1 of this series, Kubernetes enables us to build highly composable system through various add-ons (or plugins). That’s why it’s essential for every component that we add to support IPv6. Each add-on component comes with its own set of default values which we will likely need to override to make it work with IPv6.

## Staging k3s config file

**On Controller node**

1. Create the k3s directory & a config.yaml file on our **control1** node

```sh
mkdir -p /etc/rancher/k3s
touch /etc/rancher/k3s/config.yaml
```

2. Add the following to `config.yaml` file. You may need to update the address bits to match your lab environment.

```yaml
---
# initializes k3s controller with embedded etcd
cluster-init: true
write-kubeconfig-mode: "0644"
# k3s ships with flannel as the default CNI plugin. 
# Since we will be using Cilium we need to disable this.
flannel-backend: none
disable-network-policy: true
# kube-proxy is a network proxy that runs on each node which uses iptables/nftables to
# route any traffic directed to a service's ClusterIP to the appropriate backend Pods.
# we will be using Cilium's eBPF based kube-proxy-replacement hence we disable this 
disable-kube-proxy: true
# subnet for ClusterIP services. Since these are floating addresses, 
# this prefix does not need to be routed
# maximum size supported by k3s is a /108
service-cidr: "fd00::/108"
# k3s out of the box ships with whole bunch of addons which we don't need
disable:
  - traefik
  - servicelb
  - metrics-server
tls-san:
  - "::1"
  - 127.0.0.1
  - "2001:db8::1"
```

## Installing k3s on control plane node

Use this one liner command as shown in [official k3s docs](https://docs.k3s.io/quick-start)

```sh
curl -sfL https://get.k3s.io | sh -
```

Before proceeding verify the k3s systemd service is up and running `systemctl status k3s`. The one liner above should have also generated a kubeconfig file under `/etc/rancher/k3s/k3s.yaml`.

```yaml
---
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUJlRENDQVIyZ0F3SUJBZ0lCQURBS0JnZ3Foa2pPUFFRREFqQWpNU0V3SHdZ.. snipped ..
    server: https://[::1]:6443
  name: default
contexts:
- context:
    cluster: default

.. snipped ..
```

Let's copy the `k3s.yaml` file over to our local workstation and update the server address from `server: https://[::1]:6443` to `server: https://[2001:db8::1]:6443`. Now we should be able to administer the k3s cluster using `kubectl` & `helm` directly from our workstation instead of running any commands on the control node over SSH.

### Installing Cilium

In its current state our k3s cluster is pretty much useless. We have only bootstrapped k3s on 1 control plane node. Before we can even join a worker node we or deploy a real application to our cluster need to install Cilium CNI.

Create a file named `values.yaml` with the following key value pairs.

```yaml
---
k8sServiceHost: localhost
k8sServicePort: 6444
kubeProxyReplacement: true
ipv4:
  enabled: false
ipv6:
  enabled: true
hubble:
  relay:
    enabled: true
underlayProtocol: ipv6
```

I used `k8sServicePort: 6444` instead of 6443 because of how node registration works in k3s and how it leverages port 6444 for client side load balancer. More in depth explanation can be found [here](https://docs.k3s.io/architecture#fixed-registration-address-for-agent-nodes)

Then install Cilium using helm. We can also use the official [cilium-cli](https://github.com/cilium/cilium-cli) to interface with cilium running on our k3s cluster.

```sh
helm repo add cilium https://helm.cilium.io/ 
helm install cilium cilium/cilium --version 1.18.2 -f values.yaml

# Installation can take 2-3 mins 

❯ cilium status
    /¯¯\
 /¯¯\__/¯¯\    Cilium:             OK
 \__/¯¯\__/    Operator:           OK
 /¯¯\__/¯¯\    Envoy DaemonSet:    OK
 \__/¯¯\__/    Hubble Relay:       OK
    \__/       ClusterMesh:        disabled

```

Our control plane node is fully primed now, and we should be able to add worker nodes to it. Our worker nodes will need this token to authenticate with the control plane node before they can join the cluster. Make sure to *copy this token file* from control plane node to /tmp on all the worker nodes.

```sh
$ ls /var/lib/rancher/k3s/server/node-token
/var/lib/rancher/k3s/server/node-token
```

## Installing k3s on worker nodes

1. Create the k3s directory & a config.yaml file on our **worker1** node just how we did it on control1 node above.

```sh
mkdir -p /etc/rancher/k3s
touch /etc/rancher/k3s/config.yaml
```

2. Add the following lines to our `config.yaml`

```yaml
---
token-file: /tmp/node-token
server: https://[2001:db8::1]:6443
```

3. Use this one liner command

```sh
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="agent" sh -s -
```

Verify that the `k3s-agent` systemd service is running and that the worker is able to connect to the kube api server running on control plane node.

```sh
root@worker1:# systemctl status k3s-agent
● k3s-agent.service - Lightweight Kubernetes
     Loaded: loaded (/etc/systemd/system/k3s-agent.service; enabled; preset: enabled)
     Active: active (running) since Thu 2025-10-16 23:14:49 CDT; 2 days ago

root@worker1:# curl localhost:6444
Client sent an HTTP request to an HTTPS server.
```

Repeat the same process for all the other worker nodes.

## Verify Connectivity
