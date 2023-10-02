The minimum requirements for the viable setup are:

- Memory: 2+ GiB each
- CPUs: 2+ CPUs on master.
- Internet connection
- Full network connectivity between machines in the cluster

**Step 1: Install K8s Servers**

```
sudo apt update && sudo apt -y full-upgrade [ -f /var/run/reboot-required ] &&
sudo reboot -f
```


**Step 2: Install kubelet, kubeadm and kubectl**

*add Kubernetes repository for Ubuntu to all the servers*

```
sudo apt -y install curl apt-transport-https curl -fsSL
https://packages.cloud.google.com/apt/doc/apt-key.gpg|sudo gpg --dearmor -o
/etc/apt/trusted.gpg.d/kubernetes.gpg echo "deb https://apt.kubernetes.io/
kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

*Install required packages:*


```
sudo apt update
sudo apt -y install vim git curl wget kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```


*Check:*


```
kubectl version --client && kubeadm version
```


**Step 3: Disable Swap**

*3.1. Turn off swap*

```
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

*3.2. Disable Linux swap space permanently in /etc/fstab*


```
$ sudo vim /etc/fstab
#/swap.img	none	swap	sw	0	0
```


*3.3. Confirm setting*


```
sudo swapoff -a
sudo mount -a
free -h
```

*3.4 Enable kernel modules and configure sysctl.*

- Enable kernel modules and configure sysctl.

```
# Enable kernel modules
sudo modprobe overlay
sudo modprobe br_netfilter

# Add some settings to sysctl
sudo tee /etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

# Reload sysctl
sudo sysctl --system
```

**Step 4: Install Container runtime - Docker CE or CRI-O or Containerd**

*4.1 Docker CE*


For Docker Engine you need a shim interface. You can install Mirantis cri-dockerd as covered in the guide below.

-[Install Mirantis cri-dockerd as Docker Engine shim for Kubernetes](https://computingforgeeks.com/install-mirantis-cri-dockerd-as-docker-engine-shim-for-kubernetes/)

Mirantis cri-dockerd CRI socket file path is /run/cri-dockerd.sock. This is what will be used when configuring Kubernetes cluster.

```
# Add repo and Install packages
sudo apt update
sudo apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/docker-archive-keyring.gpg
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
sudo apt update
sudo apt install -y containerd.io docker-ce docker-ce-cli

# Create required directories
sudo mkdir -p /etc/systemd/system/docker.service.d

# Create daemon json config file
sudo tee /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF

# Start and enable Services
sudo systemctl daemon-reload
sudo systemctl restart docker
sudo systemctl enable docker
```



*4.2 CRI-O*

```
# Ensure you load modules
sudo modprobe overlay
sudo modprobe br_netfilter

# Set up required sysctl params
sudo tee /etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

# Reload sysctl
sudo sysctl --system

# Add Cri-o repo
sudo su -
OS="xUbuntu_20.04"
VERSION=1.27
echo "deb https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/ /" > /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list
echo "deb http://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/$VERSION/$OS/ /" > /etc/apt/sources.list.d/devel:kubic:libcontainers:stable:cri-o:$VERSION.list
curl -L https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable:cri-o:$VERSION/$OS/Release.key | apt-key add -
curl -L https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/Release.key | apt-key add -

# Install CRI-O
sudo apt update
sudo apt install cri-o cri-o-runc

# Update CRI-O CIDR subnet
sudo sed -i 's/10.85.0.0/10.99.0.0/g' /etc/cni/net.d/100-crio-bridge.conf
sudo sed -i 's/10.85.0.0/10.99.0.0/g' /etc/cni/net.d/100-crio-bridge.conflist

# Start and enable Service
sudo systemctl daemon-reload
sudo systemctl restart crio
sudo systemctl enable crio
sudo systemctl status crio
```


*4.3. Containerd*


```
# Configure persistent loading of modules
sudo tee /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF

# Load at runtime
sudo modprobe overlay
sudo modprobe br_netfilter

# Ensure sysctl params are set
sudo tee /etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

# Reload configs
sudo sysctl --system

# Install required packages
sudo apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates

# Add Docker repo
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/docker-archive-keyring.gpg
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"

# Install containerd
sudo apt update
sudo apt install -y containerd.io

# Configure containerd and start service
sudo mkdir -p /etc/containerd
sudo containerd config default|sudo tee /etc/containerd/config.toml

# restart containerd
sudo systemctl restart containerd
sudo systemctl enable containerd
systemctl status  containerd
```


To use the systemd cgroup driver, set plugins.cri.systemd_cgroup = true in /etc/containerd/config.toml. When using kubeadm, manually configure the [cgroup driver for kubelet](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#configure-cgroup-driver-used-by-kubelet-on-control-plane-node)



**Step 5: Initialize master node**


- Login to the server to be used as master and make sure that the br_netfilter module is loaded:

````
$ lsmod | grep br_netfilter
br_netfilter           22256  0
bridge                151336  2 br_netfilter,ebtable_broute

````

- Enable kubelet service.

```sudo systemctl enable kubelet```


We now want to initialize the machine that will run the control plane components which includes etcd (the cluster database) and the API Server.

- Pull container images:

```
$ sudo kubeadm config images pull
[config/images] Pulled registry.k8s.io/kube-apiserver:v1.27.2
[config/images] Pulled registry.k8s.io/kube-controller-manager:v1.27.2
[config/images] Pulled registry.k8s.io/kube-scheduler:v1.27.2
[config/images] Pulled registry.k8s.io/kube-proxy:v1.27.2
[config/images] Pulled registry.k8s.io/pause:3.9
[config/images] Pulled registry.k8s.io/etcd:3.5.7-0
[config/images] Pulled registry.k8s.io/coredns/coredns:v1.10.1
```

- If you have multiple CRI sockets, please use --cri-socket to select one:

```
# CRI-O
sudo kubeadm config images pull --cri-socket unix:///var/run/crio/crio.sock

# Containerd
sudo kubeadm config images pull --cri-socket unix:///run/containerd/containerd.sock

# Docker
sudo kubeadm config images pull --cri-socket unix:///run/cri-dockerd.sock
```


- These are the basic kubeadm init options that are used to bootstrap cluster.


```
--control-plane-endpoint :  set the shared endpoint for all control-plane nodes. Can be DNS/IP
--pod-network-cidr : Used to set a Pod network add-on CIDR
--cri-socket : Use if have more than one container runtime to set runtime socket path
--apiserver-advertise-address : Set advertise address for this particular control-plane node's API server
```

*Bootstrap without shared endpoint*
- To bootstrap a cluster without using DNS endpoint, run:

```
### With Docker CE ###
sudo sysctl -p
sudo kubeadm init \
  --pod-network-cidr=10.99.0.0/16 \
  --cri-socket unix:///run/cri-dockerd.sock

### With CRI-O###
sudo sysctl -p
sudo kubeadm init \
  --pod-network-cidr=10.99.0.0/16 \
  --cri-socket unix:///var/run/crio/crio.sock

### With Containerd ###
sudo sysctl -p
sudo kubeadm init \
  --pod-network-cidr=10.99.0.0/16 \
  --cri-socket unix:///run/containerd/containerd.sock
```

*Bootstrap with shared endpoint (DNS name for control plane API)*
- Set cluster endpoint DNS name or add record to /etc/hosts file.

```
$ sudo vim /etc/hosts
10.99.105.31 k8s-cluster.domain.com
```

**Create cluster:**

```
sudo sysctl -p
sudo kubeadm init \
  --pod-network-cidr=10.99.0.0/16 \
  --upload-certs \
  --control-plane-endpoint=k8s-cluster.domain.com
```

Note: If 10.99.0.0/16 is already in use within your network you must select a different pod network CIDR, replacing 10.99.0.0/16 in the above command.


- Container runtime sockets:

| Runtime    |       Path to Unix domain socket       |
| ---------- | :------------------------------------: |
| Docker     |      unix:///run/cri-dockerd.sock      |
| Containerd | unix:///run/containerd/containerd.sock |
| CRI-O      |     unix:///var/run/crio/crio.sock     |


- You can optionally pass Socket file for runtime and advertise address depending on your setup.

```
# CRI-O
sudo kubeadm init \
  --pod-network-cidr=10.92.0.0/16 \
  --cri-socket unix:///var/run/crio/crio.sock \
  --upload-certs \
  --control-plane-endpoint=k8s-cluster.domain.com

# Containerd
sudo kubeadm init \
  --pod-network-cidr=10.92.0.0/16 \
  --cri-socket unix:///run/containerd/containerd.sock \
  --upload-certs \
  --control-plane-endpoint=k8s-cluster.domain.com

# Docker
# Must do https://computingforgeeks.com/install-mirantis-cri-dockerd-as-docker-engine-shim-for-kubernetes/
sudo kubeadm init \
  --pod-network-cidr=10.92.0.0/16 \
  --cri-socket unix:///run/cri-dockerd.sock  \
  --upload-certs \
  --control-plane-endpoint=k8s-cluster.domain.com
```

- Configure kubectl using commands in the output:

```
mkdir -p $HOME/.kube
sudo cp -f /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

- Check cluster status:

```
$ kubectl cluster-info
Kubernetes master is running at https://k8s-cluster.domain.com:6443
KubeDNS is running at https://k8s-cluster.domain.com:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```


- Additional Master nodes can be added using the command in installation output:

```
kubeadm join k8s-cluster.computingforgeeks.com:6443 --token sr4l2l.2kvot0pfalh5o4ik \
    --discovery-token-ca-cert-hash sha256:c692fb047e15883b575bd6710779dc2c5af8073f7cab460abd181fd3ddb29a18 \
    --control-plane
```

**Step 6: Install network plugin on Master**

In this guide we’ll use _Calico_. You can choose [any other supported network plugins](https://kubernetes.io/docs/concepts/cluster-administration/addons/).

Download operator and custom resource files. See [releases page](https://github.com/projectcalico/calico/releases) for latest version.

- First, install the operator on your cluster.

```
$ kubectl create -f tigera-operator.yaml
```


- If you wish to customize the Calico install, customize the downloaded custom-resources.yaml. For example we are updating CIDR.

```
$ sed -ie 's/192.168.0.0/10.99.0.0/g' custom-resources.yaml
``


- Use the manifest locally then install Calico:

```
\$ kubectl create -f custom-resources.yaml
````

- Confirm that all of the pods are running:

```
$ kubectl get pods --all-namespaces -w
```

- For Single node cluster allow Pods to run on master nodes:

```
kubectl taint nodes --all node-role.kubernetes.io/master-
kubectl taint nodes --all  node-role.kubernetes.io/control-plane-
```

- Confirm master node is ready:

````# CRI-O
$ kubectl get nodes -o wide
# Containerd
$ kubectl get nodes -o wide
# Docker
$ kubectl get nodes -o wide
````

**Step 7: Add worker nodes**

- If endpoint address is not in DNS, add record to /etc/hosts.

```$ sudo vim /etc/hosts```

- The join command that was given is used to add a worker node to the cluster.

```
kubeadm join k8s-cluster.computingforgeeks.com:6443 \
  --token sr4l2l.2kvot0pfalh5o4ik \
  --discovery-token-ca-cert-hash sha256:c692fb047e15883b575bd6710779dc2c5af8073f7cab460abd181fd3ddb29a18
```

- Run below command on the control-plane to see if the node joined the cluster.

```
kubectl get nodes
--------
kubectl get nodes -o wide
```

- If the join token is expired, refer to our guide on how to join worker nodes.
[Join new Kubernetes Worker Node to an existing Cluster](https://computingforgeeks.com/join-new-kubernetes-worker-node-to-existing-cluster/)

**Step 8: Deploy application on cluster**

- If you only have a single node cluster, check our guide on how to run container pods on master nodes:

[Scheduling Pods on Kubernetes Control plane (Master) Nodes](https://computingforgeeks.com/how-to-schedule-pods-on-kubernetes-control-plane-node/)

- We need to validate that our cluster is working by deploying an application.

`kubectl apply -f https://k8s.io/examples/pods/commands.yaml`


- Check to see if pod started

```
$ Kubectl get pods
```

**Step 9: Install Kubernetes Dashboard (Optional)**

Kubernetes dashboard can be used to deploy containerized applications to a Kubernetes cluster, troubleshoot your containerized application, and manage the cluster resources.

**Step 10: Install Metrics Server ( For checking Pods and Nodes resource usage)**
Metrics Server is a cluster-wide aggregator of resource usage data. It collects metrics from the Summary API, exposed by Kubelet on each node. Use our guide below to deploy it:

[How To Deploy Metrics Server to Kubernetes Cluster](https://computingforgeeks.com/how-to-deploy-metrics-server-to-kubernetes-cluster/)

**Step 11: Deploy Prometheus / Grafana Monitoring**
Prometheus is a full fledged solution that enables you to access advanced metrics capabilities in a Kubernetes cluster. Grafana is used for analytics and interactive visualization of metrics that’s collected and stored in Prometheus database. We have a complete guide on how to setup complete monitoring stack on Kubernetes Cluster:

- [Setup Prometheus and Grafana on Kubernetes using prometheus-operator](https://computingforgeeks.com/setup-prometheus-and-grafana-on-kubernetes/)

Reference[^ref]
[^ref]: [Computingforgeeks](https://computingforgeeks.com/deploy-kubernetes-cluster-on-ubuntu-with-kubeadm/?expand_article=1).
Viblo[^ref]
[^ref]: [Viblo](https://viblo.asia/p/k8s-phan-1-kubernetes-la-gi-bJzKmAyDK9N).
