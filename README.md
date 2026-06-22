# Jenkins Agent Node Setup (RHEL 10+)
Guide including kernel modules, sysctl, firewalld, and SELinux configurations required for RHEL 10+.

---

## Part 1: Jenkins Master / Controller
*Run these commands on your **Jenkins Master** to generate the SSH key.*

### 1. Generate SSH Key
```bash
# Create .ssh directory
sudo mkdir -p /var/lib/jenkins/.ssh
sudo chown jenkins:jenkins /var/lib/jenkins/.ssh
sudo chmod 700 /var/lib/jenkins/.ssh

# Generate the key directly as the jenkins user
sudo -u jenkins ssh-keygen -t ed25519 -f /var/lib/jenkins/.ssh/id_ed25519 -N ""

# Display the public key
sudo cat /var/lib/jenkins/.ssh/id_ed25519.pub
```
> **Action:** Copy the entire output (starts with `ssh-ed25519 AAAA...`) for use in Part 2, Step 4.

---

## Part 2: RHEL 10 Agent Node
*Run these commands on your **new RHEL 10 VM**.*

### 1. System Prep & Kernel Modules
```bash
# Update and install prerequisites (including extra kernel modules)
sudo dnf update -y
sudo dnf install -y nano git wget curl dnf-plugins-core kernel-modules-extra

# REBOOT REQUIRED to load the new kernel modules
sudo reboot
```
*(Log back in after the reboot completes)*

```bash
# Load required kernel modules
sudo modprobe overlay
sudo modprobe br_netfilter

# Make kernel modules persistent
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

### 2. Firewalld & SELinux Configuration
```bash
# Allow SSH
sudo firewall-cmd --permanent --add-service=ssh

# Uncomment below ONLY if this node will host K8s workloads/services
# sudo firewall-cmd --permanent --add-port=6443/tcp       
# sudo firewall-cmd --permanent --add-port=30000-32767/tcp 
# sudo firewall-cmd --permanent --add-port=10250/tcp      

sudo firewall-cmd --reload

# SELinux: Allow containers to manage cgroups (Required for Jenkins Docker pipelines)
sudo setsebool -P container_manage_cgroup true
```

### 3. Install Docker, Trivy, and kubectl
```bash
# Install Docker
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker

# Install Trivy
sudo tee /etc/yum.repos.d/trivy.repo << 'EOF'
[trivy]
name=Trivy repository
baseurl=https://aquasecurity.github.io/trivy-repo/rpm/releases/$basearch/
gpgcheck=1
enabled=1
gpgkey=https://aquasecurity.github.io/trivy-repo/rpm/public.key
EOF
sudo dnf install -y trivy

# Install kubectl
sudo tee /etc/yum.repos.d/kubernetes.repo <<'EOF'
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.32/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.32/rpm/repodata/repomd.xml.key
EOF
sudo dnf install -y kubectl
```

### 4. Configure Jenkins User & SSH Access
```bash
# Create user and add to docker group
sudo useradd -m -s /bin/bash jenkins
sudo usermod -aG docker jenkins

# Fix home directory permissions (Crucial for SSH to accept keys)
sudo chmod 755 /home/jenkins

# Setup SSH directory
sudo mkdir -p /home/jenkins/.ssh
sudo chmod 700 /home/jenkins/.ssh

# Open the file in nano to add your public key
sudo nano /home/jenkins/.ssh/authorized_keys
```
> **Inside nano:** Paste the public key copied in Part 1. 
> Save: `Ctrl+O`, `Enter`. Exit: `Ctrl+X`.

```bash
# Fix permissions, ownership, and SELinux context
sudo chmod 600 /home/jenkins/.ssh/authorized_keys
sudo chown -R jenkins:jenkins /home/jenkins/.ssh
sudo restorecon -Rv /home/jenkins/.ssh
```

### 5. Final Verification
```bash
# Test tools as the jenkins user
sudo -u jenkins docker ps
sudo -u jenkins trivy --version
sudo -u jenkins kubectl version --client
sudo -u jenkins git --version
```
