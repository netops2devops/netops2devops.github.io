---
title: Simplest IPv6 only k3s cluster using Cilium
date: 2025-10-24
tags: ["IPv6", "Cilium", "Kubernetes"]
authors: ["Kapil Agrawal"]
comments: false
series: ["Kubernetes in IPv6 only networks"]
series_order: 2
---

Now that our underlying network infrastructure requirements are squared (as described in Part 1) we are ready to install [k3s](https://docs.k3s.io) as our Kubernetes distribution. First we need to prepare our desired k3s config file and override a few default settings. From my experience, *this is where most users get tripped up early on* when setting up Kubernetes on an IPv6 only network. They assume the default configurations will work out of the box with IPv6, but that’s far from true. As mentioned in Part 1 of this series, Kubernetes enables us to build highly composable system through various add-ons (or plugins). That’s why it’s essential for every component that we add to support IPv6. Each add-on comes with its own set of default values which we likely need to override to *make it work* with IPv6. So hang tight and follow along.

![Kubernetes in IPv6 only network using k3s and cilium](img/k3s-ipv6-cilium.png)

## Bootstrap k3s Controller

On the controller1 node

1. Create the k3s directory & a config.yaml file

```sh
mkdir -p /etc/rancher/k3s
touch /etc/rancher/k3s/config.yaml
```

2. Add the following to `config.yaml` file. You may need to update the address bits to match your lab environment.

```yaml
# /etc/rancher/k3s/config.yaml
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
# maximum size supported by k3s is a /108 due to a bitmap limitation
service-cidr: "fd00::/108"
# k3s out of the box ships with whole bunch of addons which we don't need since we will be using Cilium
# consider enabling metrics-server if you want pretty some grafana dashboards for monitoring
disable:
  - traefik
  - servicelb
  - metrics-server
tls-san:
  - "::1"
  - 127.0.0.1
  - "2001:db8::1"
```

Use this one liner command as shown in [official k3s docs](https://docs.k3s.io/quick-start)

```sh
curl -sfL https://get.k3s.io | sh -
```

Before proceeding verify the k3s systemd service is up and running `systemctl status k3s`.

```
root@control1:/home/kagraw# systemctl status k3s
● k3s.service - Lightweight Kubernetes
     Loaded: loaded (/etc/systemd/system/k3s.service; enabled; preset: enabled)
     Active: active (running) since Sat 2025-10-25 00:05:07 UTC; 50min ago
       Docs: https://k3s.io
```

Locate the kubeconfig file under `/etc/rancher/k3s/k3s.yaml`.

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

### Install Cilium

For any CNI a fundamental requirement is to provide pod to pod connectivity within the cluster. This can be done in two ways using Cilium.

1. `Native-Routing` : The native packet forwarding mode leverages the routing capabilities of the network Cilium runs on instead of performing encapsulation.
2. `Encapsulation` : Cilium defaults to using VxLAN based tunnel mechanism to form a mesh of tunnels between all cluster nodes.

Each of these approaches have their own pros and cons some of those are documented in [the official cilium docs](https://docs.cilium.io/en/stable/network/concepts/routing/#configuration) Unless `routing-mode: native` is set, Cilium will default to using tunnel mode. A strong reason for using default tunnel mode is if you want simplicity and low operational overhead to familiarize ourselves with what deploying Kubernetes in IPv6 only environment looks like. It abstracts away most of the complexity that is Kubernetes networking :-) Support for using tunnel mode in IPv6 only networks has been newly introduced in Cilium 1.18.0 which really makes setting up Kubernetes networking in IPv6 only environment a walk in the park! But there are some really strong reasons where native-routing makes more sense which I will cover in a subsequent post in this series.

In its current state our k3s cluster is pretty much useless. We have only bootstrapped k3s on 1 control plane node. Before we can even join a worker node we or deploy a real application to our cluster need to install Cilium CNI. So let's create a file named `values.yaml` with the following

```yaml
---
# kube-proxy-replacement for cilium's eBPF magic
k8sServiceHost: localhost
k8sServicePort: 6444
kubeProxyReplacement: true

# configures cilium to use IPv6-in-IPv6 vxlan tunnel
underlayProtocol: ipv6

ipv4:
  enabled: false
ipv6:
  enabled: true
hubble:
  relay:
    enabled: true
```

Note that we are using `k8sServicePort: 6444` instead of standard port 6443 because of how node registration works in k3s and how it leverages port 6444 for client side load balancer. More in depth explanation can be found in official [k3s docs here](https://docs.k3s.io/architecture#fixed-registration-address-for-agent-nodes) Then install Cilium using helm. We can also use the official [cilium-cli](https://github.com/cilium/cilium-cli) to interface with cilium running on our k3s cluster.

```sh
helm repo add cilium https://helm.cilium.io/ 
helm install cilium cilium/cilium --version 1.18.3 -f values.yaml

# Installation can take 2-3 mins 

❯ cilium status
    /¯¯\
 /¯¯\__/¯¯\    Cilium:             OK
 \__/¯¯\__/    Operator:           OK
 /¯¯\__/¯¯\    Envoy DaemonSet:    OK
 \__/¯¯\__/    Hubble Relay:       OK
    \__/       ClusterMesh:        disabled

```

Our control plane node is fully primed now, and we should be able to add worker nodes to it. Our worker nodes will need an auth token to join the cluster. Make sure to *copy this token file* from control plane node to /tmp on all the worker nodes.

```sh
$ ls /var/lib/rancher/k3s/server/node-token
/var/lib/rancher/k3s/server/node-token
```

## Adding worker nodes

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

3. Finally, install k3s on worker1

```sh
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="agent" sh -s -
```

Verify that the `k3s-agent` systemd service is running and that the worker is able to connect to the kube-api server running on control plane node.

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

All our nodes have joined the cluster and showing status Ready

```
❯ kubectl get nodes
NAME       STATUS   ROLES                       AGE   VERSION
control1   Ready    control-plane,etcd,master   85m   v1.33.5+k3s1
worker1    Ready    worker                      85m   v1.33.5+k3s1
worker2    Ready    worker                      85m   v1.33.5+k3s1

```

cilium-agent and envoy have fully converged

```
homelab ❯ cilium status
    /¯¯\
 /¯¯\__/¯¯\    Cilium:             OK
 \__/¯¯\__/    Operator:           OK
 /¯¯\__/¯¯\    Envoy DaemonSet:    OK
 \__/¯¯\__/    Hubble Relay:       OK
    \__/       ClusterMesh:        disabled

DaemonSet              cilium                   Desired: 3, Ready: 3/3, Available: 3/3
DaemonSet              cilium-envoy             Desired: 3, Ready: 3/3, Available: 3/3
Deployment             cilium-operator          Desired: 2, Ready: 2/2, Available: 2/2
Deployment             hubble-relay             Desired: 1, Ready: 1/1, Available: 1/1
Containers:            cilium                   Running: 3
                       cilium-envoy             Running: 3
                       cilium-operator          Running: 2
                       clustermesh-apiserver
                       hubble-relay             Running: 1
Cluster Pods:          5/5 managed by Cilium
Helm chart version:    1.18.3
```

All pods in our cluster show as `Running`

```
❯ kubectl get po -A -o wide

kube-system  cilium-9p5wk                             1/1    Running  2001:db8::a
kube-system  cilium-envoy-8m9rq                       1/1    Running  2001:db8::a
kube-system  cilium-envoy-hbdhr                       1/1    Running  2001:db8:0:1::a
kube-system  cilium-envoy-lrdv4                       1/1    Running  2001:db8:0:1::b
kube-system  cilium-nn7cl                             1/1    Running  2001:db8:0:1::b
kube-system  cilium-operator-5995cb558b-6zqjh         1/1    Running  2001:db8::a
kube-system  cilium-operator-5995cb558b-hgcr8         1/1    Running  2001:db8:0:1::b
kube-system  cilium-xwb48                             1/1    Running  2001:db8:0:1::a
kube-system  coredns-64fd4b4794-gv8mw                 1/1    Running  fd00::f8
kube-system  hubble-relay-76dfc677f9-rq5vl            1/1    Running  fd00::3e
kube-system  local-path-provisioner-774c6665dc-26rjf  1/1    Running  fd00::62
```

So far so good! Now let's deploy a couple of pods to verify a few things

```
❯ kubectl create ns hello-world
namespace/hello-world created

❯ kubectl create deployment --image rancher/hello-world hello-world --replicas 2 -n hello-world
deployment.apps/hello-world created

❯ kubectl get po -n hello-world
NAME                           READY   STATUS    RESTARTS   AGE
hello-world-5b945cc466-fmmjs   1/1     Running   0          5m54s
hello-world-5b945cc466-jqchw   1/1     Running   0          5m54s

❯ kubectl expose deployment hello-world --port 80 -n hello-world
service/hello-world exposed

❯ kubectl get svc -n hello-world
NAME          TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
hello-world   ClusterIP   fd00::58ed   <none>        80/TCP    3s

❯ kubectl run netshoot -it --image=nicolaka/netshoot -n default -- bash
```

check pod to pod connectivity and pod to ClusterIP (east-west)

```
# ping one of the hello-world pods
netshoot:~# ping fd00::286
PING fd00::286 (fd00::286) 56 data bytes
64 bytes from fd00::286: icmp_seq=1 ttl=63 time=0.328 ms
64 bytes from fd00::286: icmp_seq=2 ttl=63 time=0.362 ms

# pod to ClusterIP service
netshoot:~# curl http://hello-world.hello-world
<html>
  <head>
    <title>Rancher</title>
    <link rel="icon" href="img/favicon.png">
    <style>
      body {
```

check pod to outside world connectivity (north-south)

```
# This example also validates DNS64 and NAT64 cuz github resolves to IPv4 only :(
netshoot:~# curl -v github.com
* Host github.com:80 was resolved.
* IPv6: 64:ff9b::8c52:7103
* IPv4: 140.82.113.3
* Trying [64:ff9b::8c52:7103]:80...
* Connected to github.com (64:ff9b::8c52:7103) port 80

# our pod's gua IPv6 address as seen by the outside world
netshoot:~# curl ifconfig.io
2001:db8:0:1::a
```

The above tests confirm that our IPv6 only k3s cluster is operational from a networking point of view as it has IPv6 connectivity within the cluster as well as outside.

## Observations

In this part 2 of the series we successfully deployed

1. k3s cluster in an IPv6 only network.
2. hello-world app and verified connectivity in and out of the cluster

But before I wrap up this post let's chew on this for a bit. Something to think about

1. Where did those pods get their IPv6 addresses from?
2. Running `kubectl get po -A -o wide` shows pod IPv6 addresses using a mix of GUA and ULA. Why is that?
3. Where is that ULA addressing for the pods coming from? Is it possible to change those a GUA address? What are the implications of doing so?
4. Even if we didn't enable masquerading in our helm values, why is the pod'a address being masqueraded to the node's address when it tries to `curl ifconfig.io` from netshoot pod?
5. How do I expose my application outside the cluster with a routable IPv6 address?
