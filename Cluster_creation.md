#Step 1: Create EC2 Instances

Go to AWS Console → EC2 → Launch Instances

Create 3 instances:

AMI: Ubuntu 22.04 LTS
Instance type: t3.medium (minimum)
Key pair: Create or use an existing one
Network: Default VPC
Subnet: Same for all three

#Security group: Create new one with below rules 👇

Security Group Rules
Type	Protocol	Port Range	Source
SSH	TCP	22	Your IP
Kubernetes API	TCP	6443	Custom: 0.0.0.0/0 or VPC CIDR
Kubelet API	TCP	10250	Custom: VPC CIDR
NodePort Services	TCP	30000–32767	Custom: 0.0.0.0/0
ICMP	-	-	Custom: VPC CIDR

#Allow intra-node communication by allowing traffic from the same security group.

#Step 2: Set Hostnames on Each Node
#SSH into each node and set hostname:
# Control Plane
sudo hostnamectl set-hostname k8s-master

# worker 1
sudo hostnamectl set-hostname k8s-worker1

# Worker 2
sudo hostnamectl set-hostname k8s-worker2

#Then reconnect SSH to see new hostname.

#Step 3: Common Setup (Run on ALL 3 Nodes)
sudo apt update && sudo apt upgrade -y
sudo apt install -y apt-transport-https ca-certificates curl gnupg lsb-release

#Disable Swap
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

#Enable Required Kernel Modules
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

#Configure sysctl for Kubernetes Networking
cat <<EOF | sudo tee /etc/sysctl.d/kubernetes.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system

#Step 4: Install Container Runtime (Containerd)
sudo apt install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml > /dev/null

#Change cgroup driver to systemd:

sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

#Then restart:

sudo systemctl restart containerd
sudo systemctl enable containerd

#Step 5: Install kubeadm, kubelet, kubectl (All Nodes)
#create the keyring directory properly
sudo mkdir -p /etc/apt/keyrings
sudo chmod 755 /etc/apt/keyrings

#Download the key correctly
#Sometimes curl fails due to permission issues — so run with sudo tee instead:

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

#Check that file exists:

ls -l /etc/apt/keyrings/kubernetes-apt-keyring.gpg

#You should see a file (around 11–12 KB in size).

#Add the Kubernetes APT repo again
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" | \
sudo tee /etc/apt/sources.list.d/kubernetes.list

#Update and install packages
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

#Verify After installation:

kubeadm version
kubectl version --client
kubelet --version

#Step 6: Initialize Control Plane (Only on Master)
#Find your master node’s private IP:

hostname -I

#Run kubeadm init (replace <PRIVATE_IP>):

sudo kubeadm init \
  --apiserver-advertise-address=<PRIVATE_IP> \
  --pod-network-cidr=192.168.0.0/16

#This will take a few minutes.

#At the end, it will display a command like:

kubeadm join 10.0.0.10:6443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:11223344aabbccddeeff...

#Save that command — you’ll need it for workers.

#Step 7: Configure kubectl Access (on Master)
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

#Test:

kubectl get nodes

#You’ll see only the master (not Ready yet — because networking isn’t installed).

#Step 8: Install Network Plugin (Calico)
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

#Wait until pods are ready:

kubectl get pods -n kube-system

#Once Calico pods are running, master node will show Ready.

#Step 9: Join Worker Nodes

#On each worker node, paste the join command you saved earlier.
#Example:

sudo kubeadm join 10.0.0.10:6443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:11223344aabbccddeeff...

#If you lost it, recreate it on the master:

kubeadm token create --print-join-command

#Step 10: Verify Cluster

#Back on the control plane:

kubectl get nodes

#Expected output:

NAME           STATUS   ROLES           AGE   VERSION
k8s-master     Ready    control-plane   10m   v1.30.x
k8s-worker1    Ready    <none>          2m    v1.30.x
k8s-worker2    Ready    <none>          2m    v1.30.x
