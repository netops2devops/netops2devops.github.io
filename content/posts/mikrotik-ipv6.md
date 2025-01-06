---
title: IPv6 prefix delegation on Mikrotik
date: 2024-12-14
tags: ["Networking"]
author: "Kapil Agrawal"
comments: false
---
## RouterOS config

```sh
# Configure interface group
/interface list add comment="UPLINK to ISP" name=WAN
/interface list member add comment="WAN facing interface" interface=ether1 list=WAN

# Enable router-advertisement incoming from the ISP
/ipv6 settings set accept-router-advertisements=yes

# Request an IPv6 prefix over WAN interface; my ISP hands out a /56
/ipv6 dhcp-client add add-default-route=yes interface=ether1 pool-name=delegation pool-prefix-length=56 prefix-hint=::/56 request=address,prefix

# Only accept inbound router-advertisements on the WAN interface
/ipv6 firewall filter add action=drop chain=input icmp-options=134:0-255 in-interface-list=!WAN protocol=icmpv6

# Allow prefix delegatation on WAN interface
/ipv6 firewall filter add action=accept chain=input comment="accept DHCPv6-Client prefix delegation." dst-port=546 protocol=udp src-address=fe80::/10
```
