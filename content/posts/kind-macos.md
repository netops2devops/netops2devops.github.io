---
title: IPv6 enabled Kubernetes cluster on MacOS
date: 2025-03-15
tags: ["Kubernetes", "IPv6", "Cilium"]
author: "Kapil Agrawal"
comments: false
---

If you use MacOS as your daily driver and need an IPv6 capable Kubernetes cluster for local development/testing, this article is meant for you! I have been using [orbstack](https://orbstack.dev) for a few years now as a drop in replacement for Docker desktop and I have to say it's beyond impressive in terms of performance and features. My main motivation to move away from Docker desktop was the lack of support IPv6 on a MacOS and the horrendous performance. The next thing I tried was Colima which is quite flexible but still [lacks IPv6 support on MacOS](https://news.ycombinator.com/item?id=41422931).

While there are multiple kubernetes distributions such as [minikube](https://github.com/kubernetes/minikube/issues/8535), [k3d](https://github.com/k3d-io/k3d/issues/833) which work quite well on a MacOS, neither of those support IPv6 😭 Eventually I ended up trying [KIND](https://kind.sigs.k8s.io) and turns out, it works perfectly fine with IPv6 and is fully customizable 🎉

If you look under the hood, Kubernetes is an API server which relies on composable set of components to build a distributed system. Kubernetes by itself does not handle networking just as it does not handle managing container lifecycle, it relies on a container runtime such as containerd. Similarly, k8s relies on a container network interface [CNI](https://cni.dev) which is the component that manages cluster networking and routing network traffic in and out of the cluster. But to make a Kubernetes cluster IPv6 capable, we need to install a CNI which supports IPv6. Cilium is my CNI of choice as it support IPv6, has a large community around it and is a CNCF graduate project which speaks for it's maturity. I will write more about Cilium and other CNI's in a separate blog post ;)

To run an IPv6 capable Kubernetes cluster on MacOS we need:

- [Orbstack](https://orbstack.dev)
- `brew install kind`
- `brew install helm`
- `brew install kubectl`
- `brew install cilium-cli`

### Orbstack

After installing [orbstack](https://orbstack.dev) make sure your docker context is pointing to Orbstack. You may have to explicitly switch the docker context to orbstack if you have Docker desktop installed. In my case, I uninstalled Docker desktop.

```bash
~
❯ docker context ls
NAME            DESCRIPTION                               DOCKER ENDPOINT                                  ERROR
default         Current DOCKER_HOST based configuration   unix:///var/run/docker.sock
desktop-linux                                             unix:///Users/kagraw/.docker/run/docker.sock
orbstack *      OrbStack                                  unix:///Users/kagraw/.orbstack/run/docker.sock
```

Enable IPv6 in Docker daemon with the following config.

```sh
❯ cat .orbstack/config/docker.json
{
  "ip6tables" : false,
  "ipv6" : true
}
```

### Create a KIND cluster

Create a file named `kind-config.yaml` to pre-configure KIND with the following config options. Notice that I am disabling the default CNI (flannel) so I can install Cilium 😋

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
  serviceSubnet: fd00:10:96::/112,10.96.0.0/16
nodes:
  - role: control-plane
  - role: worker
  - role: worker
```

Bootstrap the cluster.

```bash
# Creates a new cluster with our custom config
❯ kind create cluster --config kind-config.yaml
Creating cluster "kind" ...
 ✓ Ensuring node image (kindest/node:v1.32.2) 🖼
 ✓ Preparing nodes 📦 📦 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing StorageClass 💾
 ✓ Joining worker nodes 🚜
Set kubectl context to "kind-kind"
You can now use your cluster with:

kubectl cluster-info --context kind-kind

Thanks for using kind! 😊
```

Next we install Cilium CNI with IPv6 enabled

```bash
❯ helm install cilium cilium/cilium --namespace=kube-system --set ipv6.enabled=true
```

Optionally, we can enable Hubble as well!

```bash
❯ helm upgrade -n kube-system \
    cilium cilium/cilium \
  --reuse-values --version 1.17.1 \
  --set hubble.relay.enabled=true \
  --set hubble.ui.enabled=true
```

The installation takes about 2-3 mins. We can monitor progress using the following

```sh
❯ cilium status --wait

❯ kubectl get nodes -o wide | awk '{print $1, $2, $3, $5, $6}' | column -t
NAME                STATUS  ROLES          VERSION  INTERNAL-IP
kind-control-plane  Ready   control-plane  v1.32.2  fc00:f853:ccd:e793::2
kind-worker         Ready   worker         v1.32.2  fc00:f853:ccd:e793::3
kind-worker2        Ready   worker         v1.32.2  fc00:f853:ccd:e793::4
```

Now let's try deploying a simple application and verify connectivity over IPv6.

### Deploy and Verify

```bash
❯ kubectl create deployment --image nginx nginx --port 80
deployment.apps/nginx created

❯ kubectl expose deployment nginx --type NodePort
service/nginx exposed

❯ kubectl get svc
NAME         TYPE        CLUSTER-IP         EXTERNAL-IP   PORT(S)        AGE
nginx        NodePort    fd00:10:96::dbc9   <none>        80:30368/TCP   12s
```

External IP shows as none but that is expected since we created a service of type `NodePort` which is exposed over port `30368`. To access a NodePort service we can send a http GET request to any of the cluster nodes over this port.

```bash
❯ curl -6 http://[fc00:f853:ccd:e793::3]:30368
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

Voila!🍾 our application is running on Kubernetes and is accessible over IPv6!

### Conclusion

Running an IPv6 capable Kubernetes cluster is absolutely possible on a MacOS and developers can leverage this setup to develop and test their applications in a IPv6 native environment.
