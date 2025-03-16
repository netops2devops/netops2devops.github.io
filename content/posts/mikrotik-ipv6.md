---
title: IPv6 prefix delegation on Mikrotik
date: 2025-01-01
tags: ["IPv6","Networking"]
author: "Kapil Agrawal"
comments: false
---

In this post I am sharing the relavent config bits that I needed to configure on my Mikrotik router to get an IPv6 block from my ISP using a process called as [prefix-delegation](https://blog.lacnic.net/en/introducing-dhcpv6-prefix-delegation/) The pre-requisite for prefix delegation is that (well, obviously) the ISP must support IPv6 on it's network and your router must be capable of making that request. Now the prefix size that an internet provider hands out varies from provider to provider, unfortunately. For e.g: Comcast is known to hand out a /60 where as ATT provdies a /56. My ISP which is a regional provider also hands out a /56

Here's what my routerOS config looks like 

```bash
# Configure interface group
/interface list add comment="UPLINK to ISP" name=WAN
/interface list member add comment="WAN facing interface" interface=ether1 list=WAN

# Enable router-advertisement incoming from the ISP
/ipv6 settings set accept-router-advertisements=yes

# Request an IPv6 prefix over WAN interface; my ISP hands out a /56
/ipv6 dhcp-client add add-default-route=yes interface=ether1 pool-name=delegation pool-prefix-length=56 prefix-hint=::/56 request=address,prefix

# Allow prefix delegatation on WAN interface
/ipv6 firewall filter add action=accept chain=input comment="accept DHCPv6-Client prefix delegation." dst-port=546 protocol=udp src-address=fe80::/10

# Only accept inbound router-advertisements on the WAN interface
/ipv6 firewall filter add action=drop chain=input icmp-options=134:0-255 in-interface-list=!WAN protocol=icmpv6
```
