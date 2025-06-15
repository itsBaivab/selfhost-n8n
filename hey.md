# How to Set Up Kubeadm on Ubuntu

This guide walks you through setting up a Kubernetes cluster using `kubeadm` on Ubuntu. We'll cover prerequisites, installation, and cluster initialization for a single-node or multi-node setup.

## Prerequisites

- **OS**: Ubuntu 20.04 or 22.04 (LTS recommended)
- **Hardware**:
    - Minimum 2 CPUs, 2GB RAM per machine
    - At least one machine for the control plane (master) and optional worker nodes
- **Network**:
    - Full network connectivity between nodes
    - Static IP addresses for each node
- **User**: Non-root user with `sudo` privileges
- **Swap**: Disabled on all nodes (Kubernetes requirement)

## Step 1: Prepare the System

Update the system and install necessary packages.

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y apt-transport-https ca-certificates curl software-properties-common

```

Disable swap to meet Kubernetes requirements.

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

```

Verify swap is disabled:

```bash
free -h

```

## Step 2: Install Container Runtime

Kubernetes requires a container runtime like Docker. Here, we'll use Docker.

### Install Docker

```bash
sudo apt install -y docker.io
sudo systemctl enable docker
sudo systemctl start docker

```

### Configure Docker for Kubernetes

Kubernetes uses `systemd` as the cgroup driver for Docker.

```bash
sudo mkdir -p /etc/docker
cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {"max-size": "100m"},
  "storage-driver": "overlay2"}
}
EOF

```

Restart Docker to apply changes:

```bash
sudo systemctl restart docker

```

## Step 3: Install Kubeadm, Kubelet, and Kubectl

Add the Kubernetes repository and install the required Kubernetes components.

### Add Kubernetes APT Repository

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release/key | sudo tee /etc/apt/keyrings/kubernetes.asc
echo "deb [signed-by=/etc/apt/keyrings/kubernetes.asc] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list

```

Update package list:

```bash
sudo apt update

```

### Install `kubeadm`, `kubelet`, and `kubectl`

```bash
sudo apt install -y kubeadm kubelet kubectl
sudo apt-mark hold kubeadm kubelet kubectl

```

Verify installation:

```bash
kubeadm version
kubectl version --client
kubelet --version

```

## Step 4: Initialize the Control Plane (Master Node Only)

Initialize the control plane on the master node.

### Configure the Control Plane

Run `kubeadm init` with a pod network CIDR (adjust for your chosen network plugin, e.g., Flannel`):

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16

```

Set up `kubectl` for the non-root user:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

```

### Install a Pod Network (CNI)

Install Flannel` as the pod network plugin:

```bash
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml

```

Verify nodes and pods:

```bash
kubectl get nodes
kubectl get pods -n kube-system

```

## Step 5: Join Worker Nodes (Optional)

On each worker node, join the cluster using the `kubeadm join` command provided at the end of `kubeadm init` output on the master node. It looks like:

```bash
sudo kubeadm join <master-ip>:<master-port> --token <token> --discovery-token-ca-cert-hash <hash>

```

If you lost the join command, generate a new one on the master node:

```bash
kubeadm token create --print-join-command

```

## Step 6: Verify the Cluster

On the master node, check the cluster status:

```bash
kubectl get nodes
kubectl get pods --all-namespaces

```

Ensure all nodes are in the `Ready` state and system pods are running.

## Troubleshooting Tips

- **Node Not Ready**: Check pod network plugin installation or firewall rules.
- **Kubeadm Init Fails**: Review logs with `journalctl -u kubelet` or `kubectl describe`.
- **CNI Issues**: Ensure the correct CIDR is used and the CNI plugin is applied.
- **API Server Unreachable**: Verify `kube-apiserver` pod status in `kube-system` namespace.

## Conclusion

You’ve set up a Kubernetes cluster using `kubeadm` on Ubuntu! You can now deploy applications, scale nodes, or explore advanced configurations like high availability. For production, consider adding more nodes, enabling backups, and securing the cluster with RBAC and network policies.

For further details, refer to the [Kubernetes Documentation](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/) or [Flannel GitHub](https://github.com/flannel-io/flannel).
