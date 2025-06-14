
# CI/CD Pipeline Setup Using Jenkins, Ansible, and Apache Web Server

## Step – 1: Four Instances Launch

1. Jenkins  
2. Ansible  
3. Web-Server  
4. Developer  

---

## Ansible Server

### Step – 2: Install Ansible
```bash
sudo apt update
sudo apt install software-properties-common -y
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install ansible -y
```

### Step – 3: Set Root User Password
```
passwd root
```

### Step – 4: Generate SSH Key Pair
```
ssh-keygen
```

### Step – 5: Enable SSH Root Login
```
sudo nano /etc/ssh/sshd_config
# Uncomment or add:
PermitRootLogin yes
PasswordAuthentication yes
```
```
sudo nano /etc/ssh/sshd_config.d/10-cloud-init.conf
# Add:
PermitRootLogin yes
PasswordAuthentication yes
```

### Step – 6: Restart SSH Service & Verify Configuration
```
sudo systemctl restart ssh
sudo sshd -T | grep -Ei 'passwordauthentication|permitrootlogin'
```

---

## Jenkins Server

### Step – 7: Create Jenkins Install Script
```
nano jenkins.sh
```

Paste the following:
```
#!/bin/bash

set -e

echo "Updating system packages..."
sudo apt update && sudo apt upgrade -y

echo "Installing Java (OpenJDK 17)..."
sudo apt install openjdk-17-jdk -y

echo "Adding Jenkins GPG key..."
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null

echo "Adding Jenkins repository..."
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

echo "Updating package list with Jenkins repo..."
sudo apt update

echo "Installing Jenkins..."
sudo apt install jenkins -y

echo "Starting and enabling Jenkins service..."
sudo systemctl start jenkins
sudo systemctl enable jenkins

echo "Allowing firewall on port 8080 (if UFW is active)..."
sudo ufw allow 8080 || true
sudo ufw reload || true

echo "Jenkins installation completed!"
echo "Access Jenkins via: http://<your-server-ip>:8080"
echo "Initial admin password:"
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

### Step – 8: Give Permission to Script
```
chmod +x jenkins.sh
```

### Step – 9: Run the Script
```
sudo ./jenkins.sh
```

### Step – 10 to 13: Root Setup and SSH Config (Same as Ansible)
```
passwd root
ssh-keygen
sudo nano /etc/ssh/sshd_config
# Edit:
PermitRootLogin yes
PasswordAuthentication yes
```
```
sudo nano /etc/ssh/sshd_config.d/10-cloud-init.conf
# Add:
PermitRootLogin yes
PasswordAuthentication yes
```
```
sudo systemctl restart ssh
sudo sshd -T | grep -Ei 'passwordauthentication|permitrootlogin'
```

---

## Web Server

### Step – 14: Install Apache
```
apt install apache2
```

### Step – 15 to 18: Root Setup and SSH Config
```
passwd root
ssh-keygen
sudo nano /etc/ssh/sshd_config
# Edit:
PermitRootLogin yes
PasswordAuthentication yes
```
```
sudo nano /etc/ssh/sshd_config.d/10-cloud-init.conf
# Add:
PermitRootLogin yes
PasswordAuthentication yes
```
```
sudo systemctl restart ssh
sudo sshd -T | grep -Ei 'passwordauthentication|permitrootlogin'
```

---

## SSH Authentication Setup

### Step – 19: Jenkins → Ansible
```
ssh-copy-id root@<ansible_private_ip>
ssh root@<ansible_private_ip>
```

### Step – 20: Ansible → Web Server
```
ssh-copy-id root@<web-server_private_ip>
ssh root@<web-server_private_ip>
```

---

## Ansible Project Setup

### Step – 21: Create and Navigate to Project Directory
```
mkdir -p /home/ubuntu/sourcecode
cd /home/ubuntu/sourcecode
```

### Step – 22: Create Inventory File
```
nano hosts
```
Content:
```
[webserver]
<web-server-private-ip> ansible_user=root
```

### Step – 23: Create Playbook
```
nano playbook.yml
```
Content:
```
- name: Deploy index.html to webserver
  hosts: webserver
  become: yes
  tasks:
    - name: Copy index.html to web root
      copy:
        src: /opt/index.html
        dest: /var/www/html/index.html
```

---

## Jenkins GitHub Integration

### Step – 24: Create API Token

- Go to: Dashboard > Your Username > Configure > API Token > Generate Token  
- Save the token securely

### Step – 25: GitHub Webhook

- Go to: GitHub Repo > Settings > Webhooks > Add webhook  
- Payload URL:  
```
http://<jenkins_ip>:8080/github-webhook/
```
- Content Type: `application/json`  
- Secret: (Optional)  
- Save

### Step – 26: Jenkins Plugin + Remote Hosts

1. Dashboard → Manage Jenkins → Plugin Manager → Install “Publish Over SSH”  
2. Configure System → Add SSH Hosts:

```
Jenkins Host:
- Name: Jenkins
- Hostname: <jenkins_private_ip>
- Username: root

Ansible Host:
- Name: ansible
- Hostname: <ansible_private_ip>
- Username: root
```

---

## Jenkins Freestyle Project

### Step – 27: Project Configuration

- **Source Code Management**  
```
Repository: https://github.com/your/repo.git
Branch: <branch-name>
```

- **Build Triggers**  
```
Check: GitHub hook trigger for GITScm polling
```

- **Build Steps**  
```
# Step 1: Send File to Ansible
rsync -avh /var/lib/jenkins/workspace/<project_name>/index.html root@<ansible_ip>:/opt/

# Step 2: Trigger Ansible Playbook
ansible-playbook -i /home/ubuntu/sourcecode/hosts /home/ubuntu/sourcecode/playbook.yml
```

---

## Step – 28: Access Web Server
```
http://<web-server-ip>
```
