---
title: Accessing embedded etcd in K3s
date: 2025-08-17
lastmod: 2025-12-30
tags: ["kubernetes"]
author: "Kapil Agrawal"
comments: false
---

[etcd](https://etcd.io) is the de-facto KV store for Kubernetes. [k3s](https://k3s.io) can be run with an embedded etcd as it's KV store which is a great option for running production grade and highly available Kubernetes while keeping overall architecture simple.

There may be situations when you want to interact with it directly in situations like disaster recovery, troubleshooting cluster issues. etcd sync related issues, control leader election, predictability over quorum etc. Accessing the embedded etcd is _trivial_ although k3s docs do not explain how to do so. All you really need is the `etcdctl` binary. As I said, it's _trivial_ 😉

Unlike RKE2, k3s does not provide with `etcdctl` client binary during installation so it needs to be installed separately. Below is a simple shell script which downloads the `etcdctl` binary.

## 1. Download `etcdctl`

Save the following shell script in a file called `get-etcdctl.sh` on the control plane nodes

{{< gist "netops2devops" "ce77206bedca137927c1e1cdbc386a65" >}}

## 2. Run the script

sudo if necessary

```
chmod +x get-etcdctl.sh && ./get-etcdctl.sh
```

## 3. Export etcd connection vars

```sh
export ETCDCTL_ENDPOINTS='https://[::1]:2379'
export ETCDCTL_CACERT='/var/lib/rancher/k3s/server/tls/etcd/server-ca.crt'
export ETCDCTL_CERT='/var/lib/rancher/k3s/server/tls/etcd/server-client.crt'
export ETCDCTL_KEY='/var/lib/rancher/k3s/server/tls/etcd/server-client.key'
```

## 4. Connect to etcd

```sh
./etcdctl member list
./etcdctl endpoint status --cluster -w table
```
