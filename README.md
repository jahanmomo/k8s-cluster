### Sudo
sudo -s

## Configure Infrastructure
sudo swapoff -a
printf "\n192.168.17.136 worker2\n\n" >> /etc/hosts


##on master server
sudo hostnamectl set-hostname master 

# CRI (containerd install)- containerd.sh: 

## Now initialize container runtime interface (CRI) both control plane & workers
https://github.com/containerd/containerd/blob/main/docs/getting-started.md
https://kubernetes.io/docs/setup/production-environment/container-runtimes/#containerd

### Here it is:
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
	
sudo modprobe overlay
sudo modprobe br_netfilter
	
### sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF
	
### Apply sysctl params without reboot
sudo sysctl --system

# Install & configure containerd 
sudo apt-get update 
sudo apt-get -y install containerd

sudo containerd config default | sudo tee /etc/containerd/config.toml  
sudo systemctl restart containerd	
service containerd status

sudo wget https://github.com/containerd/containerd/releases/download/v1.7.11/containerd-1.7.11-linux-amd64.tar.gz  # Corrected URL for wget
	
sudo tar -C /usr/local -xzvf containerd-1.7.11-linux-amd64.tar.gz  # Corrected option order for tar
	
sudo systemctl daemon-reload 
sudo systemctl enable --now containerd
service containerd status


# kubeadm, kubelet & kubectl install both worker & node- kube.sh
apt-get update

## apt-transport-https may be a dummy package; if so, you can skip that package
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

## This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update

reboot

sudo apt-get install -y kubelet=1.21.0-00 kubeadm=1.21.0-00 kubectl=1.21.0-00 or 
sudo apt-get install -y kubelet=1.29.0-1.1 kubeadm=1.29.0-1.1 kubectl=1.29.0-1.1 or
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

### For checking those installed or not:
service kubelet status (will not active because it is waiting for kubeadm initializing) 
kubeadm --help
kubeadm version
kubectl version


# Now initializing kubeadm first and other things

### check swap config, ensure swap is 0
free -m

# Initialize K8s cluster in control plane
sudo kubeadm init --pod-network-cidr=192.168.0.0/16 or 
sudo kubeadm init # default

### Check kubelet process running 
service kubelet status or
systemctl status kubelet

## Create:
mkdir -p /etc/kubernetes
sudo kubectl get node --kubeconfig /etc/kubernetes/admin.conf

## edit and change systemdCgroup to true		
sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml		
sudo cat /etc/kubernetes/admin.conf

### Check extended logs of kubelet service
journalctl -u kubelet

## Access cluster as admin
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config


# Container Network Interface
Using weave

## Install pod network plugin - download and install the manifest
wget "https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml" 
kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml
https://www.weave.works/docs/net/latest/kubernetes/kube-addon/

[Link to the Weave-net installation guide](https://www.weave.works/docs/net/latest/kubernetes/kube-addon/#-installation)    

# on master
kubeadm token create --help
kubeadm token create --print-join-command

# copy the output command and execute on worker node as ROOT
sudo kubeadm join 172.31.43.99:6443 --token 9bds1l.3g9ypte9gf69b5ft --discovery-token-ca-cert-hash sha256:xxxx

## open weave net port both control plane & worker nodes
port: 6783
port: 6784

## weave status
kubectl get pod -n kube-system -o wide | grep weave

