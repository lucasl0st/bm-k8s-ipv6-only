# bm-k8s-ipv6-only
Bare Metal Kubernetes IPv6 Only Setup

# Sytem Setup

Operating System: Arch Linux   
IPv6 Subnet: `fdd3:7046:2ad5:4300::/56`

Control plane has IP Address: `fdd3:7046:2ad5:4300::1/56`   
Node 1 has IP Address: `fdd3:7046:2ad5:4300::2/56`   
Node 2 has IP Address: `fdd3:7046:2ad5:4300::3/56`   

# Network Setup
Control plane Hostname: `k8s-cp.server.lan`   
Node 1 hostname: `k8s-node-1.server.lan`   
Node 2 hostname: `k8s-node-2.server.lan`   

DNS setup: `AAAA` Record `k8s-cp.server.lan` -> `fdd3:7046:2ad5:4300::1`   
DNS setup: `AAAA` Record `k8s-node-1.server.lan` -> `fdd3:7046:2ad5:4300::2`   
DNS setup: `AAAA` Record `k8s-node-2.server.lan` -> `fdd3:7046:2ad5:4300::3`   

# Kubernetes Installation Preparation

add to: `/etc/modules-load.d/k8s.conf`
```
overlay
br_netfilter
```

```bash
modprobe overlay   
modprobe br_netfilter
```

add to: `/etc/sysctl.d/k8s.conf`
```
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv6.conf.all.forwarding=1
```

```bash
sysctl --system
```

```bash
pacman -S containerd

mkdir /etc/containerd   
containerd config default > /etc/containerd/config.toml
```

edit: `/etc/containerd/config.toml`

Change `SystemdCgroup` to true at [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]   

```bash
systemctl enable --now containerd
```

# Kubernetes Control Plane Installation

```bash
pacman -S kubeadm kubelet kubectl

systemctl enable kubelet

kubeadm init \
	--cri-socket=unix:///run/containerd/containerd.sock \
	--control-plane-endpoint=k8s-cp.server.lan \
	--apiserver-advertise-address=fdd3:7046:2ad5:4300::1 \
	--pod-network-cidr=fdd3:7046:2ad5:430a::/64 \
	--service-cidr=fdd3:7046:2ad5:430b::/108
```

edit: `/etc/kubernetes/kubelet.env`
```
KUBELET_ARGS=--node-ip=fdd3:7046:2ad5:4300::1
```

```bash
systemctl restart kubelet
```

# Install Calico

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/tigera-operator.yaml
```

edit: `custom-resources.yaml`
```yaml
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
     # Note: The ipPools section cannot be modified post-install.
    ipPools:
      - blockSize: 122
        cidr: fdd3:7046:2ad5:430a::/64
        encapsulation: None
        natOutgoing: Enabled
        nodeSelector: all()
```

```bash
kubectl create -f custom-resources.yaml
```

# Node Setup

Repeat this for both nodes

```
kubeadm join k8s-cp.server.lan:6443 --token TOKEN \
        --discovery-token-ca-cert-hash sha256:HASH
```

edit: `/etc/kubernetes/kubelet.env` (adjust IP for second node)
```
KUBELET_ARGS=--node-ip=fdd3:7046:2ad5:4300::2
```

```bash
systemctl restart kubelet
```

# Install MetalLB

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.10/config/manifests/metallb-frr.yaml
```

edit: `address-pool.yaml`
```
apiVersion: metallb.io/v1beta1
kind: AddressPool
metadata:
  name: pool
  namespace: metallb-system
spec:
  protocol: layer2
  addresses:
  - fdd3:7046:2ad5:430c::/64
```

```bash
kubectl apply -f address-pool.yaml
```

## Install ingress nginx controller

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.3.0/deploy/static/provider/cloud/deploy.yaml
```