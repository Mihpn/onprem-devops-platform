# Kubernetes HA Cluster — 3 Control Plane (kubeadm + containerd)

Nodes:

- 192.168.100.105 → k8s-master-1
- 192.168.100.106 → k8s-master-2
- 192.168.100.107 → k8s-master-3

CNI: Calico  
Ingress: ingress-nginx (NodePort)

---

## Step 1 — Prepare Nodes (Run on ALL 3 servers)

### Add hosts

```bash
sudo bash -c 'cat >> /etc/hosts' <<EOF
192.168.100.105 k8s-master-1
192.168.100.106 k8s-master-2
192.168.100.107 k8s-master-3
EOF
```

### Update system

```bash
sudo apt update -y && sudo apt upgrade -y
```

### Disable swap

```bash
sudo swapoff -a
sudo sed -i '/swap.img/s/^/#/' /etc/fstab
```

### Kernel modules

```bash
echo -e "overlay\nbr_netfilter" | sudo tee /etc/modules-load.d/containerd.conf
sudo modprobe overlay
sudo modprobe br_netfilter
```

### Sysctl config

```bash
cat <<EOF | sudo tee /etc/sysctl.d/kubernetes.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system
```

---

## Step 2 — Install containerd (ALL nodes)

```bash
sudo apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmour -o /etc/apt/trusted.gpg.d/docker.gpg
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"

sudo apt update -y
sudo apt install -y containerd.io
```

Configure containerd:

```bash
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml

sudo systemctl restart containerd
sudo systemctl enable containerd
```

---

## Step 3 — Install Kubernetes packages (ALL nodes)

```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update -y
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

---

## Step 4 — Init cluster (ONLY k8s-master-1)

```bash
sudo kubeadm init --control-plane-endpoint "192.168.100.105:6443" --upload-certs
```

Setup kubeconfig:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Install Calico:

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/calico.yaml
```

---

## Step 5 — Join control planes (master-2 & master-3)

Run the join command printed from kubeadm init:

```bash
sudo kubeadm join 192.168.100.105:6443 \
--token YOUR_TOKEN \
--discovery-token-ca-cert-hash YOUR_HASH \
--control-plane \
--certificate-key YOUR_CERT_KEY
```

---

## Step 6 — Install Helm (master-1)

```bash
wget https://get.helm.sh/helm-v3.16.2-linux-amd64.tar.gz
tar xvf helm-v3.16.2-linux-amd64.tar.gz
sudo mv linux-amd64/helm /usr/local/bin/
helm version
```

---

## Step 7 — Install ingress-nginx (NodePort)

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm pull ingress-nginx/ingress-nginx
tar -xzf ingress-nginx-*.tgz
```

Edit values.yaml:

```
type: NodePort
http: 30080
https: 30443
```

Install:

```bash
kubectl create ns ingress-nginx
helm -n ingress-nginx install ingress-nginx -f ingress-nginx/values.yaml ingress-nginx
```

---

## Verification

```bash
kubectl get nodes -o wide
kubectl get pods -n kube-system
kubectl get svc -n ingress-nginx
```

---

## Reset Cluster (if needed)

```bash
sudo kubeadm reset -f
sudo rm -rf /var/lib/etcd
sudo rm -rf /etc/kubernetes/manifests/*
```
