---
title: IPv6 infrastructure before deploying Kubernetes
date: 2025-10-20
tags: ["IPv6", "Cilium", "Kubernetes"]
authors: ["Kapil Agrawal"]
comments: false
series: ["Kubernetes in IPv6 only networks"]
series_order: 1
weight: 10
---

IPv6 support in vanilla Kubernetes reached stable status in v1.23, yet running IPv6-only clusters still feels like a niche pursuit shrouded in wizardry and arcane magic. But vanilla Kubernetes is borderline useless out of the box. Due to its flexible nature, Kubernetes lets us build highly composable distributed systems. The big selling point behind Kubernetes is you get to manage (aka. orchestrate) full lifecycle of containerized applications. But to run those applications a Kubernetes cluster needs a few add-ons (plugins so to speak).

Heck, you need a plugin to even build the most basic cluster i.e a Container Network Interface (CNI). It's a plugin which not only glues all the cluster components together but also provides networking and security capabilities for the applications running in the cluster. Choosing a CNI is one of the most important decisions you have to make early on.

In this series, I will demonstrate how to configure and deploy Kubernetes in an IPv6 only environment from the ground up using battle tested, production grade components such as [k3s](https://k3s.io) and [Cilium](https://docs.cilium.io/en/stable/) which work in an IPv6 only environment and all the design considerations that go into it.

## Setting up an IPv6 lab environment

The most basic lab scenario to create a Kubernetes cluster contains 1 control plane and 2 worker nodes. We can certainly increase the control plane count to 3 nodes for high availability later on but for now, 1 controller is good enough to get started.

- 3 VMs running Ubuntu 24.04 LTS
- IPv6 address plan
- DNS64 and NAT64
- BGP Router (FRR, VyOS either work just fine)

![IPv6 only lab network topology](img/k8s-ipv6-infra.png)

### Address planning

Let's assume `2001:db8::/48` as the parent block which we will break down into smaller chunks of /56 prefixes. I have found this online [subnetting calculator](https://www.cidr.eu/en/calculator) to be of great help for address planning. A `/48` gives us `2^8 = 256 x /56` prefixes and each `/56` gives us `256 x /64` subnets. We want to stick to best practices outlined in RFC 7421, RFC 8504 hence we are going to treat /64 as the network boundary. We will be using different non overlapping /64 subnets for addressing our nodes, pods, services, gateway/ingress addresses etc. though our most immediate need is an address block for our cluster nodes. Let's allocate the first /56 prefix i.e `2001:db8::/56` to address all our cluster nodes.

While there is no such hard rule, it is a good practice to keep control plane nodes and worker nodes contained in their own respective Layer3 boundaries which provides some degree of network isolation and logical separation which helps keep the overall network design clean.

```sh
2001:db8::/56           # node addressing pool
|
|-- 2001:db8::/64       # control nodes
|-- 2001:db8:0:1::/64   # worker nodes 
|-- 2001:db8:0:2::/64   # reserved

..snipped..
```

### DNS64 and NAT64

If you don't have a NAT64 gateway configured in your IPv6 only network, you are going to need one to pull packages, container images from registries (such as ghcr.io) which only work from the legacy IPv4 only network. I wrote a blog post on how to run [Cilium as a standalone NAT46x64Gateway](https://netops2devops.net/posts/cilium-nat64/) which can be used here.

Both Cloudflare and Google offer free DNS64 service. Either of those can be used.

```
~
❯ host dns64.cloudflare-dns.com
dns64.cloudflare-dns.com has IPv6 address 2606:4700:4700::64
dns64.cloudflare-dns.com has IPv6 address 2606:4700:4700::6400

~
❯ host dns64.dns.google
dns64.dns.google has IPv6 address 2001:4860:4860::6464
dns64.dns.google has IPv6 address 2001:4860:4860::64
```

### Node network configuration

Once the Ubuntu VMs are provisioned, we will have to configure netplan on the nodes as shown below. Take note of different /64 networks that control plane vs. worker nodes are in and also note cloudflare DNS64 server in use. My Router is configured to route packets destined to `64:ff9b::/96` as per [RFC 6052](https://datatracker.ietf.org/doc/html/rfc6052) towards the NAT64 gateway.

**On controller1**

```yaml
# /etc/netplan/01-netcfg.yaml
---
network:
  version: 2
  renderer: networkd
  ethernets:
    ens18:
      dhcp4: false
      dhcp6: false
      set-name: eth0
      match:
        macaddress: bc:24:11:7f:9a:6d
      accept-ra: false
      addresses:
        - "2001:db8::1/64"
      nameservers:
        addresses:
        - "2606:4700:4700::64"   # Cloudflare DNS64 service
      routes:
      - to: default
        via: "2001:db8::"        # Router (default gateway)
```

**On worker1**

```yaml
# /etc/netplan/01-netcfg.yaml
---
network:
  version: 2
  renderer: networkd
  ethernets:
    ens18:
      dhcp4: false
      dhcp6: false
      set-name: eth0
      match:
        macaddress: bc:24:11:7f:9b:6e
      accept-ra: false
      addresses:
        - "2001:db8:0:1::1/64"
      nameservers:
        addresses:
        - "2606:4700:4700::64"   # Cloudflare DNS64 service
      routes:
      - to: default
        via: "2001:db8:0:1::"    # Router (default gateway)
```

Configuration for worker2 will be similar to worker1. Same subnet but different mac address and ipv6 address bits.

#### Verify

Configure the corresponding IPv6 addressing on the router side as well. Once the router side is configured, we should be able to confirm -

1. Layer3 connectivity between the controller and worker nodes
2. Connectivity to `github.com` and `ghcr.io` via DNS64 and NAT64 on all the nodes
