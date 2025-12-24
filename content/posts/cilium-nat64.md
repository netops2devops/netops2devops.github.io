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

- Create a VM for the NAT46x64Gateway - I am using Ubuntu 24.04.03 LTS with kernel version 6.8.0-83-generic
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
        macaddress: "bc:24:11:ee:19:90"
      accept-ra: false
      addresses:
        - "192.168.64.2/24"
        - "2001:db8:46:64::2/64"
      nameservers:
        addresses:
          - "1.1.1.1"
          - "2606:4700:4700::64"
      routes:
        - to: default
          via: "192.168.2.1"
        - to: default
          via: "2001:db8:46:64::1"
```

To get Cilium up and running as a NAT46x64Gateway simply run the Cilium container image with the following options. Notice that we're running cilium with `enabled-k8s=false`. Also pay special attention to `--devices` flag as it must match the interface name (eth0) from our netplan config above. Traffic entering/leaving this interface will be subject to translation.

```sh
docker run --name cilium-nat64 -itd \
 -v /sys/fs/bpf:/sys/fs/bpf \
 -v /lib/modules:/lib/modules \
  --privileged=true \
  --restart=always \
  --network=host \
"quay.io/cilium/cilium:v.17.7" cilium-agent \
  --enable-ipv4=true \
  --enable-ipv6=true \
  --devices=eth0 \
  --datapath-mode=lb-only \
  --enable-k8s=false \
  --bpf-lb-mode=snat \
  --enable-nat46x64-gateway=true
```

> [!NOTE]
>There was a [breaking change](https://github.com/cilium/cilium/commit/feaf96b4a4804b320c06e498822b777e94ccc9c3) introduced in Cilium v1.18.0 which deprecates the use of `datapath-mode=lb-only` so I reached out to the good folks in #dev-lb on Cilium community slack and the man, the myth, the legend [Daniel Borkman](http://borkmann.ch) was very kind to point me in the right direction. To be honest I wasn't expecting to get my question directly answered by the guy who co-created eBPF amongst many other things in linux networking stack (such as netkit) so it was quite a humbling feeling to say the least 😅

![Alt Text](img/dev-lb-slack.png)

To run **Cilium v1.18.0 or above as a standalone NAT46x64Gateway**, use the following command

```sh
docker run --name cilium-nat64 -itd \
 -v /sys/fs/bpf:/sys/fs/bpf \
 -v /lib/modules:/lib/modules \
 --privileged=true \
 --restart=always \
 --network=host \
"quay.io/cilium/cilium:stable" cilium-agent \
 --enable-ipv4=true \
 --enable-ipv6=true \
 --devices=eth0 \
 --enable-k8s=false \
 --bpf-lb-nat46x64=true \
 --enable-nat46x64-gateway=true  \
 --enable-bpf-masquerade \
 --kube-proxy-replacement=true \
 --datapath-mode=netkit
```

Let's check to see if cilium-agent successfully enabled NAT64 support or not

```sh
root@nat64gw:~# docker exec -it cilium-nat64 cilium-dbg status --verbose | awk "/NAT46\/64/ {found=1} found"
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
```

## Test

A good test to check if our NAT46x64Gateway is performing the 4to6 translation correctly, we can try connecting to an application that is accessible only via IPv4. So let's provision another Ubuntu VM on our IPv6 only network with the following netplan config as shown below.

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    ens18:
      set-name: eth0
      match:
        macaddress: "bc:24:11:55:02:a5"
      accept-ra: false
      addresses:
        - "2001:db8:dead:beef::2/64"
      nameservers:
        addresses:
          - "2606:4700:4700::64"
          - "2001:4860:4860::64"
      routes:
        - to: default
          via: "2001:db8:dead:beef::1"
        - to: "64:ff9b::/96"
          via: "2001:db8:46:64::2/64"
```

Few noteworthy points:

- The two nameservers are DNS64 servers from dns64.cloudflare-dns.com and dns64.dns.google respectively. You can use any dns server which has dns64 capability. When a client queries a DNS64 server for a hostname which only has an A record setup, the dns64 server sends a response containing the corresponding IPv4 address as well as a translated IPv6 address.

Example:

google.com has both an A record and a AAAA record.

```sh
root@testvm# host google.com
google.com has address 142.250.190.78
google.com has IPv6 address 2607:f8b0:4009:803::200e
```

github.com only has an A record but since we're using a DNS64 server we receive a (translated) AAAA record as well.

```sh
root@testvm# host github.com
github.com has address 140.82.113.4
github.com has IPv6 address 64:ff9b::8c52:7104
```

- Static route to `64::ff9b/96` which is a special prefix that is used by IPv4/IPv6 translators as defined in [RFC6502](https://datatracker.ietf.org/doc/html/rfc6052). When the DNS64 server responds with the translated IPv6 address, our ipv6 only test host looks up it's routing table and forwards the packet directly to our NAT46x64Gateway i.e `2001:db8:abcd::2`

> [!Note]
> Using a static route is not a hard requirement. The bottom line is that your router needs to know where to forward IPv6 packets going to `64:ff9b::/96` i.e what the next hop is.

Our moment of truth has finally arrived ⏳

```sh
root@testvm# curl -6 -v github.com
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

Additionally, we can also validate this on our VM that is running the Cilium container.

```sh
root@nat64gw:~# docker exec -it cilium-lb cilium-dbg bpf nat list

TCP IN [64:ff9b::8c52:7104]:80 -> [192.168.64.2]:33388 XLATE_DST [2001:db8:dead:beef::2]:33388 Created=155sec ago NeedsCT=0
TCP OUT [2001:db8:dead:beef::2]:33388 -> [64:ff9b::8c52:7204]:80 XLATE_SRC [192.168.64.2]:33388 Created=155sec ago NeedsCT=0
```

Voila!🍾 this confirms our Cilium based NAT46x64Gateway is up and running!
