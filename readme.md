# Kubernetes Cluster on EC2s

## Master Node & Worker Nodes

Follow these steps on all nodes to configure the EC2 for Kubernetes, then install kubelet, kubeadm and kubectl.

### System Config

You must disable Swap for kubelet to work properly.
```bash
sudo swapoff -a 
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

Create containerd configuration file.
```bash
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
```

Load modules.
```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

Setup required sysctl params for Kubernetes networking, this ensure that iptables correctly see bridged traffic  
```bash
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
```

Apply sysctl params without reboot 
```bash
sudo sysctl --system
```

Install the containerd.io package from the official Docker repositories
```bash
sudo yum update -y
sudo yum install -y containerd
```

Create configuration directory for containerd
```bash
sudo mkdir -p /etc/containerd
```

Generate containerd configuration and save the configuration file in the created directory
```bash
containerd config default | sudo tee /etc/containerd/config.tom
```

Restart containerd to ensure it use the newly created configuration file
```bash
sudo systemctl restart containerd
```

Verify that containerd is running
```bash
sudo systemctl status containerd
```

### Kubernetes Install
Install kubeadm, kubelet and kubectl

Download the Google Cloud public signing key
```bash
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF
```

Set SELinux in permissive mode (effectively disabling it). This is required to allow containers to access the host filesystem.
```bash
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

Install kubelet, kubeadm and kubectl
```bash
sudo yum install -y kubelet-1.28.2 kubeadm-1.28.2 kubectl-1.28.2 --disableexcludes=kubernetes
```

Enable kubelet
```bash
sudo systemctl enable --now kubelet
```

## Cluster Configuration

Copy kube-config.yml to K8s master.

Initialize the Kubernetes cluster with kubeadm and passing the created config file
```bash
sudo kubeadm init --config kube-config.yml --ignore-preflight-errors=all
```

Set kubectl access
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Check the cluster nodes - the control plane will remain in 'Not Ready' state until the Calico pod networking is applied.
```bash
kubectl get nodes
```

Apply Calico pod networking configuration on the master
```bash
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

Check the cluster nodes (it could take few moments until the node become available)
```bash
kubectl get nodes
```

In the Master (Control Plane) Node, create a token by running this command.  Copy the output to the worker nodes, but prefix the auotgenerated command with sudo
```bash
kubeadm token create --print-join-command
```

In the Master (Control Plane) Node, check the cluster status (it could take few moments until the worker nodes become available)
```bash
kubectl get nodes
```

## Configure a worker node

Repeat all of the steps up to the Cluster configuation section on one of the worker nodes.

Once the installation is complete, run the output generated from the Kubeadm token create command.

Return to the control plane and re-run the following command until the worker node you have configured has joined the cluster.
```bash
kubectl get nodes
```

Repeat the process for the other worker node.


## Deploy a container to the cluster

Copy react-app-pod.yml to the naster node and apply.
```bash
kubectl create -f react-app-pod.yml
```

Verify that the pod is up and running.
```bash
kubectl get pods -o wide
```

Verify that the deployment complete.
```bash
kubectl get deployment
```

# Monitoring & Observability Tooling 
## Installing Helm

As part of this example we will start to use Helm.  It is effectively a package manager for Kubernetes, which we will use here to install Prometheus & Grafana.

To start we fetch the Helm installer script, change the ownership of it and execute.
```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

Next verify that it was installed successfully
```bash
helm version
```

Create a role to the "default" service account in a namespace 
```bash
kubectl --namespace=kube-system create clusterrolebinding add-on-cluster-admin \
  --clusterrole=cluster-admin \
  --serviceaccount=kube-system:default
```

With the helm install complete, next step is to start on the Prometheus install.

## Installing Prometheus

Add the repository to Helm configuration
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

Create yml (vi prometheus.yml) to disable persistent volume. In a real environment, you will not do this because all data will be lost when the server and AlertManager pods are terminated.
Then install Prometheus using this configuration and check to see if it is running.
```bash
helm install -f prometheus.yml prometheus prometheus-community/prometheus
kubectl get pods
```

Expose Node Port Service to access Prometheus from outside the K8s Cluster
```bash
kubectl expose service prometheus-server --type=NodePort --target-port=9090 --name=prometheus-server-np
kubectl get svc
```

Using the port number assigned to the prometheus-server-np NodePort and the public IP of the master node, you can load the Prometheus UI in your browser. 

## Using Prometheus

Once loaded, you can run a number of PromQL (Prometheus Query Language) commands to better understand what is happening on the cluster.


1. Count the number of nodes in the Kubernetes cluster
```bash
count(kubelet_node_name)
```

2. Count the number of nodes that are currently ready in the Kubernetes cluster
```bash
sum(kube_node_status_condition{condition="Ready", status="true"}==1)
```

3. Count pods per cluster by namespace
```bash
sum by (namespace) (kube_pod_info)
```

4. Count pods not ready by namespace
```bash
sum by (namespace) (kube_pod_status_ready{condition="false"})
```


# Install Grafana

Add the repository to Helm configuration
```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

Create a copy of grafana.yml on the master node.

Install Grafana by using the provided charts
```bash
helm install -f grafana.yml grafana grafana/grafana
kubectl get pods
```