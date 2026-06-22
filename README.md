# Jenkins Agent Node Setup (RHEL 10+)
Guide including kernel modules, sysctl, firewalld, and SELinux configurations required for RHEL 10+.

---

## Part 1: Jenkins Master / Controller
*Run these commands on your **Jenkins Master** to generate the SSH key.*

Here is how to switch to a 4096-bit RSA key. 

*(Note: RHEL 10 requires RSA keys to be at least 2048 bits and use SHA-2, so we will use 4096 bits to ensure it works perfectly).*

### 1. Generate RSA Key on Jenkins Master
*Run on your **Jenkins Master**.*

```bash
# 1. Remove the old Ed25519 key to avoid confusion
sudo rm -f /var/lib/jenkins/.ssh/id_ed25519 /var/lib/jenkins/.ssh/id_ed25519.pub

# 2. Generate a 4096-bit RSA key
sudo -u jenkins ssh-keygen -t rsa -b 4096 -f /var/lib/jenkins/.ssh/id_rsa -N ""

# 3. Display the new public key
sudo cat /var/lib/jenkins/.ssh/id_rsa.pub
```
*(Copy the output starting with `ssh-rsa AAAA...`)*

---

### 2. Update the Agent Node
*Run on your **Agent Node**.*

```bash
# Open the authorized_keys file
sudo nano /home/jenkins/.ssh/authorized_keys
```
> **Inside nano:** Delete the old `ssh-ed25519` line and paste the new `ssh-rsa` line you just copied.
> Save: `Ctrl+O`, `Enter`. Exit: `Ctrl+X`.

*(The permissions and SELinux contexts we fixed earlier are still correct, so no need to change them).*

---

### 3. Test the Connection
*Run on your **Jenkins Master**.*

**CRITICAL:** Make sure you use the **Agent's actual IP address** this time, not the Master's IP! (Check the agent's IP by running `ip a` on the agent).

```bash
# Replace <AGENT_IP> with the actual IP of your agent node (e.g., 10.0.1.11)
sudo -u jenkins ssh jenkins@<AGENT_IP>
```

If it connects without asking for a password, you are good to go! Type `exit` to disconnect.

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
---
#Add your new RHEL 10 agent to the Jenkins UI.

### 1. Get the Private Key (Run on Jenkins Master)
Jenkins needs the **private** key to connect to the agent. Run this on your master to copy it:
```bash
sudo cat /var/lib/jenkins/.ssh/id_rsa
```
*(Copy the entire output, including `-----BEGIN OPENSSH PRIVATE KEY-----` and `-----END OPENSSH PRIVATE KEY-----`)*

---

### 2. Add SSH Credentials in Jenkins UI
1. Go to **Manage Jenkins** -> **Credentials**.
2. Click on **(global)** (or `System` -> `Global credentials`).
3. Click **Add Credentials** on the left.
4. Fill in the details:
   * **Kind:** `SSH Username with private key`
   * **Scope:** `Global`
   * **ID:** `jenkins-rhel10-agent` (or any name you prefer)
   * **Username:** `jenkins`
   * **Private Key:** Select **Enter directly**, click **Add**, and paste the private key you copied in Step 1.
5. Click **Create**.

---

### 3. Create the New Node
1. Go to **Manage Jenkins** -> **Nodes**.
2. Click **New Node** on the left.
3. **Node name:** Enter a name (e.g., `rhel10-docker-agent`).
4. **Type:** Select **Permanent Agent** and click **Create**.

---

### 4. Configure the Node
Fill in the configuration page:

* **Description:** `RHEL 10 Agent with Docker, Trivy, and kubectl`
* **Number of executors:** `2` (or however many concurrent jobs you want)
* **Remote root directory:** `/home/jenkins`
* **Labels:** `docker trivy kubectl rhel10` (Use these in your Jenkinsfiles to target this node)
* **Usage:** Select **Use this node as much as possible** (or "Only build jobs with label expressions" if you prefer).
* **Launch method:** Select **Launch agents via SSH**.
  * **Host:** Enter the **Agent's IP address** (e.g., `10.0.1.11`).
  * **Credentials:** Select the `jenkins-rhel10-agent` credential you created in Step 2.
  * **Host Key Verification Strategy:** Select **Non verifying Verification Strategy** (This prevents SSH host key prompts from blocking the connection).
5. Scroll to the bottom and click **Save**.

---

### 5. Launch the Node
1. You will be taken back to the Nodes page. The new node will likely say "Disconnected" or "Launch agent".
2. Click on the node name, then click **Launch agent** (or click the "Relaunch" button if it failed initially).
3. Watch the console output. If it ends with `Agent successfully connected and online`, your setup is complete!
