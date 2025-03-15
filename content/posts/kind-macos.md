# Running Kubernetes in Docker (kind) on Macos

I have been using [orbstack](https://orbstack.dev) for over two years now as a drop in replacement for Docker desktop and I have to say it's beyond impressive in terms of performance and feature set. My main motivation to move away from Docker desktop was because it does not support IPv6 on a MacOS. The next thing I tried was Colima but that doesn't support IPv6 either. 

### Orbstack supports IPv6 containers on MacOS
```bash
~
❯ docker context ls
NAME            DESCRIPTION                               DOCKER ENDPOINT                                  ERROR
default         Current DOCKER_HOST based configuration   unix:///var/run/docker.sock
desktop-linux                                             unix:///Users/kagraw/.docker/run/docker.sock
orbstack *      OrbStack                                  unix:///Users/kagraw/.orbstack/run/docker.sock


❯ cat .orbstack/config/docker.json
{
  "ip6tables" : false,
  "ipv6" : true
}
```

#### kind-config.yaml
```YAML
---
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: kind
runtimeConfig:
  "api/alpha": "false"
networking:
  apiServerPort: 6443
  ipFamily: dual
  disableDefaultCNI: true
  # IPv6 prefix must be specified first
  serviceSubnet: fd00:10:96::/112,10.96.0.0/12
nodes:
  - role: control-plane
  - role: worker
  - role: worker 
```

#### IPv6 enabled #KIND cluster
```
# Creates a new cluster
❯ kind create cluster --config ~/.config/kind/kind-config.yaml

# Install Cilium with IPv6 enabled
❯ helm install cilium cilium/cilium --namespace=kube-system --set ipv6.enabled=true
```

#### Delete a #KIND cluster
```
❯ kind delete cluster --name $(kind get clusters)
```
