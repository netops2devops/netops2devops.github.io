---
title: Using Cilium as a standalone NAT46x64Gateway
date: 2025-05-01
tags: ["IPv6", "Cilium", "eBPF"]
authors: ["Kapil Agrawal"]
comments: false
---

IPv4 and IPv6 are two different protocols they are incompatible with one another i.e a host with IPv4 address cannot connect to an application that is only being served over IPv6 and vice-versa. This is where a v4 to v6 translator comes into play. Having a NAT46x64Gateway on the local network simplifies the transition from IPv4 to IPv6 while providing seamless connectivity to applications.

# Cilium

In this blog post I cover how I am running [Cilium](https://cilium.io) as a standalone NAT46x64 gateway which harnesses the power of [eBPF](https://docs.ebpf.io) in the linux kernel. Cilium is a CNCF graduate project which brings advanced networking capabilities to Kubernetes. While it is most commonly used as a CNI (Container Network Interface), not much has been documented about it's capabilities outside of Kubernetes, specially as a standalone NAT46x64 gateway. For my fellow kernel nerds and C lovers, Cilium's NAT46x64 implementation can be found [here](https://github.com/cilium/cilium/blob/main/bpf/lib/nat_46x64.h)

![Alt Text](img/cilium-nat64.png)

## Configuration

Before we get to the real meat and potatoes we need to do some prep work.

- Create a VM for the NAT46x64Gateway - I am using Ubuntu 22.04LTS with kernel version 5.15.0-138-generic in my setup.
- [Install Docker](https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository) on the VM
- Configure networking - A `NAT46x64Gateway` must be dual stacked as it acts as a bridge between IPv4 and IPv6 networks. Here's an example netplan config I am using.

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    ens18:
      set-name: eth0
      match:
        macaddress: bc:24:11:ee:19:90
      accept-ra: false
      addresses:
        - 192.168.2.10/24
        - 2001:db8:abcd::2/64
      nameservers:
        addresses:
          - 1.1.1.1
          - 2606:4700:4700::64
      routes:
        - to: default
          via: 192.168.2.1
        - to: default
          via: 2001:db8:abcd::1/64
```

To get Cilium up and running as a NAT46x64Gateway simply run the Cilium container image with the following options. Notice that we're running cilium with `enabled-k8s=false`. Also pay special attention to `--devices` flag as it must match the interface name (eth0) from our netplan config above. Traffic entering/leaving this interface will be subject to translation.

```sh
docker run --name cilium-lb -itd \
	-v /sys/fs/bpf:/sys/fs/bpf \
	-v /lib/modules:/lib/modules \
	--privileged=true \
	--restart=always \
	--network=host \
	"quay.io/cilium/cilium:stable" cilium-agent --enable-ipv4=true --enable-ipv6=true --devices=eth0 --datapath-mode=lb-only --enable-k8s=false --bpf-lb-mode=snat --enable-nat46x64-gateway=true
```

To check the status of our standalone cilium install with NAT64 enabled

```sh
root@nat64gw:~# docker exec -it cilium-lb cilium status --verbose | awk "/NAT46\/64/ {found=1} found"
  NAT46/64 Support:
  - Services:         Enabled
  - Gateway:          Enabled
    Prefixes:         64:ff9b::/96
  XDP Acceleration:   Disabled
  Services:
  - ClusterIP:      Enabled
  - NodePort:       Enabled (Range: 30000-32767)
  - LoadBalancer:   Enabled
  - externalIPs:    Enabled
  - HostPort:       Disabled
... snipped ...
```

## Test

A good test to check if our NAT46x64Gateway is performing the 4to6 translation correctly, we can try connecting to an application that is accessible only via IPv4. So let's provision another Ubuntu VM on our IPv6 only network with the following netplan config as shown below.

```yaml
root@testvm# cat /etc/netplan/00-installer-config.yaml
network:
    version: 2
    renderer: networkd
    ethernets:
      ens18:
        set-name: eth0
        match:
          macaddress: bc:24:11:55:02:a5
        accept-ra: false
        addresses:
          - 2001:db8:dead:beef::2/64
        nameservers:
          addresses:
            - 2606:4700:4700::64
            - 2001:4860:4860::64
        routes:
          - to: default
            via: 2001:db8:dead:beef::1
          - to: 64:ff9b::/96
            via: 2001:db8:abcd::2
```

Few noteworthy points:

- The two nameservers are DNS64 servers from dns64.cloudflare-dns.com and dns64.dns.google respectively. You can use any dns server which has dns64 capability. When a client queries a DNS64 server for a hostname which only has an A record setup, the dns64 server sends a response containing the corresponding IPv4 address as well as a translated IPv6 address.

Example:

google.com has both an A record and a AAAA record.

```sh
root@testvm:/home/kagraw# host google.com
google.com has address 142.250.190.78
google.com has IPv6 address 2607:f8b0:4009:803::200e
```

github.com only has an A record but since we're using a DNS64 server we receive a (translated) AAAA record as well.

```sh
root@testvm:/home/kagraw# host github.com
github.com has address 140.82.113.4
github.com has IPv6 address 64:ff9b::8c52:7104
```

- Static route to `64::ff9b/96` which is a special prefix that is used by IPv4/IPv6 translators as defined in [RFC6502](https://datatracker.ietf.org/doc/html/rfc6052). When the DNS64 server responds with the translated IPv6 address, our ipv6 only test host looks up it's routing table and forwards the packet directly to our NAT46x64Gateway i.e `2001:db8:abcd::2`

{{< alert >}}
**Note!** Using a static route is not a hard requirement. The bottom line is that your router needs to know where to forward IPv6 packets going to `64:ff9b::/96` i.e what the next hop is.
{{< /alert >}}

Our moment of truth has finally arrived ⏳

```sh
root@testvm:/home/kagraw# curl -6 -v github.com
*   Trying 64:ff9b::8c52:7104:80...
* Connected to github.com (64:ff9b::8c52:7104) port 80 (#0)
> GET / HTTP/1.1
> Host: github.com
> User-Agent: curl/7.81.0
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 301 Moved Permanently
< Content-Length: 0
< Location: https://github.com/
<
* Connection #0 to host github.com left intact
```

Voila!🍾 our Cilium based NAT46x64Gateway is up and running!
