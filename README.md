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
Save the output from this init command to continue setup and to add nodes
to this cluster.

Continue setup for Kubeconfig
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
Install networking addon - or do your own research on which one to pick
from https://kubernetes.io/docs/concepts/cluster-administration/addons/#networking-and-network-policy
```
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```
Check if the cluster is up and running with just this control plane machine
in it
```
kubectl get nodes
kubectl get pods --all-namespaces
```


### adding nodes
From each node (B or C)
```
kubeadm join ip_address_A:6443 --token <token> --discovery-token-ca-cert-hash sha256:<token-ca-cert-hash>
```
From the control planes:
```
kubectl get nodes
```

if a node shows as NotReady, can try this
```
systemctl stop apparmor
systemctl disable apparmor 
systemctl restart containerd.service
```

## Install docker
https://docs.docker.com/engine/install/ubuntu/
```
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### allow user to run docker without having to sudo
https://stackoverflow.com/questions/48957195/how-to-fix-docker-got-permission-denied-issue
```
sudo groupadd docker
sudo usermod -aG docker username
newgrp docker
sudo systemctl restart docker
```

## Changes to allow local registry and http instead of https image pull
DO THIS ON ALL MACHINES (A, B and C)
### docker daemon
create ```/etc/docker/daemon.json``` and add line below
```
{ "insecure-registries":["ip_address_C:5000"] }
```
then add to ```/etc/default/docker```
```
DOCKER_OPTS="--config-file=/etc/docker/daemon.json"
```
and restart
```
systemctl stop docker
systemctl start docker
```

### containerd
https://stackoverflow.com/questions/65681045/adding-insecure-registry-in-containerd/67310470#67310470
Edit ```/etc/containerd/config.toml``` to append lines as below where IP is ip_address_C
```
[plugins."io.containerd.grpc.v1.cri".registry]
  [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
    [plugins."io.containerd.grpc.v1.cri".registry.mirrors."<IP>:5000"]
      endpoint = ["http://<IP>:5000"]
  [plugins."io.containerd.grpc.v1.cri".registry.configs]
    [plugins."io.containerd.grpc.v1.cri".registry.configs."<IP>:5000".tls]
      insecure_skip_verify = true
```
and restart containerd
```
sudo systemctl restart containerd
```


## Docker images

### running local registry
https://www.docker.com/blog/how-to-use-your-own-registry-2/
e.g. do this using port 5000
```
docker run -d --restart=always -p 5000:5000 --name registry registry:2.7
```

### sample Flask app
Create a sample python app
```
from flask import Flask, render_template
app = Flask(__name__)

@app.route("/")     
def hello():
   return "Hello World!"

@app.route("/home")
def home():
   return render_template("hello.html")

if __name__ == "__main__":
   # run the flask app
   app.run(host='0.0.0.0', debug=True, port=5123)
```

