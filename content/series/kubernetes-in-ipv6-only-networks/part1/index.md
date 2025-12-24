---
title: IPv6 infrastructure before deploying Kubernetes
date: 2025-10-22
tags: ["IPv6", "Cilium", "Kubernetes"]
authors: ["Kapil Agrawal"]
comments: false
series: ["Kubernetes in IPv6 only networks"]
series_order: 1
---

IPv6 support in vanilla Kubernetes reached stable status in v1.23, yet running IPv6-only clusters still feels like a niche pursuit shrouded in wizardry and arcane magic. But vanilla Kubernetes is borderline useless out of the box. Due to its flexible nature, Kubernetes lets us build highly composable distributed systems. The big selling point behind Kubernetes is you get to manage (aka. orchestrate) full lifecycle of containerized applications. But to run those applications a Kubernetes cluster needs a few add-ons (plugins so to speak).

Heck, you need a plugin to even build the most basic cluster i.e a Container Network Interface (CNI). It's a plugin which not only glues all the cluster components together but also provides networking and security capabilities for the applications running in the cluster. Choosing a CNI is one of the most important decisions you have to make early on.

In this blog series, I will demonstrate how to configure and deploy Kubernetes in an IPv6 only environment from the ground up using battle tested, production grade components such as [k3s](https://k3s.io) and [Cilium](https://docs.cilium.io/en/stable/) and all the network design considerations that needs to go in it.

## Setting up an IPv6 lab environment

The most basic lab scenario to create a Kubernetes cluster contains 1 control plane and 2 worker nodes. We can certainly increase the control plane count to 3 nodes for high availability later on but for now, 1 controller is good enough to get started.

- 3 VMs running Ubuntu 24.04 LTS
- IPv6 address plan
- DNS64 and NAT64
- BGP Router (FRR, VyOS either work just fine)

![IPv6 only lab network topology](k8s-ipv6-infra.png)

### IPv6 Address Planning

Let's assume `2001:db8::/48` as the parent block which we will break down into smaller chunks of /56 prefixes. I have found this online [subnetting calculator](https://www.cidr.eu/en/calculator) to be of great help for address planning. A `/48` gives us `2^8 = 256 x /56` prefixes and each `/56` gives us `256 x /64` subnets. We want to stick to best practices outlined in RFC 7421, RFC 8504 hence we are going to treat /64 as the network boundary. We will be using different non overlapping /64 subnets for addressing our nodes, pods, services, gateway/ingress addresses etc. though our most immediate need is an address block for our cluster nodes. Let's allocate the first /56 prefix i.e `2001:db8::/56` to address all our cluster nodes.

While there is no such hard rule, it is a good practice to keep control plane nodes and worker nodes contained in their own respective Layer3 boundaries which provides some degree of network isolation as well as logical separation to keep the overall design clean.

In part 3 of this series I will cover more about address planning for various Kubernetes components (pods, services, ingress, egress etc.), but the following is the bare minimum need to get underlying infrastructure setup

```sh
2001:db8::/48               # parent block
  |-- 2001:db8::/56         # node address pool
    \-- 2001:db8::/64       # control nodes
    \-- 2001:db8:0:1::/64   # worker nodes 
    \-- 2001:db8:0:2::/64   # reserved
    ...
```

### DNS64 and NAT64

If you don't have a NAT64 gateway configured in your IPv6 only network, you are going to need one to pull packages, container images from registries (such as ghcr.io) which only work from the legacy IPv4 only network. I wrote a blog post on how to run [Cilium as a standalone NAT46x64Gateway](https://netops2devops.net/posts/cilium-nat64/) which can be used here.

Both Cloudflare and Google offer free DNS64 service. Either of those can be used.

```sh
~
❯ host dns64.cloudflare-dns.com
dns64.cloudflare-dns.com has IPv6 address 2606:4700:4700::64
dns64.cloudflare-dns.com has IPv6 address 2606:4700:4700::6400

~
❯ host dns64.dns.google
dns64.dns.google has IPv6 address 2001:4860:4860::6464
dns64.dns.google has IPv6 address 2001:4860:4860::64
```

### Node Network Configuration

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
        - "2001:db8::a/64"
      nameservers:
        addresses:
        - "2606:4700:4700::64"   # Cloudflare DNS64 service
      routes:
      - to: default
        via: "2001:db8::1"        # Router (default gateway)
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
        - "2001:db8:0:1::a/64"
      nameservers:
        addresses:
        - "2606:4700:4700::64"   # Cloudflare DNS64 service
      routes:
      - to: default
        via: "2001:db8:0:1::1"    # Router (default gateway)
```

Configuration for worker2 will be similar to worker1. Same subnet but different mac address and ipv6 address bits.

### Verify Connectivity

Configure the corresponding IPv6 addressing on the router side as well. Once the router side is configured, we should be able to confirm -

1. Layer3 connectivity between the controller and worker nodes using ping
2. Connectivity to `github.com` and `ghcr.io` via DNS64 and NAT64 on all the nodes

```
# controller's ipv6 address as seen by the world
control1:~$ curl ifconfig.io
2001:db8::a

# also validates dns64 and nat64 are working as expected
control1:~$ curl -v github.com
* Host github.com:80 was resolved.
* IPv6: 64:ff9b::8c52:7103
* IPv4: 140.82.113.3
*   Trying [64:ff9b::8c52:7103]:80...
* Connected to github.com (64:ff9b::8c52:7103) port 80

# Layer3 reachability between the nodes
control1:~$ ping6 2001:db8:0:1::a
PING 2001:db8:0:1::a (2001:db8:0:1::a) 56 data bytes
64 bytes from 2001:db8:0:1::a icmp_seq=1 ttl=63 time=0.476 ms
64 bytes from 2001:db8:0:1::a icmp_seq=2 ttl=63 time=0.645 ms
```

## Conclusion

While setting up DNS64/NAT64 may have taken some initial effort, it has freed us from the constraints of the legacy IPv4 protocol without sacrificing compatibility with the parts of the internet that still rely on it. Think of it as an investment that will continue to pay off over time, especially when you realize you no longer need to:

1. Maintain and operate two parallel networks in a dual-stack configuration.
2. Worry about IPv4 address exhaustion ever again.

Even better, your internal infrastructure can now be designed around IPv6-only networking. For external accessibility, you can simply dual-stack your ingress or external-facing load balancer to handle translation between a dual-stack virtual IP (VIP) and your IPv6-only backends thus giving you the best of both worlds.

![bernie-wants-you-to-run-ipv6only](bernie-ipv6only-meme.jpg)

Now that the required underlying IPv6 infrastructure is ready to go, in the next part of this series we will deploy Kubernetes using k3s and cilium in this IPv6 only environment.
