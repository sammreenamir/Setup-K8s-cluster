**Comprehensive Guide to Setting Up a Kubernetes Cluster with Master and
Worker Nodes**

This guide provides a step-by-step walkthrough for setting up a
Kubernetes cluster, including the

creation of a master node and connecting worker nodes to the cluster.
The guide includes the use of

provided bash scripts (cleanup.sh and k8sSetup.sh) and additional
commands to ensure

proper configuration.

**Prerequisites**

> 1\. **Ubuntu 24.04 LTS** installed on all machines.
>
> 2\. **Root or sudo access** on all machines.
>
> 3\. **Internet connection** for downloading necessary packages and
> images.
>
> 4\. **SSH access** between the master and worker nodes.

**1. Cleanup Existing Kubernetes Setup**

Before setting up a new Kubernetes cluster, it is crucial to clean up
any existing setup to avoid

conflicts. The provided cleanup.sh script handles this.

> 1\. Make directory
>
> 2\. sudo nano cleanup.sh
>
> 3\. copy the code and paste it into file
>
> 4\. chmod +x cleanup.sh
>
> 5\. sudo ./cleanup.sh

**cleanup.sh**

#!/bin/bash

\# Define the ports that need to be checked\
PORTS=(10259 10257 10250)

\# Function to kill processes using specific ports\
kill_process_on_ports() {\
for port in \"\${PORTS\[@\]}\"; do\
echo \"Checking for processes using port \$port\...\" PIDS=\$(sudo lsof
-t -i :\$port)\
if \[ -n \"\$PIDS\" \]; then\
echo \"Killing processes using port \$port: \$PIDS\" sudo kill -9
\$PIDS\
echo \"Processes killed successfully.\"\
else\
echo \"No processes found using port \$port.\" fi\
done\
}

\# Function to remove existing Kubernetes manifest files

remove_kubernetes_manifests() {\
MANIFESTS=(\
\"/etc/kubernetes/manifests/kube-apiserver.yaml\"\
\"/etc/kubernetes/manifests/kube-controller-manager.yaml\"
\"/etc/kubernetes/manifests/kube-scheduler.yaml\"\
\"/etc/kubernetes/manifests/etcd.yaml\"\
)

for manifest in \"\${MANIFESTS\[@\]}\"; do\
if \[ -f \"\$manifest\" \]; then\
echo \"Removing file: \$manifest\"\
sudo rm -f \"\$manifest\"\
echo \"File removed: \$manifest\"\
else\
echo \"File not found: \$manifest\"\
fi\
done\
}

\# Function to handle existing Kubernetes configuration files
handle_existing_k8s_files() {\
FILES=(\
\"/etc/kubernetes/kubelet.conf\"\
\"/etc/kubernetes/pki/ca.crt\"\
)

for file in \"\${FILES\[@\]}\"; do\
if \[ -f \"\$file\" \]; then\
echo \"File already exists: \$file\"\
echo \"Backing up \$file to \$file.bak\"\
sudo mv \"\$file\" \"\$file.bak\"\
echo \"Backup created for \$file\"\
else\
echo \"File not found: \$file\"\
fi\
done\
}

\# Function to reset Kubernetes and clear iptables
reset_kubernetes_and_iptables() {\
echo \"Resetting Kubernetes\...\"\
sudo kubeadm reset -f\
echo \"Kubernetes reset complete.\"

echo \"Flushing iptables rules\...\"\
sudo iptables -F\
echo \"iptables rules flushed.\"\
}

\# Main script execution\
echo \"Starting Kubernetes cleanup script\...\"\
kill_process_on_ports

remove_kubernetes_manifests\
handle_existing_k8s_files\
reset_kubernetes_and_iptables\
echo \"Kubernetes cleanup complete.\"

**2. Set Up the Master Node:**

The k8sSetup.sh script is used to install Docker, containerd, Kubernetes
tools, and set up the

master node.

> 1\. Make directory
>
> 2\. sudo nano k8sSetup.sh
>
> 3\. copy the code and paste it into file
>
> 4\. chmod +x k8sSetup.sh
>
> 5\. sudo ./k8sSetup.sh

**k8sSetup.sh**

#!/bin/bash

\# Purpose: Set up Docker and Kubernetes on a machine, and join it to an
existing Kubernetes cluster.

\# OS = Ubuntu 24.04 LTS

\# Function to get the machine\'s IP address\
get_machine_ip() {\
IP_ADDRESS=\$(hostname -I \| awk \'{print \$1}\')\
if \[ -z \"\$IP_ADDRESS\" \]; then\
read -p \"Could not automatically determine the IP address. Please enter
your machine\'s IP address: \" IP_ADDRESS\
else\
read -p \"Detected IP address is \$IP_ADDRESS. Press Enter to use this
IP or enter a different one: \" input_ip\
if \[ ! -z \"\$input_ip\" \]; then\
IP_ADDRESS=\$input_ip\
fi\
fi\
}

\# Function to check if a command is installed and return its version\
check_installed() {\
local cmd=\$1\
if command -v \$cmd &\> /dev/null; then\
echo \"\$cmd is already installed. Version: \$(\$cmd \--version \| head
-n 1)\" return 0\
else\
return 1\
fi

}

\# Function to install Docker and containerd\
install_docker_and_containerd() {\
if check_installed docker && check_installed containerd; then return\
fi

> echo \"Installing Docker and containerd\...\"\
> sudo apt-get update\
> sudo apt-get install -y ca-certificates curl gnupg

sudo install -m 0755 -d /etc/apt/keyrings\
curl -fsSL https://download.docker.com/linux/ubuntu/gpg \| sudo gpg
\--dearmor -o /etc/apt/keyrings/docker.gpg

echo \\\
\"deb \[arch=\\\"\$(dpkg \--print-architecture)\\\"
signed-by=/etc/apt/keyrings/docker.gpg\]
https://download.docker.com/linux/ubuntu \\\
\$(. /etc/os-release && echo \\\"\$VERSION_CODENAME\\\") stable\" \| \\\
sudo tee /etc/apt/sources.list.d/docker.list \> /dev/null

sudo apt-get update\
sudo apt-get install -y docker-ce docker-ce-cli containerd.io
docker-buildx-plugin docker-compose-plugin

\# Configure containerd\
sudo mkdir -p /etc/containerd\
containerd config default \| sudo tee /etc/containerd/config.toml\
sudo sed -i \'/SystemdCgroup =/c\\SystemdCgroup = true\'
/etc/containerd/config.toml sudo sed -i \'/sandbox_image
=/c\\sandbox_image = \"registry.k8s.io/pause:3.8\"\'
/etc/containerd/config.toml

\# Restart containerd to apply changes\
sudo systemctl restart containerd\
echo \"Docker and containerd installed successfully!\" docker
\--version\
containerd \--version\
}

\# Function to install kubeadm, kubelet, kubectl\
install_kubernetes_tools() {\
local tools_installed=true

> for tool in kubeadm kubelet kubectl; do\
> if ! check_installed \$tool; then\
> tools_installed=false\
> fi\
> done
>
> if \$tools_installed; then\
> return
>
> fi
>
> echo \"Setting up Kubernetes repository\...\"\
> sudo apt-get update\
> sudo apt-get install -y apt-transport-https ca-certificates curl

\# Add Kubernetes apt repository\
echo \'deb \[signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg\]\
https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /\' \| sudo tee
/etc/apt/sources.list.d/kubernetes.list

\# Retrieve and add the GPG key for the Kubernetes repository\
sudo mkdir -p /etc/apt/keyrings\
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key \|
sudo gpg \--dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

> \# Update system again to reflect new repository\
> sudo apt-get update

echo \"Installing kubeadm, kubelet, and kubectl\...\"\
sudo apt-get install -y kubelet kubeadm kubectl\
sudo apt-mark hold kubelet kubeadm kubectl\
echo \"kubeadm, kubelet, and kubectl installed successfully!\" kubeadm
version\
kubelet \--version\
kubectl version \--client\
}

\# Function to disable swap\
disable_swap() {\
echo \"Disabling swap\...\"\
sudo swapoff -a\
sudo sed -i \'/ swap / s/\^\\(.\*\\)\$/#\\1/g\' /etc/fstab\
echo \"Swap disabled!\"\
}

\# Function to configure sysctl settings\
configure_sysctl() {\
echo \"Configuring sysctl settings for Kubernetes\...\" sudo modprobe
overlay\
sudo modprobe br_netfilter

\# Ensure sysctl params are set\
sudo tee /etc/sysctl.d/k8s.conf\<\<EOF\
net.bridge.bridge-nf-call-ip6tables = 1\
net.bridge.bridge-nf-call-iptables = 1\
net.ipv4.ip_forward = 1\
EOF

\# Apply sysctl params without reboot\
sudo sysctl \--system\
echo \"Sysctl configured!\"\
}

\# Function to initialize Kubernetes control plane\
initialize_control_plane() {\
echo \"Initializing Kubernetes control plane with IP address
\$IP_ADDRESS\...\" sudo kubeadm init
\--apiserver-advertise-address=\$IP_ADDRESS
\--pod-network-cidr=192.168.0.0/16
\--cri-socket=unix:///var/run/containerd/containerd.sock }

\# Function to ensure \`admin.conf\` is set up and apply Calico
networking setup_admin_conf_and_apply_calico() {\
if \[ ! -f /etc/kubernetes/admin.conf \]; then\
echo \"admin.conf not found. Creating admin.conf\...\"\
export KUBECONFIG=/etc/kubernetes/admin.conf\
sudo kubeadm init phase kubeconfig admin\
echo \"admin.conf created.\"\
else\
echo \"admin.conf already exists.\"\
fi

> \# Applying Calico network plugin\
> echo \"Applying Calico network plugin\...\"\
> kubectl apply -f
> https://docs.projectcalico.org/v3.24/manifests/calico.yaml
> \--validate=false

\# Check if Calico pods are running\
echo \"Waiting for Calico pods to be ready\...\"\
kubectl wait \--namespace kube-system \--for=condition=ready pod -l
k8s-app=calico-node \--timeout=2m\
}

\# Function to fix cni plugin issue\
fix_cni_plugin() {\
echo \"Checking and fixing CNI plugin\...\"\
if \[ ! -d /etc/cni/net.d \]; then\
echo \"Directory /etc/cni/net.d does not exist. Creating it\...\" sudo
mkdir -p /etc/cni/net.d\
fi

echo \"CNI plugin configuration should be applied automatically by the
Calico manifest. If you still encounter issues, ensure that Calico was
applied correctly.\"\
}

\# Main function to orchestrate the setup\
main() {\
get_machine_ip\
install_docker_and_containerd\
disable_swap\
configure_sysctl\
install_kubernetes_tools\
read -p \"Do you want to set up this machine as a Kubernetes control
plane node? (yes/no): \" response\
if \[\[ \"\$response\" == \"yes\" \]\]; then\
initialize_control_plane

> setup_admin_conf_and_apply_calico\
> fix_cni_plugin

\# Export KUBECONFIG to use the admin.conf\
echo \"export KUBECONFIG=/etc/kubernetes/admin.conf\" \>\> \~/.bashrc\
source \~/.bashrc\
else\
echo \"Kubernetes tools installed. You can join this node to a cluster
using \'kubeadm join\' command.\"\
fi\
}

\# Execute the main function\
main

***After getting joining command:***

Save the kubeadm join command output at the end of the master node setup
for use on worker nodes.

> 1\. **Download and Apply the Manifest**:
>
> kubectl apply -f
>
> 2\. Sudo systemctl restart kubelet
>
> 3\. sudo systemctl restart containerd

**3. Set Up Worker Nodes**

After setting up the master node, you need to configure the worker nodes
and join them to the

master node.

**4. Worker Node Setup Script**

Again run cleanup.sh file first and then K8sSetup file and write master
IP address while running a bash file and at the end it ask for setup
control plane then write NO.

> 1\. mkdir .kube\
> 2. copy admin.conf file from master node /etc/kubernetes 3. paste it
> on worker node /etc/kubernetes\
> 4. mv /etc/kubernetes/admin.conf \~/.kube/config\
> 5. export KUBECONFIG=\~/.kube/config
>
> **5. Verify Cluster Setup**

After setting up both the master and worker nodes, verify that the nodes
are correctly connected and that the cluster is functioning as expected.

> 1\. On the master node, check the status of nodes:
>
> kubectl get nodes
>
> 2\. Verify that all nodes (master and workers) are in the Ready state.

**Conclusion**

By following these steps and using the provided scripts, you can set up
a Kubernetes cluster with one master node and multiple worker nodes.
Ensure to adapt any IP addresses and specific configurations as per your
network and infrastructure requirements.
