# Create-K8s-Cluster

## 🖥️ Thông Tin Tài Nguyên Cụm K8s

| Hostname       | OS           | IP              | RAM    | CPU |
|----------------|--------------|------------------|--------|-----|
| master-node-1  | Ubuntu 22.04 | 192.168.154.130 | 3 GB   | 2   |
| worker-node-1  | Ubuntu 22.04 | 192.168.154.131 | 3 GB   | 2   |
| worker-node-2  | Ubuntu 22.04 | 192.168.154.132 | 3 GB   | 2   |

---

## 🛠️ Các Bước Cài Đặt Chung

### 1. Cập nhật hệ thống

```bash
sudo apt update -y && sudo apt upgrade -y
```

### 2. Thêm vào file `/etc/hosts`

```bash
sudo vi /etc/hosts
```

Thêm dòng sau:

```
192.168.154.130 master-node-1
192.168.154.131 worker-node-1
192.168.154.132 worker-node-2
```

### 3. Tắt Swap

```bash
sudo swapoff -a
sudo sed -i '/swap.img/s/^/#/' /etc/fstab
```

### 4. Kích hoạt các kernel module cần thiết

```bash
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

### 5. Cấu hình sysctl

```bash
cat <<EOF | sudo tee /etc/sysctl.d/kubernetes.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

### 6. Cài đặt containerd

```bash
sudo apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmour -o /etc/apt/trusted.gpg.d/docker.gpg
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu \$(lsb_release -cs) stable"
sudo apt update -y
sudo apt install -y containerd.io

containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd
```

### 7. Cài đặt Kubernetes (Kubelet, Kubeadm, Kubectl)

```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update -y
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

---

## 🔁 Reset cụm nếu cần

```bash
sudo kubeadm reset -f
sudo rm -rf /var/lib/etcd
sudo rm -rf /etc/kubernetes/manifests/*
```

---

## 🚀 Mô Hình Triển Khai

### Mô Hình 1: 1 Master - 2 Worker

**Thực hiện trên `master-node-1`:**

```bash
sudo kubeadm init
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/calico.yaml
```

**Thực hiện trên các Worker Node:**

```bash
sudo kubeadm join 192.168.154.130:6443 --token <your_token> --discovery-token-ca-cert-hash <your_sha>
```

---

### Mô Hình 2: 3 Master Node

**Thực hiện trên `master-node-1`:**

```bash
sudo kubeadm init --control-plane-endpoint "192.168.154.130:6443" --upload-certs
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/calico.yaml
```

**Thực hiện trên `master-node-2` và `master-node-3`:**

```bash
sudo kubeadm join 192.168.154.130:6443 \
  --token <your_token> \
  --discovery-token-ca-cert-hash <your_sha> \
  --control-plane \
  --certificate-key <your_cert>

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

---

## ✅ Ghi chú

- Thay thế `<your_token>`, `<your_sha>`, và `<your_cert>` bằng giá trị thực tế do `kubeadm init` trả về.

---

