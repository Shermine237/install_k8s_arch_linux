# install_k8s_arch_linux
How to install kubernetes barremetal on arch linux and create one cluster

## 1- Deployment tools
```bash
sudo pacman -S kubeadm kubectl kubelet docker
```

## 2- Setup forwarding IPv4 and letting iptables see bridged traffic
[run] :
```bash
sudo vim /etc/modules-load.d/k8s.conf
```
[add] :
```bash
overlay
br_netfilter
```
[run] :
```bash
sudo vim /etc/sysctl.d/k8s.conf
```
[add] :
```bash
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
```
Apply them without rebooting :
```bash
sudo sysctl --system
sudo systemctl enable containerd.service
sudo systemctl start containerd.service
```

## 3- Disable swap
```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

## 4- Choose cgroup drive
```bash
sudo containerd
sudo mkdir /etc/containerd
sudo touch /etc/containerd/config.toml
sudo chown $(id -u):$(id -g) /etc/containerd/config.toml
containerd config default > /etc/containerd/config.toml
```
[run] :
```bash
vim /etc/containerd/config.toml
```
[search and replace] :
```bash
SystemdCgroup = true
```

## 5- Restart containerd.service
```bash
sudo systemctl restart containerd.service
```

## 6- Start docker
```bash
sudo systemctl daemon-reload
sudo systemctl enable docker
sudo systemctl start docker
```

## 7- Configure crictl to use containerd cri
```bash
sudo crictl config --set runtime-endpoint=unix:///run/containerd/containerd.sock --set image-endpoint=unix:///run/containerd/containerd.sock
```

## 8- Create virtual interface to use as controler ip
```bash
sudo pacman -S net-tools
```
[read this repository and create veth] :
https://github.com/Shermine237/script-init-k8s.git

## 9- Init kubernetes
```bash
sudo systemctl enable kubelet
sudo kubeadm init --cri-socket unix:///run/containerd/containerd.sock --pod-network-cidr=192.168.0.0/16 --apiserver-advertise-address 192.168.100.100
```

## 10- Initialising
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
export KUBECONFIG=$HOME/.kube/config
```

## 11- Config network plugin (project calico)
```bash
curl -L https://github.com/projectcalico/calico/releases/download/v3.29.3/calicoctl-linux-amd64 -o calicoctl
sudo chmod +x calicoctl 
sudo mv calicoctl /usr/bin
```
```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.3/manifests/tigera-operator.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.3/manifests/custom-resources.yaml
watch kubectl get pods -n calico-system
```
Wait until each pod has the STATUS of Running

## 12- Remove the taints on the control plane so that you can schedule pods on it
```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

## 13- Confirm that you now have a node in your cluster with the following command (state running)
```bash
kubectl get nodes -o wide
```







