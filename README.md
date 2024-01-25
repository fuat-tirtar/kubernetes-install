# Kubernetes Cluster Kurulumu
![image](https://github.com/fuat-tirtar/kubernetes-install/assets/58062840/33fa7d13-1dd2-4dd3-9a50-c5501ebae050)

# **Step 1. Install containerd**
To install containerd, follow these steps on both VMs:

**1.** Load the br_netfilter module required for networking

**sudo modprobe overlay
sudo modprobe br_netfilter
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF**

**2.** To allow iptables to see bridged traffic, as required by Kubernetes, we need to set the values of certain fields to 1.

**sudo tee /etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF**

**3.** Apply the new settings without restarting.                                            
**sudo sysctl --system**     
                                             
**4.** Install curl.
**sudo apt install curl -y**
                                              
**5.** Get the apt-key and then add the repository from which we will install containerd.
**curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"**  
                                              
Expect an output similar to the following:
**Get:2 https://download.docker.com/linux/ubuntu focal/stable amd64 Packages [17.6 kB]                                   
Get:3 http://security.ubuntu.com/ubuntu focal-security InRelease [114 kB]                                     
Hit:4 http://archive.ubuntu.com/ubuntu focal InRelease                      
Get:5 http://archive.ubuntu.com/ubuntu focal-updates InRelease [114 kB]
Get:6 http://archive.ubuntu.com/ubuntu focal-backports InRelease [108 kB]
Fetched 411 kB in 2s (245 kB/s)    
Reading package lists... Done**
                                              
**6.** Update and then install the containerd package.
**sudo apt update -y 
sudo apt install -y containerd.io**

**7.** Set up the default configuration file.
**sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml**
                                              
**8.** Next up, we need to modify the containerd configuration file and ensure that the cgroupDriver is set to systemd. To do so, edit the following file:  
**sudo nano /etc/containerd/config.toml**
Scroll down to the following section:
**[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]**
                                              
And ensure that value of **SystemdCgroup** is set to **true** Make sure the contents of your section match the following:
**[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
BinaryName = ""
CriuImagePath = ""
CriuPath = ""
CriuWorkPath = ""
IoGid = 0
IoUid = 0
NoNewKeyring = false
NoPivotRoot = false
Root = ""
ShimCgroup = ""
SystemdCgroup = true**
**9.** Finally, to apply these changes, we need to restart containerd.
**sudo systemctl restart containerd**

#  Step 2. Install Kubernetes       
With our container runtime installed and configured, we are ready to install Kubernetes.
**1.** Add the repository key and the repository.
**curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list**
                                              
**2.** Update your system and install the 3 Kubernetes modules
**sudo apt update -y
sudo apt install -y kubelet kubeadm kubectl**
                                              
**3.** Set the appropriate hostname for each machine.
**sudo hostnamectl set-hostname "master-node"
exec bash**
                                              
**4.** Add the new hostnames to the /etc/hosts file on both servers.
**sudo nano /etc/hosts
192.168.10.161 master-node**   

**5.** Set up the firewall by installing the following rules on the master node:
**sudo ufw allow 6443/tcp
sudo ufw allow 2379/tcp
sudo ufw allow 2380/tcp
sudo ufw allow 10250/tcp
sudo ufw allow 10251/tcp
sudo ufw allow 10252/tcp
sudo ufw allow 10255/tcp
sudo ufw reload**

**6.** To allow kubelet to work properly, we need to disable swap on both machines.
**sudo swapoff –a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
sudo nano /etc/fstab**
comment the swapfile line                               
**##/swap.img     none    swap    sw      0       0**

**7.** Configure persistent loading of modules 
	**tee /etc/modules-load.d/containerd.conf <<EOF
	overlay
	br_netfilter
	vhost_vsock
	EOF**
    
**8.** Load at runtime
	**modprobe overlay
	modprobe br_netfilter
	modprobe vhost_vsock**
  
**9.** Apply the new settings without restarting.   
 **sudo sysctl --system** 
                                                 
# Step 3. Setting up the cluster                                               
With our container runtime and Kubernetes modules installed, we are ready to initialize our Kubernetes cluster.
                                                 
**1.** Run the following command on the master node to allow Kubernetes to fetch the required images before cluster initialization: 
**sudo kubeadm config images pull**
                                                 
**2.** Initialize the cluster 
**sudo kubeadm init --pod-network-cidr=10.244.0.0/16**                                         
The initialization may take a few moments to finish. Expect an output similar to the following: **Your Kubernetes control-plane has initialized successfully!**
                                                 
To start using your cluster, you need to run the following as a regular user:
**mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config**

Alternatively, if you are the root user, you can run:
**export KUBECONFIG=/etc/kubernetes/admin.conf**
                                                 
You should now deploy a pod network to the cluster. Run kubectl apply -f [podnetwork].yaml with one of the options listed at Kubernetes.

Then you can join any number of worker nodes by running the following on each as root: 
**kubeadm join 102.130.122.60:6443 --token s3v1c6.dgufsxikpbn9kflf \
        --discovery-token-ca-cert-hash sha256:b8c63b1aba43ba228c9eb63133df81632a07dc780a92ae2bc5ae101ada623e00**
                                                 
 You will see a kubeadm join at the end of the output. Copy and save it in some file. We will have to run this command on the worker node to allow it to join the cluster. But fear not, if you forget to save it, or misplace it, you can also regenerate it using this command:  
**sudo kubeadm token create --print-join-command**

**3.**  Now create a folder to house the Kubernetes configuration in the home directory. We also need to set up the required permissions for the directory, and export the KUBECONFIG variable.
**mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
export KUBECONFIG=/etc/kubernetes/admin.conf**

**4.** Deploy a pod network to our cluster. This is required to interconnect the different Kubernetes components.
**NODENAME=$(kubectl describe nodes | grep "Name:" | awk '{print $2}')
kubectl taint nodes $NODENAME node-role.kubernetes.io/control-plane:NoSchedule-                                               
kubectl apply -f https://github.com/coreos/flannel/raw/master/Documentation/kube-flannel.yml**
                                                 
Expect an output like this: 
**podsecuritypolicy.policy/psp.flannel.unprivileged created
clusterrole.rbac.authorization.k8s.io/flannel created
clusterrolebinding.rbac.authorization.k8s.io/flannel created
serviceaccount/flannel created
configmap/kube-flannel-cfg created
daemonset.apps/kube-flannel-ds created**
                                                 
**5.** Use the get nodes command to verify that our master node is ready
**kubectl get nodes**   
Expect the following output:
**NAME          STATUS   ROLES           AGE     VERSION
master-node   Ready    control-plane    6d22h   v1.28.2** 
                                                 
**6.** Install dashboard
**wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml**
nano recommended.yaml

**--
spec:
  ports:
    - port: 443
      targetPort: 8443
      nodePort: 30001
  type: NodePort**
 
**kubectl apply -f recommended.yaml**

nano dashboard-admin-bind-cluster-role.yaml 
                                                 
**apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: dashboard-admin-bind-cluster-role
  labels:
    k8s-app: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
-kind: ServiceAccount
  name: dashboard-admin
  namespace: kubernetes-dashboard**     

**kubectl apply -f dashboard-admin-bind-cluster-role.yaml
kubectl create serviceaccount dashboard-admin -n kubernetes-dashboard
kubectl create clusterrolebinding dashboard-admin --clusterrole=cluster-admin --serviceaccount=kubernetes-dashboard:dashboard-admin
kubectl -n kubernetes-dashboard create token dashboard-admin**
#From Browser 192.168.10.161:3001 login and add token#
                                                 
**7.** Install Helm 
**wget https://get.helm.sh/helm-v3.8.0-linux-amd64.tar.gz**
tar -zxvf helm*.tar.gz
**sudo cp /home/dev/helm/linux-amd64/helm /usr/local/bin/helm** 
                                                 
**8.** Install openebs (to create PVC, local disk is used on physical and virtual servers.. storage plugin (addition) example..
**helm repo add openebs https://openebs.github.io/charts 
	helm repo update
	helm install openebs --namespace openebs openebs/openebs --create-namespace
	kubectl get storageclass
	kubectl patch storageclass openebs-hostpath -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'**                                     
                                                 
**9. Nginx Install**
                                                 
 ![ng.png](/kubernetes/ng.png)  
Create Namespace                                                 
**kubectl create ns ingress-nginx** 
**kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.2/deploy/static/provider/cloud/deploy.yaml -n ingress-nginx**
                                                
**10.Cert Manager Install**
                                                 
![cert.png](/kubernetes/cert.png)
**kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.0.3/cert-manager.yaml**                                                 
                                                 
  
                                                 
                                                
                                                 
                                                 
                                                 
                                                 
                                                 
                                                 
                                                 
                                                 
                                              


