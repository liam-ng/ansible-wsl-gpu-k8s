# Feature

Ansible playbooks to provision a single-node, GPU-enabled Kubernetes cluster on WSL2 using Infrastructure as Code.
- Easy to deploy k8s GPU workload in 3 stages.
- Auto detection for different system architecture (amd64 / arm64)
- Highly customizable ansible yml files for your homelab or application.


### Components
- WSL2 (ubuntu-26.04): Linux environment on Windows that hosts the cluster nodes and GPU passthrough.
- Nvidia: GPU drivers and container toolkit that expose NVIDIA GPUs to containers and Kubernetes.
- Ansible: Automation engine that provisions the host, installs components, and configures the cluster.
- Kubernetes (k8s): Container orchestration platform for running and scheduling GPU workloads.
- Helm: Package manager used to deploy Calico and the NVIDIA device plugin as charts.
- Calico: Container network interface (CNI) that provides pod networking for the cluster.

# Pre-requisites
- WSL2 and ubuntu installed

```powershell
# Install WSL2
wsl --install

# Install Ubuntu-26.04 or a distro of your choice for your WSL2
wsl.exe --list --online
wsl.exe --install -d Ubuntu-26.04
```

# Quickstart
Run following commands within WSL

```bash
# Install Ansible for deployment / configurations
sudo apt update
sudo apt install -y ansible

# Clone ansible infra files
mkdir ~/projects
cd ~/projects
rm -rf ~/projects/ansible-wsl-gpu-k8s
git clone https://github.com/liam-ng/ansible-wsl-gpu-k8s.git
cd ansible-wsl-gpu-k8s

# Step 1: Bootstrap host, Kubernetes, and Nvidia GPU components
sudo ansible-playbook ansible/playbooks/10-host-provision.yml
# Step 2: Install kubeadm, kubelet, kubectl & init 
sudo ansible-playbook ansible/playbooks/20-kubernetes-bootstrap.yml
# Step 3: Deploy Calico CNI & nvidia device plugin via Helm 
sudo ansible-playbook ansible/playbooks/30-helm-deployment.yml
```


# Project Structure

You can revise project files from top-down approach.

Naming convention:

Playbook / Task: `<stage#>-<stage name>.yml`

    Stage number increment by 10 for unforeseen addition in between stages.

Role: `<playbook#>-<order>-<component>`

```
ansible-wsl-gpu-k8s/
├── ansible.cfg                         ← project settings (inventory path, roles path)
├── versions.yml                        ← version pins (loaded by playbooks)
└── ansible/
    ├── inventories/                    ← WHO to run against (localhost/WSL)
    │   ├── localhost.yml
    │   └── group_vars/                 ← variables shared by all hosts
    ├── playbooks/                      ← WHAT to run, in WHAT order
    │   ├── 10-host-provision.yml       ← Contains `common`, `containerd`, `nvidia_container_toolkit` roles
    │   ├── 20-kubernetes-bootstrap.yml ← Contains `kubernetes` role
    │   └── 30-helm-deployment.yml      ← Contains `helm`, `calico`, `nvidia_device_plugin` roles
    └── roles/                          ← HOW each component is installed
        └── <role>/
            ├── defaults/               ← overridable role defaults
            ├── handlers/               ← service restart/reload actions when the config task reports a change
            ├── tasks/                  ← steps to execute, split by responsibility
            ├── templates/              ← Jinja2-managed configuration files
            └── vars/                   ← role-specific static values, e.g. Helm value files
```

## Detailed Ansible Work Flow
```
WSL2 (Ubuntu 26.04)
    │
    ▼
ansible-playbook (localhost inventory, 3 stages)
    │
    ├── Stage 1: ansible/playbooks/10-host-provision.yml
    │      ├── 10-1-common
    │      │      ├── disable swap and remove from fstab
    │      │      ├── install base packages
    │      │      ├── load kernel modules (overlay, br_netfilter)
    │      │      └── apply Kubernetes sysctl settings
    │      ├── 10-2-containerd
    │      │      ├── add Docker/containerd apt repository
    │      │      ├── install containerd (version pinned, held)
    │      │      ├── configure containerd (SystemdCgroup)
    │      │      └── verify containerd service is active
    │      └── 10-3-nvidia_container_toolkit
    │             ├── add NVIDIA Container Toolkit apt repository
    │             ├── install pinned packages:
    │             │      nvidia-container-toolkit
    │             │      nvidia-container-toolkit-base
    │             │      libnvidia-container-tools
    │             │      libnvidia-container1
    │             ├── configure containerd NVIDIA runtime (nvidia-ctk)
    │             └── verify with nvidia-container-cli info
    │
    ├── Stage 2: ansible/playbooks/20-kubernetes-bootstrap.yml
    │      └── 20-1-kubernetes
    │             ├── add Kubernetes apt repository
    │             ├── install kubeadm, kubelet, kubectl (version pinned, held)
    │             ├── kubeadm init (skipped if cluster already initialized)
    │             ├── configure kubeconfig for the current user
    │             └── verify API is reachable; remove control-plane taint (single-node)
    │
    └── Stage 3: ansible/playbooks/30-helm-deployment.yml
           ├── 30-1-helm
           │      ├── install Helm CLI (version pinned)
           │      └── verify helm version
           ├── 30-2-calico
           │      ├── add Tigera/Calico Helm repository
           │      ├── apply Calico CRDs (helm template | kubectl apply)
           │      ├── helm upgrade --install Calico (skipped if release exists)
           │      └── verify Calico pods ready and node is Ready
           └── 30-3-nvidia_device_plugin
                  ├── add NVIDIA device plugin Helm repository
                  ├── helm upgrade --install device plugin (skipped if release exists)
                  ├── label node nvidia.com/gpu.present=true
                  └── verify daemonset, GPU resource (nvidia.com/gpu), and GPU test pod
```