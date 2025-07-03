## Lab Environmental

## 1. Kubernetes Pre-requisites
### Disable Swap
Run the following on all nodes (master and workers):
```
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```
### Enable Firewall (UFW)
For local lab environment, you can disable UFW to avoid connectivity issues:
```
sudo systemctl disable --now ufw
```
or if you prefer to keep the firewall active, following ports needs to be enabled:
On Control Plane (kmaster):
```
sudo ufw allow 6443/tcp        # Kubernetes API server
sudo ufw allow 2379:2380/tcp   # etcd
sudo ufw allow 10250/tcp       # kubelet API
sudo ufw allow 10251/tcp       # kube-scheduler
sudo ufw allow 10252/tcp       # kube-controller-manager
```
On Worker Nodes (kworker1, kworker2):
```
sudo ufw allow 10250/tcp
sudo ufw allow 30000:32767/tcp # NodePort range
```
Then reload:
```
sudo ufw reload
```
### Firewall Bridging (Required for CNI)
Run this on all nodes:
```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```
Next execute:
```
sudo modprobe overlay
sudo modprobe br_netfilter
```
Similarly apply the following sysctl rules:
```
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
```
Execute to apply the changes:
```
sudo sysctl --system
```
## 2. Install Container Runtime (containerd)
Install this on all master and worker nodes:
Install pre-requisite packages:
```
sudo apt update && sudo apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates
```
Install containerd from Ubuntu repo
```
sudo apt install -y containerd
```
Generate and edit containerd config
```
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
```
Restart containerd
```
sudo systemctl restart containerd
sudo systemctl enable containerd
```
Verify containerd is working
```
sudo ctr version
```
## 3. Install kubelet, kubeadm, and kubectl
Install this on all master and worker nodes:
Remove Existing Kubernetes APT Source (if present):
```
sudo rm -f /etc/apt/sources.list.d/kubernetes.list
```
Create Keyrings Directory:
```
sudo mkdir -p /etc/apt/keyrings
```
Add the Kubernetes GPG Key. Replace v1.33 with your desired Kubernetes version if needed.
```
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key |  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```
Add the Kubernetes APT Repository:
```
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```
Update Package Lists and Install Kubernetes Components:
```
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```
This will install the latest available version (e.g., 1.33.x).

Verify Installation:
```
kubeadm version
kubectl version --client
```
## 4. Initialize the Kubernetes Cluster
To get the IP address, we can use ip a or ifconfig command. In my case, this is the interface which I intend to use with IP Address 192.168.56.105:
```
enp0s8: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.10.10  netmask 255.255.255.0  broadcast 192.168.10.255
        inet6 fe80::3ea0:3d4f:bbaf:9b41  prefixlen 64  scopeid 0x20<link>
        ether 08:00:27:78:0b:5b  txqueuelen 1000  (Ethernet)
        RX packets 31071  bytes 3677613 (3.6 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 18208  bytes 8486623 (8.4 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```
Execute the following command on master only. Calico CNI expects --pod-network-cidr=192.168.0.0/16
```
sudo kubeadm init --apiserver-advertise-address=192.168.10.10 --pod-network-cidr=192.168.0.0/16
```
> don't forget to replace IP-HOST ONLY MASTER> to your enp0s8 IP!
If the above command execution is successful then towards the end, you may see this output which contains kubeadm join command:
```
Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.56.105:6443 --token ggm1j4.n0diiqe2j4wvkh57 \
        --discovery-token-ca-cert-hash sha256:3c455b38ffd671a154b327e8a9a3b21cfb07103c06712b0c523baf44ad50b748
```
We will using this kubeadm join command later to add worker nodes to the Kubernetes Cluster.
Next let's set Up kubectl for our regular user on the master node:
```
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
Verify Cluster is Ready:
```
kubectl get nodes
```
> if nodes it's NotReady just wait 1-2 minutes and check again.
## 4.1 HANDLING ERROR (Opsional)
IF you get some error in above more like 'Error from server (ServiceUnavailable): the server is currently unable to handle the request (get nodes)' or whatever it is, for opsional strategis it's use this things. BUT, if Ready and your Cilium Status doesn't have error, that's good! you can skip this and do next method.
Bcs, after i use that (above) method, i got cilium status is error and cannot handdle it (actualy that's me who can't handdle it BCS ITS SO DAMN FRUSTASII)
So, what will you do is:
1) Cleaning ALL OF IT "Scorched Earth" (Gemini AI recommanded this but its usefull to me LMAO)
    ```
     # stop and delete service k3s
      /usr/local/bin/k3s-uninstall.sh || echo "k3s server not found, skipping."
      /usr/local/bin/k3s-agent-uninstall.sh || echo "k3s agent not found, skipping."
      
      # stop service if have
      sudo systemctl stop kubelet containerd
      
      # delete all of packet containerd.io if have
      sudo apt-get purge -y containerd.io
      
      # delete all directory if have in force
      sudo rm -rf /var/lib/containerd/ /var/lib/rancher/ /etc/rancher/ /etc/cni/
      
      # Clean remaining packet if no use
      sudo apt autoremove -y
      
      # Reboot all node for CLEANING
      sudo reboot
     ```
2) Install Cilium with Helm (kube-master):
    ```
    # Getting ready kubectl (if do)
    mkdir -p $HOME/.kube
    sudo cp -i /etc/rancher/k3s/k3s.yaml $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config
    
    # Instal Helm (if doesn't have)
    sudo snap install helm --classic
    
    # Instal Cilium
    helm repo add cilium https://helm.cilium.io/
    helm repo update
    helm install cilium cilium/cilium --namespace kube-system
    ```
3) Re-install K3s
   - Kube-master:
     ```
      # Ganti enp0s8 jika nama interface Host-Only Anda berbeda
     curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="server --node-ip=192.168.10.10 --flannel-iface=enp0s8 --flannel-backend=none" sh -
     ```
      > Get that token from master:  
     ```
     sudo cat /var/lib/rancher/k3s/server/node-token
     ```
   - Kube-worker1 and kube-worker2:
      Use token from master to join.
      > Worker1:
        ```
         curl -sfL https://get.k3s.io | K3S_URL=https://192.168.10.10:6443 K3S_TOKEN=<TOKEN_FROM_MASTER> INSTALL_K3S_EXEC="agent --node-ip=192.168.10.11" sh -
        ```
      > Worker2:
        ```
        curl -sfL https://get.k3s.io | K3S_URL=https://192.168.10.10:6443 K3S_TOKEN=<TOKEN_FROM_MASTER> INSTALL_K3S_EXEC="agent --node-ip=192.168.10.12" sh -
        ```
4) Verification
   Wait for 1-2 minutes, and in kube-master, run:
   ```
   sudo k3s kubectl get nodes
   ```
   ```
   cilium status
   ```
   > All of status of nodes will be like this:
     ![image](https://github.com/user-attachments/assets/d42c7deb-dc67-497b-9292-9105bfabfb71)
   > If your hubble-ui is disabled just run:
     ```
     cilium hubble enable --ui
     ```
     and wait for another minutes and run cilium status again.
AND DONE for OPSIONAL METHOD/STRATEGIES!
And for below it's for your kubeadm is ready.
## 5. Choosing and Installing CNI (Calico CNI)
Once the master node is initialized and kubectl is configured, run:
```
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.0/manifests/calico.yaml
```
Sample Output:
```
poddisruptionbudget.policy/calico-kube-controllers created
serviceaccount/calico-kube-controllers created
serviceaccount/calico-node created
serviceaccount/calico-cni-plugin created
configmap/calico-config created
customresourcedefinition.apiextensions.k8s.io/bgpconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/bgpfilters.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/bgppeers.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/blockaffinities.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/caliconodestatuses.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/clusterinformations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/felixconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworksets.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/hostendpoints.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamblocks.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamconfigs.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamhandles.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ippools.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipreservations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/kubecontrollersconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networksets.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/tiers.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/adminnetworkpolicies.policy.networking.k8s.io created
clusterrole.rbac.authorization.k8s.io/calico-kube-controllers created
clusterrole.rbac.authorization.k8s.io/calico-node created
clusterrole.rbac.authorization.k8s.io/calico-cni-plugin created
clusterrolebinding.rbac.authorization.k8s.io/calico-kube-controllers created
clusterrolebinding.rbac.authorization.k8s.io/calico-node created
clusterrolebinding.rbac.authorization.k8s.io/calico-cni-plugin created
daemonset.apps/calico-node created
deployment.apps/calico-kube-controllers created
```
After applying, verifiy nodes status:
```
kubectl get nodes
```
## 6. Join worker nodes with master node
Executing from both worker nodes:
```
ubuntu@kube-worker1:~$ sudo kubeadm join 192.168.10.10:6443 --token ggm1j4.n0diiqe2j4wvkh57      --discovery-token-ca-cert-hash sha256:3c455b38ffd671a154b327e8a9a3b21cfb07103c06712b0c523baf44ad50b748

ubuntu@kube-worker2:~$ sudo kubeadm join 192.168.10.10:6443 --token ggm1j4.n0diiqe2j4wvkh57   --discovery-token-ca-cert-hash sha256:3c455b38ffd671a154b327e8a9a3b21cfb07103c06712b0c523baf44ad50b748 
```
> don't forget to set your own IP if different from mine. if error, just ask Gemini AI...
After applying, verifiy nodes status:
```
kubectl get nodes
```
