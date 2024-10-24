# Comprehensive Guide to Setting Up a Kubernetes Cluster with Master and Worker Nodes

This guide provides a step-by-step walkthrough for setting up a Kubernetes cluster, including the creation of a master node and connecting worker nodes to the cluster. The guide includes the use of provided bash scripts (`cleanup.sh` and `k8sSetup.sh`) and additional commands to ensure proper configuration.

## Prerequisites

1. **Ubuntu 24.04 LTS** installed on all machines.
2. **Root or sudo access** on all machines.
3. **Internet connection** for downloading necessary packages and images.
4. **SSH access** between the master and worker nodes.

## 1. Cleanup Existing Kubernetes Setup

Before setting up a new Kubernetes cluster, it is crucial to clean up any existing setup to avoid conflicts. The provided `cleanup.sh` script handles this.

### Steps to Cleanup

1. Create a directory:
   ```bash
   mkdir -p ~/k8s_setup
   cd ~/k8s_setup
   ```

2. Create and edit the `cleanup.sh` script:
   ```bash
   sudo nano cleanup.sh
   ```

3. Copy and paste the code below into the file:

   ```bash
   #!/bin/bash

   # Define the ports that need to be checked
   PORTS=(10259 10257 10250)

   # Function to kill processes using specific ports
   kill_process_on_ports() {
       for port in "${PORTS[@]}"; do
           echo "Checking for processes using port $port..."
           PIDS=$(sudo lsof -t -i :$port)
           if [ -n "$PIDS" ]; then
               echo "Killing processes using port $port: $PIDS"
               sudo kill -9 $PIDS
               echo "Processes killed successfully."
           else
               echo "No processes found using port $port."
           fi
       done
   }

   # Function to remove existing Kubernetes manifest files
   remove_kubernetes_manifests() {
       MANIFESTS=(
           "/etc/kubernetes/manifests/kube-apiserver.yaml"
           "/etc/kubernetes/manifests/kube-controller-manager.yaml"
           "/etc/kubernetes/manifests/kube-scheduler.yaml"
           "/etc/kubernetes/manifests/etcd.yaml"
       )

       for manifest in "${MANIFESTS[@]}"; do
           if [ -f "$manifest" ]; then
               echo "Removing file: $manifest"
               sudo rm -f "$manifest"
               echo "File removed: $manifest"
           else
               echo "File not found: $manifest"
           fi
       done
   }

   # Function to handle existing Kubernetes configuration files
   handle_existing_k8s_files() {
       FILES=(
           "/etc/kubernetes/kubelet.conf"
           "/etc/kubernetes/pki/ca.crt"
       )

       for file in "${FILES[@]}"; do
           if [ -f "$file" ]; then
               echo "File already exists: $file"
               echo "Backing up $file to $file.bak"
               sudo mv "$file" "$file.bak"
               echo "Backup created for $file"
           else
               echo "File not found: $file"
           fi
       done
   }

   # Function to reset Kubernetes and clear iptables
   reset_kubernetes_and_iptables() {
       echo "Resetting Kubernetes..."
       sudo kubeadm reset -f
       echo "Kubernetes reset complete."

       echo "Flushing iptables rules..."
       sudo iptables -F
       echo "iptables rules flushed."
   }

   # Main script execution
   echo "Starting Kubernetes cleanup script..."
   kill_process_on_ports
   remove_kubernetes_manifests
   handle_existing_k8s_files
   reset_kubernetes_and_iptables
   echo "Kubernetes cleanup complete."
   ```

4. Make the script executable:
   ```bash
   chmod +x cleanup.sh
   ```

5. Execute the cleanup script:
   ```bash
   sudo ./cleanup.sh
   ```

## 2. Set Up the Master Node

The `k8sSetup.sh` script is used to install Docker, containerd, Kubernetes tools, and set up the master node.

### Steps to Set Up the Master Node

1. Create and edit the `k8sSetup.sh` script:
   ```bash
   sudo nano k8sSetup.sh
   ```

2. Copy and paste the code below into the file:

   ```bash
   #!/bin/bash

   # Purpose: Set up Docker and Kubernetes on a machine, and join it to an existing Kubernetes cluster.

   # OS = Ubuntu 24.04 LTS

   # Function to get the machine's IP address
   get_machine_ip() {
       IP_ADDRESS=$(hostname -I | awk '{print $1}')
       if [ -z "$IP_ADDRESS" ]; then
           read -p "Could not automatically determine the IP address. Please enter your machine's IP address: " IP_ADDRESS
       else
           read -p "Detected IP address is $IP_ADDRESS. Press Enter to use this IP or enter a different one: " input_ip
           if [ ! -z "$input_ip" ]; then
               IP_ADDRESS=$input_ip
           fi
       fi
   }

   # Function to check if a command is installed and return its version
   check_installed() {
       local cmd=$1
       if command -v $cmd &> /dev/null; then
           echo "$cmd is already installed. Version: $($cmd --version | head -n 1)"
           return 0
       else
           return 1
       fi
   }

   # Function to install Docker and containerd
   install_docker_and_containerd() {
       if check_installed docker && check_installed containerd; then
           return
       fi

       echo "Installing Docker and containerd..."
       sudo apt-get update
       sudo apt-get install -y ca-certificates curl gnupg

       sudo install -m 0755 -d /etc/apt/keyrings
       curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

       echo "deb [arch=\"$(dpkg --print-architecture)\" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo \"$VERSION_CODENAME\") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

       sudo apt-get update
       sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

       # Configure containerd
       sudo mkdir -p /etc/containerd
       containerd config default | sudo tee /etc/containerd/config.toml
       sudo sed -i '/SystemdCgroup =/c\SystemdCgroup = true' /etc/containerd/config.toml
       sudo sed -i '/sandbox_image =/c\sandbox_image = "registry.k8s.io/pause:3.8"' /etc/containerd/config.toml

       # Restart containerd to apply changes
       sudo systemctl restart containerd
       echo "Docker and containerd installed successfully!"
       docker --version
       containerd --version
   }

   # Function to install kubeadm, kubelet, kubectl
   install_kubernetes_tools() {
       local tools_installed=true

       for tool in kubeadm kubelet kubectl; do
           if ! check_installed $tool; then
               tools_installed=false
           fi
       done

       if $tools_installed; then
           return
       fi

       echo "Setting up Kubernetes repository..."
       sudo apt-get update
       sudo apt-get install -y apt-transport-https ca-certificates curl

       # Add Kubernetes apt repository
       echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

       # Retrieve and add the GPG key for the Kubernetes repository
       sudo mkdir -p /etc/apt/keyrings
       curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

       # Update system again to reflect new repository
       sudo apt-get update

       echo "Installing kubeadm, kubelet, and kubectl..."
       sudo apt-get install -y kubelet kubeadm kubectl
       sudo apt-mark hold kubelet kubeadm kubectl
       echo "kubeadm, kubelet, and kubectl installed successfully!"
       kubeadm version
       kubelet --version
       kubectl version --client
   }

   # Function to disable swap
   disable_swap() {
       echo "Disabling swap..."
       sudo swapoff -a
       sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
       echo "Swap disabled!"
   }

   # Function to configure sysctl settings
   configure_sysctl() {
       echo "Configuring sysctl settings for Kubernetes..."
       sudo modprobe overlay
       sudo modprobe br_netfilter

       # Ensure sysctl params are set
       sudo tee /etc/sysctl.d/k8s.conf <<

EOF
       net.bridge.bridge-nf-call-iptables = 1
       net.bridge.bridge-nf-call-ip6tables = 1
       net.ipv4.ip_forward = 1
       EOF

       sudo sysctl --system
       echo "Sysctl settings configured!"
   }

   # Function to initialize Kubernetes cluster
   initialize_kubernetes_cluster() {
       echo "Initializing Kubernetes cluster with IP address: $IP_ADDRESS..."
       sudo kubeadm init --apiserver-advertise-address=$IP_ADDRESS --pod-network-cidr=192.168.0.0/16
       echo "Kubernetes cluster initialized successfully!"

       # Set up local kubeconfig for non-root user
       mkdir -p $HOME/.kube
       sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
       sudo chown $(id -u):$(id -g) $HOME/.kube/config
       echo "Kubeconfig file set up successfully!"
   }

   # Main script execution
   echo "Starting Kubernetes setup script..."
   get_machine_ip
   install_docker_and_containerd
   install_kubernetes_tools
   disable_swap
   configure_sysctl
   initialize_kubernetes_cluster
   echo "Kubernetes setup complete."
   ```

3. Make the script executable:
   ```bash
   chmod +x k8sSetup.sh
   ```

4. Execute the setup script:
   ```bash
   sudo ./k8sSetup.sh
   ```

## 3. Install Calico Network Plugin

Once the master node is set up, you will need to install a network plugin. For this guide, we will use Calico.

### Steps to Install Calico

1. Run the following command on the master node:
   ```bash
   kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
   ```

## 4. Join Worker Nodes to the Cluster

To join worker nodes, you will need the command generated during the master node setup. It looks something like this:

```bash
kubeadm join <master-ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

### Steps to Join Worker Nodes

1. Execute the `k8sSetup.sh` script on each worker node as described in Step 2.
2. After the worker nodes are set up, execute the join command provided by the master node.

## 5. Verify Cluster Setup

After joining the worker nodes, verify that the nodes are correctly added to the cluster.

### Check Node Status

On the master node, run:
```bash
kubectl get nodes
```

You should see all nodes listed with their statuses.

## 6. Additional Configuration

### Set Up the Load Balancer (Optional)

If you need to set up a load balancer for your applications, consider using MetalLB.

### Enable Dashboard (Optional)

You can enable the Kubernetes dashboard for a visual representation of your cluster:
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
```
