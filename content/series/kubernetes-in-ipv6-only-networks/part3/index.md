---
title: "IPv6 Only Kubernetes with Cilium: What to Know Before Production"
date: 2025-11-01
lastmod: 2025-11-01
tags: ["IPv6", "Cilium", "Kubernetes"]
authors: ["Kapil Agrawal"]
comments: false
series: ["Kubernetes in IPv6 only networks"]
series_order: 3
---

In Part 1 of this series we created an IPv6 only underlay network which gave us a solid foundation to build a simple Kubernetes cluster on top. Being a distributed system, Kubernetes comprises several [key components](https://kubernetes.io/docs/concepts/overview/components/) which must be able to talk to one another for a functional cluster. The cluster that we built in Part 2 using k3s and cilium, glued those components together using VxLAN as the overlay. Cilium by default ships with `routingMode: tunnel` (which uses VxLAN) because it is easiest to operate and does a pretty good job of abstracting away most of the complexity. But using the default comes with some trade-offs which has implications for IPv6 networking & security design.

## Trade-offs between native-routing vs. tunneling

Although Cilium’s documentation clearly outlines the [pros and cons](https://docs.cilium.io/en/stable/network/concepts/routing/) of each mode, based on my own experience after having tested all possible configuration options, native routing tends to be the best choice if the goal is to treat IPv6 as a first-class citizen.

### Egress traffic predictability

- In tunnel mode, *egress traffic is masqueraded* to either the node’s default address (where the pod is scheduled) or the egress gateway’s IP, if one exists. Without a dedicated egress gateway, the egress IP depends on the node chosen by the kube-scheduler. You can use a [nodeSelector](https://kubernetes.io/docs/tasks/configure-pod-container/assign-pods-nodes/) for predictability, but this often leads to node sprawl for deployments with multiple pod replicas, unless you’re fine with sharing egress IPs across multiple applications. The complexity grows in production environments where IPs serve as application security identities.

- While IPv6 support for egress gateway only recently became available in Cilium 1.18.0 but during my testing I ran into [this limitation](https://github.com/cilium/cilium/issues/38962) where setting an IPv6 address as egressIP is not currently supported. As of writing the only way to get egress gateway working for with IPv6 is to dual stack egress gateway nodes (although I used a dummy non-routable IPv4 address and it worked 😂) and then use the corresponding egress interface in your `CiliumEgressGatewayPolicy`.

### To masquerade or not to

- End-to-End security observability in Kubernetes can be challenging. Since pod addresses are ephemeral it is difficult to fit cloud native tech with traditional enterprise firewalls or perimeter based IDS (such as Zeek, Suricata) as they treat IP addresses as security identities. These tools have no notion of Kubernetes specific metadata such as pods, services, labels etc. leaving the incident responder confused during investigation as to which application the traffic originated behind a masqueraded address, which makes it difficult to trace alerts back to specific workloads or namespaces.

## Kubernetes IPv6 Design Philosophy

Designing for IPv6 in Kubernetes isn’t just about addressing — it’s about preserving visibility, scalability, and security in a modern, cloud-native network. The following principles shape our approach:

> Route wherever possible, masquerade only when necessary

Network transparency is a key advantage of IPv6. By minimizing NAT, we retain end-to-end visibility and simplify troubleshooting across the cluster.

> Maintain clean ingress and egress paths

Avoiding unnecessary address translation ensures that north–south traffic remains predictable and auditable which is a crucial factor when integrating with enterprise firewalls and IDS solutions.

> Align with existing enterprise security models

Security teams rely on perimeter-based inspection tools like Zeek or Suricata following defense in depth approach. The way we design the cluster network must ensure these tools can continue to monitor north–south flows effectively, allowing security analysts to correlate events and logs with familiar tools.

> Adopt true micro segmentation

Each application be placed in its own /64 IPv6 subnet, following RFC 7421 and RFC 8504. This enables fine-grained policy enforcement and isolates workloads at the network layer. Further fine-grained policies may be applied via Kubernetes network policies if necessary.

Implementing an IPv6 only Kubernetes cluster with these design principles is possible using Cilium with the following approach

- Native Routing
- Multi-Pool IPAM

## Network Address Planning

As mentioned above "Designing for IPv6 in Kubernetes isn’t *just* about addressing", but proper address planning is still a big part of Kubernetes networking :-) especially when it comes to native routing. There are multiple types of networks which are required to build a cluster, and it's paramount to have an address plan for these to integrate the cluster with underlying IPv6 only network.

### Cluster Network Requirements

At a bare minimum we need the following just to get the cluster bootstrapped

| Keyword                    | Use Case                                                                         | Routed | Description                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| -------------------------- | ----------------------------------------------------------------------------------- | ------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| NodeCIDR                   | Underlay addresses for control or worker node interfaces | Yes    | This is the **node address pool** defined in part 1. It provides Layer 3 connectivity between control plane components (API server, scheduler, controller-manager, etcd) and worker nodes. Clusters can use multiple node CIDRs. For example, separating control plane and worker nodes into different subnets.                                                                                                                                                                                                                                        |
| PodCIDR or `PodIPPool`     | Subnets used for allocating address to Pods running inside the cluster. | Yes    | Pods are ephemeral, and each one receives a unique IP address when created. The kube-controller-manager assigns each node a slice of the PodCIDR, and the node’s kubelet allocates IPs to pods. Advanced CNIs like Cilium can manage IPAM directly. You provide a large address pool (e.g., 2001:db8::/56) and a maskSize (e.g., /64), and Cilium automatically divides the range, tracking allocated and available addresses. |
| ServiceCIDR<br>(ClusterIP) | cluster internal only services                                           | No     | Since pods (and their IP addresses) are ephemeral, when multiple pods (replicas) of the same application need to present a stable IP to other applications and internal cluster components, pods need to put themselves behind an internal load balancer aka Service (type ClusterIP). This stable VIP address comes from the k3s `service-cidr` pool. Under the covers Cilium tracks which endpoints (pods) are sitting behind a VIP. This internal cluster subnet does not need to be routed and since it's just a set of floating IPs internally within the cluster, it makes more sense to use ULA for this. ClusterIP is a "virtual" address — it doesn’t exist on a physical NIC!|

### Application Networking Requirements

An IPv6 only cluster sounds great, until you realize no one outside the cluster can actually reach your apps. :-)

| Keyword                          | Use case                                                                                                                                                                                                                                                                                             | Routed | Description                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| -------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Ingress CIDR                     | IP address for making an application accessible from outside the cluster. In Kubernetes lingo, ingress is deprecated and is replaced by `GatewayAPI`. But conceptually they both solve the same problem.                                                                                               | Yes    | When traffic originating from outside needs to connect to an application that is running within the cluster,  the application must expose itself behind a Gateway API object. This is analogous to using a Virtual IP (VIP) on a Load Balancer.  <br>`[end-user]-->[ingress vip]-->[app]`<br>`[end-user]<--[ingress vip]<--[app]`<br> For an external user to access the application via ingress address, the ingress VIP must be routed.  These VIPs can be announced over BGP to the upstream router.  Application's DNS A/AAAA record points to this Ingress VIP.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| Egress CIDR (Optional) | Used for outbound traffic to provide a stable/unique address (via masquerading) to each application. Since we are not using an egress gateway, we don't need to worry about this one, but it is generally a good idea to reserve some address space if we ever need to make changes to our design in the future | Yes    | This is particularly useful when using ULA/RFC1918 addresses for Pod CIDR. When pod traffic needs to reach a destination IP which is located outside the cluster those packets will need to egress out of somewhere. One common pattern is to use an egress gateway. In presence of egress gateway *egress traffic is masqueraded* with a stable IP address for the packet to reach the final destination.|

So if we try to allocate these subnets in our original address block `2001:db8::/48` which we used in part 1 here's what that address plan layout looks like

```sh
2001:db8::/48               # Parent address block
  |-- 2001:db8::/56         # Node Address pool
    \-- 2001:db8::/64         # Control nodes
    \-- 2001:db8:0:1::/64     # Worker nodes 
    \-- 2001:db8:0:2::/64     # reserved
  ...

  |-- 2001:db8:1::/56       # Ingress CIDR / GatewayAPI address block
    \-- 2001:db8:1:1::/64     # Application A
    \-- 2001:db8:1:2::/64     # Application B
  ...

  |-- 2001:db8:a::/56       # PodIPPool/PodCIDR addr block
    \-- 2001:db8:a::/64       # Default pool
    \-- 2001:db8:a:1::/64     # Application A 
    \-- 2001:db8:a:2::/64     # Application B 
  ...

  |-- 2001:db8:b::/56       # (Reserved)

  ...
```

## Traffic Patterns

### East-West

This is the *intra-cluster* traffic between pods, services, and nodes rather than traffic entering or leaving the cluster. These patterns can take several forms. The most common is `service-to-service` traffic, where two microservices communicate over the cluster network using DNS-based service discovery and CNI managed routing. The term "service-to-service traffic" is yet another abstraction which network engineers are already familiar with. Say if Pod A (part of microservice A) wants to talk to Service B, it first queries CoreDNS (usually running as a set of pods inside the cluster) to resolve `service-b.namespace.svc.cluster.local` into a ClusterIP. That DNS lookup itself is Pod-to-DNS traffic. Once the ClusterIP is resolved, the connection from Pod A to Service B is actually routed to one of the endpoints behind Service B’s (Pod B). Pod to Pod traffic can be direct (within the same node) or routed across nodes via the CNI overlay or native routing. Another common pattern is control-plane to node traffic, such as the API server coordinating with kubelets or managing workloads. Together, these patterns define high-volume flows that make up east-west traffic in a Kubernetes cluster. Some of these flows are depicted in the figure below.

![east-west-traffic-patterns](east-west-traffic.png)

### North-South

north-south traffic refers to traffic moving into or out of the cluster, as opposed to east-west traffic that stays internal. This includes several key patterns. The most common is `world-to-ingress`, where external clients or users access applications running in the cluster through an Ingress or Gateway API endpoint, often fronted by a load balancer that terminates TLS and routes requests to the appropriate backend services. Another form is `pod-to-world`, where workloads inside the cluster reach out to external applications, infrastructure, typically subject to egress policies or NAT64 gateway translation. A third pattern occurs during image pulls, where the Container Runtime Interface (CRI) on each node communicates with external container registries to fetch images needed for pods. Together, these north-south flows define how the cluster interfaces with the outside world.

![north-south-traffic-patterns](north-south-traffic.png)

Regardless of whether you choose to run Cilium in native-routing or tunnel mode, you are going to need a Gateway API Operator (aka Ingress controller) installed in your cluster to solve the two most common problems in Kubernetes networking for north-south traffic, i.e How to -

1. Expose an application to the outside world with a stable IPv6 address (since pod addresses are ephemeral)
2. Get user traffic *into* the cluster, so they can access the application

Common solutions/patterns which address those problems are to either -

- Use a commercial load balancer appliance which provide their own Gateway API implementation (such as F5, Citrix etc.)
- Use a cloud provider managed Load Balancer when running Kubernetes in public cloud
- Use BGP to advertise Ingress IP, Pod CIDR

Cilium has a robust BGP control plane implementation which fits really well for this use case.

#### Pod to World over IPv6 (egress)

For IPv6-only networks where PodCIDR is routed using Cilium's native-routing, pods can obtain a global unicast address (GUA) by using [multi-pool](https://docs.cilium.io/en/stable/network/concepts/ipam/multi-pool/) IPAM. While other IPAM modes in Cilium (such as cluster scope, Kubernetes host scope) can also be configured to hand out GUA addresses, they are [not as flexible](https://docs.cilium.io/en/stable/network/concepts/ipam/#address-management) as multi-pool IPAM. Using GUA addresses for pods eliminates the need for a dedicated egress gateway and keeps the egress traffic path clean which in lines with our design philosophy mentioned above. While Cilium's documentation (as of 1.18) indicates multi-pool is still in "Beta", 1.18.x release already flushed out most of the bugs and limitations that existed for multi-pool IPAM to work with IPv6. As of writing, it has reached *near stable* state and is slated to become generally available (GA) in 1.19 release.

## How Pods Get IPv6 Addresses

While the internal details of Cilium’s IPAM implementation are beyond the scope of this series (that could be a separate series in itself 😄), here’s a *general high-level overview* of how the multi-pool IPAM works:

1. You (cluster admin) install Cilium with `ipam.mode: multi-pool`, defining one or more IPv6 PodIPPools for pod addressing.
2. The Cilium Operator calculates, how many nodes the pool can support based on the configured `maskSize`.
3. It then allocates a prefix of that maskSize from the PodIPPool to each node it can accommodate and delegates pod address management to the cilium-agent locally.
4. When a new pod is scheduled on a node, the cilium-agent assigns it an available IPv6 address from one of the node’s allocated subnets.

## Conclusion

In this part of the series we explored key network and security architecture considerations to understand and plan for *before* deploying Kubernetes, while also touching upon some of Cilium's features which can be leveraged to meet those architecture requirements. In the next part of this series we will bring our [Kubernetes IPv6 Design Philosophy](#kubernetes-ipv6-design-philosophy) to fruition by deploying a new k3s cluster in our IPv6 only environment while leveraging Cilium's advanced capabilities such as multi-pool IPAM, native-routing, BGP Load Balancer IPAM, Gateway API.
