---
title: Network Design Considerations for IPv6-Only Kubernetes Deployments
date: 2025-10-20
tags: ["IPv6", "Cilium", "Kubernetes"]
authors: ["Kapil Agrawal"]
comments: false
series: ["Kubernetes in IPv6 only networks"]
series_order: 3
---


- Control plane components
  - api-Server
    - api objects / api endpoints (namespaces, deployment, pod, service, crd, ingress  etc. )
  - etcd
  - kube-controller-manager
    - the reconciliation loop
  - Labels and Selectors
  - DNS
  - CNI 101

- Subnets and Network Addressing
  - Node CIDR
  - Pod CIDR
  - Service CIDR
  - Ingress CIDR
  - Egress CIDR

- Traffic patterns
  - North-South
    - Ingress
    - Egress

  - East-West
    - Node to Node
    - Pod to Pod
    - Pod to Service

- Common Patterns
  - on-prem
  - public cloud
  
- Cilium networking modes
  - native-routing
  - tunneling

---

# Cluster Networking Requirements

Layer3 network connectivity between all Control-plane nodes (running api-server, scheduler, controller-manager, etcd etc.) and the Worker nodes. None of these subnets should overlap.

| Keyword      | Description                                                 | Use Case                                                                                                                                                                                                                                                                                                                                                                                      |
| ------------ | ----------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Node CIDR    | IP subnet of the control plane or worker node.              | Layer3 network connectivity between all Control-plane components (api-server, scheduler, controller-manager, etcd etc.) and Worker nodes. A cluster can use multiple node CIDRs; for instance, control plane and worker nodes may reside in different subnets.                                                                                                                                |
| Pod CIDR     | IP subnet of the Pods running inside the cluster            | Pod is an atomic unit of compute inside a K8s cluster. Pods tend to be ephemeral. Every Pod in the cluster gets assigned a unique IP address upon it's creation. The `kube-controller-manager` allocates a slice of PodCIDR to each node and the `kubelet` running on the node then assigns an IP address to a pod upon initialization. Although advanced CNIs such as Cilium c               |
| Service CIDR | IP subnet for internal services running within the cluster. | Since pods (and their IP addresses) are ephemeral, when multiple pods (replicas) of the same application need to present a stable IP address to other application and internal cluster components running inside the cluster,  pods need to put themselves behind an internal load balancer (which is referred to as Service object). That stable IP address comes from this subnet pool.<br> |

Application Networking Requirements
---

Once you have a cluster ready its time to deploy an application and make it accessible to the outside world.

| Keyword      | Description                                                                                                    | Use case                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| ------------ | -------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Ingress CIDR | IP address for making an application accessible from outside the cluster.                                      | When traffic originating from outside the cluster needs to connect to an application that is running within the cluster,  the application will expose itself behind an ingress (typically an external load balancer). This is analogous to using a Virtual IP (VIP) on a Load Balancer. <br><br>`[ end-user ] ---> [ ingress-ip ] ---> [ application ]`<br>`[ end-user ] <--- [ ingress-ip ] <--- [ application ]`<br><br>Notice that return traffic is symmetric<br><br>                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| Egress CIDR  | IP subnet used for outbound traffic to provide a stable/unique address (via masquerading) to each application. | When a pod needs to send traffic to a destination that is outside the cluster (i.e not in the PodCIDR or ServiceCIDR range) it will need to egress out of somewhere. Common pattern is to use an egress gateway or use the node address to masquerade.<br><br>If dedicated egress gateway nodes are present the pods use that egress node as it's gateway. The egress gateway nodes masquerade with a stable IP address for the packet to reach the final destination. Return traffic is symmetric. <br><br>In absence of dedicated egress gateway nodes, the cluster can be configured to instead perform masquerading on the node itself (where the pod is running) and use the routable IP address of the node.  This is done by using `nodeselector` when deploying the application which basically pins down the pods to be scheduled on a particular set of nodes.<br><br>The value add in having egress gateway nodes is that application (pods) are no longer pinned to dedicated worker nodes which significantly reduces the number of worker nodes we need to run (we get to run less and do more). The worker nodes then become a general compute environment and pods can be scheduled anywhere as desired by the kube-scheduler. If and when the pods do need to egress outside the cluster, they go via the egress gateway node and the outside world sees traffic coming from a known/predictable address which security teams usually care about. <br> |
|              |                                                                                                                |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |

General notes on egress gateway

- Inside the cluster
  - IP address as NOT an identity of the application. Hostname is NOT an identity of an application.
  - Cilium manages a unique identity (number) for each application based on the set of labels that are assigned to it during deployment
- Outside the cluster
  - Business as usual. Your firewall, IDS, etc. see the mapping 1:1 between application and it's ip address.
