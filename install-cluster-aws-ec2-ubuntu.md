# Install Kubernetes Cluster using kubeadm

> AWS EC2 - Ubuntu 20.04

## Reference

* [kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/)
* [CNI](https://kubernetes.io/docs/concepts/cluster-administration/networking/)
* [Ports](https://kubernetes.io/docs/reference/ports-and-protocols/)
* [Docker](https://docs.docker.com/engine/install/ubuntu/)

## Common Node

##### Login as `root` user

```
sudo su -
```

##### Disable swap

```
swapoff -a
sed -i '/swap/d' /etc/fstab
```

##### [Forwarding IPv4 and letting iptables see bridged traffic](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#forwarding-ipv4-and-letting-iptables-see-bridged-traffic) 

```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter

# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system
```

##### [Docker Engine](https://docs.docker.com/engine/install/ubuntu/)

```
# Set up the repository

apt-get update

apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release

mkdir -p /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```


```
# Install Docker Engine

apt-get update

apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

apt-cache madison docker-ce

apt-get install docker-ce=<VERSION_STRING> docker-ce-cli=<VERSION_STRING> containerd.io docker-compose-plugin

docker run hello-world
```

##### [Installing kubeadm, kubelet and kubectl](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl)

```
apt-get update

apt-get install -y apt-transport-https ca-certificates curl

curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg

echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

apt-get update

apt-get install -y kubelet kubeadm kubectl

apt-mark hold kubelet kubeadm kubectl
```

##### containerd

```
vim /etc/containerd/config.toml

# disabled_plugins = ["cri"]
```

```
systemctl restart containerd
```

## Master Node

| Instance |
| -------- |
| t2.medium |

##### kubeadm

```
kubeadm init

# Use Flannel
kubeadm init --pod-network-cidr=10.244.0.0/16
```

```
crictl --runtime-endpoint unix:///var/run/containerd/containerd.sock ps -a
```
![image](https://i.imgur.com/qshT5Wy.jpg)

##### Login as `ubuntu` user

```
mkdir -p $HOME/.kube

sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

##### [CNI](https://kubernetes.io/docs/concepts/cluster-administration/networking/)

Use [Flannel](https://github.com/flannel-io/flannel)
```
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

## Worker Node

| Instance |
| -------- |
| t2.micro |

##### 

```
kubeadm join <PRIVATE_IP>:6443 --token <TOKEN> \
	--discovery-token-ca-cert-hash sha256:<HASH>
```