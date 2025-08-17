---
title: Accessing embedded etcd in K3s
date: 2025-08-17
tags: ["kubernetes"]
author: "Kapil Agrawal"
comments: false
---

**etcd** is the de-facto KV store for Kubernetes. **k3s** can be run with an embedded etcd as it's KV store which is a great option for running production grade and highly available Kubernetes while keeping overall architecture simple.

There may be situations when you want to interact with it directly in situations like disaster recovery, troubleshooting cluster issues. etcd sync related issues, control leader election, predictability over quorum etc. Accessing the embedded **etcd** is quite trivial although k3s docs do not explain how to do so. All you really need is the `etcdctl` binary.

Unlike RKE2, k3s does not provide with `etcdctl` client binary during installation so it needs to be installed separately. Below is a simple shell script which downloads the `etcdctl` binary.

## 1. Download `etcdctl`

Save the following shell script in a file called `get-etcdctl.sh` on the control plane nodes

```sh
#!/usr/bin/env bash
set -euo pipefail

# k3s etcd cert/key paths
K3S_ETCD_CACERT="/var/lib/rancher/k3s/server/tls/etcd/server-ca.crt"
K3S_ETCD_CERT="/var/lib/rancher/k3s/server/tls/etcd/server-client.crt"
K3S_ETCD_KEY="/var/lib/rancher/k3s/server/tls/etcd/server-client.key"

# Check if certs exist
if [[ ! -f "$K3S_ETCD_CACERT" || ! -f "$K3S_ETCD_CERT" || ! -f "$K3S_ETCD_KEY" ]]; then
    echo "❌ This host does not appear to be a k3s control plane node with embedded etcd."
    echo "Missing one or more of:"
    echo "  $K3S_ETCD_CACERT"
    echo "  $K3S_ETCD_CERT"
    echo "  $K3S_ETCD_KEY"
    exit 1
fi

# Get latest etcd version
ETCD_VER=$(curl -s https://api.github.com/repos/etcd-io/etcd/releases/latest \
    | grep tag_name \
    | cut -d '"' -f4)

DOWNLOAD_URL="https://storage.googleapis.com/etcd"
TAR_FILE="/tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz"

echo "📥 Downloading etcdctl ${ETCD_VER}..."
curl -sSL "${DOWNLOAD_URL}/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz" -o "${TAR_FILE}"

echo "📦 Extracting etcdctl to ${PWD}..."
tar xzf "${TAR_FILE}" --strip-components=1 -C "${PWD}" etcd-${ETCD_VER}-linux-amd64/etcdctl
rm -f "${TAR_FILE}"

chmod +x "${PWD}/etcdctl"

# Export k3s etcd environment vars
export ETCDCTL_ENDPOINTS="https://[::1]:2379"
export ETCDCTL_CACERT="$K3S_ETCD_CACERT"
export ETCDCTL_CERT="$K3S_ETCD_CERT"
export ETCDCTL_KEY="$K3S_ETCD_KEY"

echo "✅ etcdctl ${ETCD_VER} is ready in ${PWD}"
echo "Example usage:"
echo "  ./etcdctl endpoint status --write-out=table"
./etcdctl version

```

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
