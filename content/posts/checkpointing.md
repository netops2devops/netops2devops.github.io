---
title: Forensic container checkpointing in Kubernetes
date: 2025-09-14
tags: ["security", "kubernetes"]
authors: ["Kapil Agrawal"]
comments: false
---

## Identify our Pod of interest

Find the node where the pod is currently running

```sh
kubectl get pod -o wide
```

Locate the container id of the Pod

```sh
kubectl desribe pod PODNAME | grep -i "Container ID"
```

## Requirements

1. Download and Install CRIU on the node
   https://criu.org/Packages

2. You may need to explicitly allow access to checkpoint api on the node

```yaml
# kubectl apply -f node-checkpoint-rbac.yaml
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-checkpoint-access
rules:
  - apiGroups: [""]
    resources: ["nodes/checkpoint"]
    verbs: ["create"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: node-checkpoint-access
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: node-checkpoint-access
subjects:
  - kind: Group
    name: system:nodes
    apiGroup: rbac.authorization.k8s.io
```

⚠️ During checkpointing, a .tar archive is created, which requires a functional tar binary. By default, k3s relies on the BusyBox implementation of tar, which is incompatible with CRIU. To ensure checkpointing works correctly, you may need to override this with the system’s full tar binary.

```sh
# /var/lib/rancher/k3s/data/ is k3s’s runtime dependency store, containing unpacked, versioned
# bundles of the k3s binary, containerd, and supporting tools

[root@x86-dev:~] ls -l /var/lib/rancher/k3s/data/current/bin/tar
lrwxrwxrwx 1 root root 7 Sep  6 19:27 tar -> busybox*

[root@x86-dev:~] rm /var/lib/rancher/k3s/data/current/bin/tar
[root@x86-dev:~] ln -s $(which tar) /var/lib/rancher/k3s/data/current/bin/tar
```

## Checkpoint a running pod on a K3s node

```sh
curl -q -s --insecure \
--cert /var/lib/rancher/k3s/agent/client-kubelet.crt \
--key /var/lib/rancher/k3s/agent/client-kubelet.key \
--cacert /var/lib/rancher/k3s/agent/client-ca.crt \
-X POST "https://$(hostname -i):10250/checkpoint/NAMESPACE/PODNAME/CONTAINERNAME"
```

## Example

Try checkpointing a running netshoot pod

```sh
[root@x86-dev:~] curl -q -s --insecure \
--cert /var/lib/rancher/k3s/agent/client-kubelet.crt \
--key /var/lib/rancher/k3s/agent/client-kubelet.key \
--cacert /var/lib/rancher/k3s/agent/client-ca.crt \
-X POST "https://$(hostname -i):10250/checkpoint/default/netshoot/netshoot"
```

Output

```sh
[root@x86-dev:~] {"items":["/var/lib/kubelet/checkpoints/checkpoint-netshoot_default-netshoot-2025-09-13T18:09:15-05:00.tar"]}

[root@x86-dev:~] ls /var/lib/kubelet/checkpoints/
checkpoint-netshoot_default-netshoot-2025-09-13T18:09:15-05:00.tar
```

## Restoring checkpoint image for analysis

Download [checkpointctl](https://github.com/checkpoint-restore/checkpointctl)

```
[root@x86-dev:~] checkpointctl list
Listing checkpoints in path: /var/lib/kubelet/checkpoints/
NAMESPACE   POD        CONTAINER   ENGINE       TIME CHECKPOINTED     CHECKPOINT NAME
---------   ---        ---------   ------       -----------------     ---------------
default     netshoot   netshoot    containerd   13 Sep 25 18:09 CDT   checkpoint-netshoot_default-netshoot-2025-09-13T18:09:15-05:00.tar

```

Inspecting a checkpoint image

```sh
[root@x86-dev:~] checkpointctl inspect \
--files \
--metadata \
--mounts \
--ps-tree \
--ps-tree-cmd \
--ps-tree-env \
--sockets \
checkpoint-netshoot_default-netshoot-2025-09-13T18:09:15-05:00.tar
```

Show memory dump

```sh
# kubelet stores checkpoint under /var/lib/kubelet/checkpoints/
[root@x86-dev:~] checkpointctl memparse <PATH-TO-CHECKPOINT-TAR>

# show full memory dump of a process
[root@x86-dev:~] checkpointctl memparse --pid PID  <PATH-TO-CHECKPOINT-TAR>
```

### Reference

--

- https://kubernetes.io/docs/reference/node/kubelet-checkpoint-api/
- https://criu.org/Containerd
- https://github.com/checkpoint-restore
