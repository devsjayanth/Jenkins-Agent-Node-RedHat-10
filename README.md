# Jenkins-Agent-Node-RedHat10
Guide including kernel modules, sysctl, firewalld, and SELinux configurations required for RHEL 10+.

### 1. Generate SSH Key on Jenkins Master
*Run on your **Jenkins Master/Controller**.*

```bash
sudo su - jenkins
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N ""
cat ~/.ssh/id_ed25519.pub
```
*(Copy the output starting with `ssh-ed25519 AAAA...` for Step 7).*

---

### 2. System Prep & Kernel Modules (RHEL Node)
*Run on your **Agent Node**.*

```bash
# Update and install prerequisites
sudo dnf update -y
sudo dnf install -y nano git wget curl dnf-plugins-core

# Load required kernel modules immediately
sudo modprobe overlay
sudo modprobe br_netfilter

# Make kernel modules persistent across reboots
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

# Configure sysctl parameters for networking
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

# Apply sysctl changes
sudo sysctl --system
```

### 3. Firewalld & SELinux Configuration
```bash
# Ensure SSH is allowed (usually default, but explicit is better)
sudo firewall-cmd --permanent --add-service=ssh

# Uncomment below if this node will host K8s workloads/services
# sudo firewall-cmd --permanent --add-port=6443/tcp       # K8s API
# sudo firewall-cmd --permanent --add-port=30000-32767/tcp # K8s NodePorts
# sudo firewall-cmd --permanent --add-port=10250/tcp      # Kubelet API

sudo firewall-cmd --reload

# SELinux: Allow containers to manage cgroups (Required for Jenkins Docker pipelines)
sudo setsebool -P container_manage_cgroup true
```

### 4. Install Docker
```bash
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker
```

### 5. Install Trivy
```bash
sudo tee /etc/yum.repos.d/trivy.repo << 'EOF'
[trivy]
name=Trivy repository
baseurl=https://aquasecurity.github.io/trivy-repo/rpm/releases/$basearch/
gpgcheck=1
enabled=1
gpgkey=https://aquasecurity.github.io/trivy-repo/rpm/public.key
EOF
sudo dnf install -y trivy
```

### 6. Install kubectl
```bash
sudo tee /etc/yum.repos.d/kubernetes.repo << 'EOF'
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.31/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.31/rpm/repodata/repomd.xml.key
EOF
sudo dnf install -y kubectl
```

### 7. Configure Jenkins User & SSH Access
```bash
# Create user and add to docker group
sudo useradd -m -s /bin/bash jenkins
sudo usermod -aG docker jenkins

# Setup SSH directory
sudo mkdir -p /home/jenkins/.ssh
sudo chmod 700 /home/jenkins/.ssh

# Open the file in nano to add your public key
sudo nano /home/jenkins/.ssh/authorized_keys
```
> **Inside nano:** Paste the public key copied in Step 1. 
> Save: `Ctrl+O`, `Enter`. Exit: `Ctrl+X`.

```bash
# Fix permissions and ownership
sudo chmod 600 /home/jenkins/.ssh/authorized_keys
sudo chown -R jenkins:jenkins /home/jenkins/.ssh
```

### 8. Final Verification
```bash
sudo su - jenkins
docker ps
trivy --version
kubectl version --client
git --version
exit
```
