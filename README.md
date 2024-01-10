# Setting a Kubernetes cluster to run a python Flask app locally

## Local resources
Three computers A, B, C running Ubuntu. In this case one was still using
18.04 (C), the other two 20.04 (A) and 22.04 (B)

## Overall config
computer A is designated as the control plane
computers B and C are nodes

## Common steps for ALL MACHINES
Following the steps
https://www.howtogeek.com/devops/how-to-start-a-kubernetes-cluster-from-scratch-with-kubeadm-and-kubectl/

### containerd
```
sudo apt update
sudo apt install -y ca-certificates curl gnupg lsb-release
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
(run the following command in bash or sh shells)
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install -y containerd.io
sudo service containerd status
```
Post-installation config:
```
sudo containerd config default > /etc/containerd/config.toml
```
and change the setting of ```SystemdCgroup``` to ```true```
```
SystemdCgroup = false
```
then restart
```
sudo service containerd restart
```

### Kubernetes

```
sudo curl -fsSLo /etc/apt/keyrings/kubernetes.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update
sudo apt install -y kubeadm kubectl kubelet
sudo apt-mark hold kubeadm kubectl kubelet
```

### turning off swap
```
sudo swapoff -a
```
and edit ```etc/fstab``` to comment out or remove the swap mount

### others
```
sudo modprobe br_netfilter
echo br_netfilter | sudo tee /etc/modules-load.d/kubernetes.conf
```

## Setup and configs for control plane 
```
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```
