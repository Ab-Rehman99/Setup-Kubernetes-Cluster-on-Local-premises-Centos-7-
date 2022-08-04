
# Setup Kubernetes Cluster on Local premises (Centos 7)




## Prerequisites

- Multiple servers running Centos 7 (1 Master Node, 2 Worker Nodes). Your all nodes should have atleast 2 CPU's.
- User with sudo or root privileges. 

# How to install Kubernetes on CentOS 7




## Update the packages

```bash
  sudo yum check-update
```
## Install the dependencies

```bash
  sudo yum install -y yum-utils device-mapper-persistent-data lvm2
```

# Step 1. Install Docker on all VMs

## Add and enable official Docker Repository

```bash
  sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

## Install the latest Docker version
```bash
  sudo yum install docker-ce
```
A successful installation output will be concluded with a Complete!



## Manage Docker Service on all VMs
```bash
  sudo systemctl start docker
```

```bash
  sudo systemctl enable docker
```
Check that the Docker is active and running.

```bash
  sudo systemctl status docker
```

# Step 2.  Set up the Kubernetes Repository

```bash
  sudo nano /etc/yum.repos.d/kubernetes.repo
```
Paste the below lines to the file...
```bash
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=0
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
```
For saving: Press ctrl+x then y then press Enter

# Step 3.  Install Kubelet

This should be done on all nodes.

```bash
  sudo yum install -y kubelet
```

# Step 4. Install kubeadm and kubectl 
This should also be done on all nodes.

```bash
  sudo yum install -y kubeadm
```
(Note that kubeadm automatically installs kubectl as a dependency)

```bash
  sudo systemctl enable kubelet
```

```bash
  sudo systemctl start kubelet
```

# Step 5. Set hostnames

## On Master
```bash 
  sudo hostnamectl set-hostname master-node
```

## On Worker
```bash 
  sudo hostnamectl set-hostname worker-node1
```

Now open the /etc/hosts file on every node and enter the hostnames for your Master, worker and their IP.
```bash
sudo nano /etc/hosts
```
Paste the below lines to the file...

```bash
x.x.x.x master-node
x.x.x.x node1 Worker-node1
```
# Step 6. Disable SElinux

If you have already Disabled Selinux you don't need to do this step.
Otherwise "aram se kro chup kr ke" 
```bash
sudo setenforce 0
```

```bash
sudo sed -i --follow-symlinks 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
```

```bash
reboot
```
Note: ( Reboot is necessary. )



# Step 7. Add firewall rules (On Master)

## On Master

```bash
sudo firewall-cmd --permanent --add-port=6443/tcp
sudo firewall-cmd --permanent --add-port=2379-2380/tcp
sudo firewall-cmd --permanent --add-port=10250/tcp
sudo firewall-cmd --permanent --add-port=10251/tcp
sudo firewall-cmd --permanent --add-port=10252/tcp
sudo firewall-cmd --permanent --add-port=10255/tcp
sudo firewall-cmd --reload
modprobe br_netfilter
echo '1' > /proc/sys/net/bridge/bridge-nf-call-iptables
```
All your firewall rule commands should output success.

## On Worker

```bash
  sudo firewall-cmd --permanent --add-port=10251/tcp
  sudo firewall-cmd --permanent --add-port=10255/tcp
  sudo firewall-cmd --permanent --add-port=6783/tcp
  sudo firewall-cmd --permanent --add-port=10250/tcp
  firewall-cmd --permanent --add-port=30000-32767/tcp
  sudo firewall-cmd --reload
  echo '1' > /proc/sys/net/bridge/bridge-nf-call-iptables
```

# Step 8. Update iptables config

This step should be perform on all nodes.

```bash 
  cat <<EOF > /etc/sysctl.d/k8s.conf
  net.bridge.bridge-nf-call-ip6tables = 1
  net.bridge.bridge-nf-call-iptables = 1
  EOF
  sudo sysctl --system
```
# Step 9. Disable swap

We also need to disable swap on all of our VMs:

```bash
sudo sed -i '/swap/d' /etc/fstab
```

```bash
sudo swapoff -a
```

# Deploying a Kubernetes Cluster


## Step 1. kubeadm initialization (On Master)

```bash
sudo kubeadm init
```

You might face errors related "Preflight". First, warning like if you have not assign 2 CPU's to each node. Second, If you feel error of CRI use the following command to resolve it.

```bash
  cat > /etc/containerd/config.toml <<EOF
  [plugins."io.containerd.grpc.v1.cri"]
  systemd_cgroup = true
  EOF
  systemctl restart containerd
```

You will also get an auto-generated command at the end of the output. Copy the text following the line Then you can join any number of worker nodes by running the following on each as root.

Note: (Have patience, this will take time).

## Step 2. Create required directories and start managing Kubernetes cluster

```bash
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## Step 3. Set up Pod network for the Cluster

```bash
  sudo kubectl get nodes
```

As you can see, the status of master–node is 'NotReady'.
 Make it Ready using below commands.

```bash 
  sudo export kubever=$(kubectl version --short | base64 | tr -d '\n')
```
(Ignore the warning, same wese jese tumhe duniya ignore krti hai.)


```bash
  kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$kubever"
```
If you see a lot of 'created' outputs then it's a good sign.

Now if you check the status of your master-node, it should be ‘Ready’.

```bash
  sudo kubectl get nodes
```

# Setting Up Worker Nodes to Join Kubernetes Cluster

## (On Worker Node)

As a final step, you need to add worker nodes to your cluster. We will use the kubeadm join auto-generated token in Step 1 (Deploying a Kubernetes Cluster). Run your own version of the following command on all of the worker node VMs:

```bash
  kubeadm join 192.168.100.24:6443 --token zrl8es.w46icfin1uu2mhfe \
        --discovery-token-ca-cert-hash sha256:41aad8d0763dc379a23639570e6bf6c5321e2136f1c9664e3ad1814cc22cecbe
```

Running the following command on the master-node should show your newly added node.

```bash
  sudo kubectl get nodes
```

(Define a ROLE to worker node)

```bash
  sudo kubectl label node w-node1 node-role.kubernetes.io/worker=worker
```

Now you’re all set up.

