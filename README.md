# CI/CD Project 1: Web (httpd) Server Deployment

This project demonstrates a CI/CD pipeline that automates the deployment of a sample HTML web application (using httpd) on an AWS EC2 instance. It uses Git, GitHub, Jenkins, Ansible, and EC2 with passwordless SSH connections and webhooks for end-to-end automation.

---

## **Flow**

```
git → GitHub → Jenkins → Ansible → Web Host (EC2)
```

1. Developer pushes `index.html` file to GitHub from local git.
2. The code moves to Jenkins (either Jenkins pulls it, or GitHub pushes via webhook).
3. Jenkins tests, builds, and delivers/deploys the code to Ansible server.
4. Jenkins triggers the playbook on the Ansible server.
5. The playbook installs and configures the web server on the Linux web host.

---

## **Setup Steps**

### 1. Prerequisites

- A GitHub account and repository ready.
- Three EC2 instances:
  - Jenkins Server
  - Ansible Server
  - Web Host
- Proper security groups (allow SSH (22), HTTP (80)).

---

### 2. Launch EC2 Instances

Sample user-data to set hostnames at launch:

**For Jenkins**
```bash
#!/bin/bash
sudo su -
hostnamectl set-hostname Jenkins
reboot
```

**For Ansible**
```bash
#!/bin/bash
sudo su -
hostnamectl set-hostname Ansible
reboot
```

**For Web Host**
```bash
#!/bin/bash
sudo su -
hostnamectl set-hostname web
reboot
```

---

### 3. Install Required Packages

- **Jenkins Server:** Install Jenkins and git.
- **Ansible Server:** Install ansible.
- **Web Host:** No pre-reqs; will be configured by Ansible.

---

### 4. Configuration

#### 4.1 GitHub ↔ Jenkins Webhook

- On Jenkins server:  
  Go to **Account → Security → API Token → Generate**. Save the token.
- In Jenkins:  
  Go to **Configure → API Token** and apply/save.
- On GitHub:  
  - Go to **Repo Settings → Webhooks → Add webhook**  
    - **Payload URL:** Paste Jenkins webhook URL (e.g., `http://<jenkins-ip>:8080/github-webhook/`)
    - **Content type:** `application/json`
    - **Secret:** Paste Jenkins token

#### 4.2 Passwordless SSH Connections

**Between Jenkins and Ansible:**
1. On Jenkins:
   ```bash
   ssh-keygen   # (Press Enter 3 times)
   cat ~/.ssh/id_rsa.pub
   ```
2. Copy the content and paste into `~/.ssh/authorized_keys` on Ansible server.
3. On Ansible, edit `/etc/ssh/sshd_config`:
   ```
   PermitRootLogin yes
   PasswordAuthentication yes
   ```
   Then restart SSH:
   ```
   systemctl restart sshd
   ```

**Between Ansible and Web Host:**
- Repeat the above steps from Ansible to Web Host.

---

### 5. Jenkins Setup

#### 5.1 Install Publish Over SSH Plugin

1. Jenkins Web UI → **Manage Jenkins → Manage Plugins**
2. Search for and install **Publish Over SSH**.

#### 5.2 Configure SSH Servers

1. Jenkins Web UI → **Manage Jenkins → Configure System**
2. Scroll to **Publish over SSH** section.
3. Click **Add** under SSH Servers.
   - **Name:** e.g., `ansible`
   - **Hostname:** Ansible server's IP/FQDN
   - **Username:** `ec2-user` or `root`
   - **Remote Directory:** e.g., `/home/ec2-user`
   - **Authentication:** Use the appropriate private key, or password if needed.
   - **Test Configuration** to verify connection.
4. Save configuration.

#### 5.3 Create Jenkins Job/Project

1. Jenkins Dashboard → **New Item** → Name: e.g., `webserver-deploy` → **Freestyle Project** → OK.

##### a. Source Code Management

- Select **Git**.
- Enter your GitHub repository URL.
- Add credentials if your repo is private.

##### b. Build Triggers

- Check **GitHub hook trigger for GITScm polling**.

##### c. Build Steps

---

### **Build & Deployment Example**

**In the build steps, first add a Jenkins Exec command to rsync the Jenkins workspace `index.html` to the root of Ansible `/opt`:**

- Go to the **Build** section.
- Click **Add build step** → **Execute shell** (or “Send files or execute commands over SSH” if using the plugin).

**Example Build Command:**
```bash
rsync /var/lib/jenkins/workspace/<jenkins_project_name>/index.html root@<ansible_ip>:/opt
```
Replace `<jenkins_project_name>` and `<ansible_ip>` with your actual values.

---

##### d. Post-build Actions

**Send build artefacts over SSH and trigger Ansible playbook:**

- **SSH server:** Ansible  
- **Name:** Ansible  
- **Exec command:**  
  ```bash
  ansible-playbook /opt/web.yaml -i /opt/invt
  ```

---

### 6. Example Ansible Playbook (`web.yaml`)

```yaml
---
- hosts: test
  become: yes
  become_method: sudo
  become_user: root
  tasks:
    - name: Install httpd
      yum:
        name: httpd
        state: latest

    - name: Start httpd service
      service:
        name: httpd
        state: started

    - name: Copy index.html
      copy:
        src: index.html
        dest: /var/www/html/

    - name: Restart httpd service
      service:
        name: httpd
        state: restarted
```

---

### 7. Testing

- After a successful Jenkins build, access your web application at:  
  `http://<webhost-public-ip>/`

---

## **Troubleshooting: Jenkins Storage, Temp, and Swap Issues**

### Example Error
- **Critical Issues Causing Built-In Node Offline:**
    - Free Swap Space: 0 B ⚠️
    - Free Temp Space: 447.93 MiB ⚠️
    - Node Status: (offline)

**Why:**  
Jenkins monitors system resources. If swap space is zero or temp space is too low, Jenkins marks the node as offline to prevent unstable builds.

### Steps to Fix

#### 1. Add Swap Space (Amazon Linux 2023)
```bash
sudo fallocate -l 1G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```
To make it permanent:
```bash
echo '/swapfile swap swap defaults 0 0' | sudo tee -a /etc/fstab
```

#### 2. Clean Temp Directory
Check and clean `/tmp`:
```bash
sudo du -sh /tmp/*
sudo rm -rf /tmp/*
sudo systemctl restart jenkins
```
**Note:** If your free disk space is only 4.75 GiB, consider cleaning up or expanding storage soon.

#### 3. Check /tmp Space
```bash
df -h /tmp
```
Example output:
```
tmpfs   452M   4M   448M  1% /tmp
```

#### 4. Two Recommended Fixes

**Option 1:** Increase /tmp Size Temporarily (if tmpfs)
```bash
sudo mount -o remount,size=2G /tmp
```
*(Note: This change lasts until reboot.)*

**Option 2:** Use Disk for Jenkins Temp Directory
1. Create new temp directory:
    ```bash
    sudo mkdir -p /opt/jenkins_tmp
    sudo chmod 777 /opt/jenkins_tmp
    ```
2. Configure Jenkins to use this directory as `JAVA_OPTS` temp location, and restart Jenkins:
    ```bash
    sudo systemctl restart jenkins
    ```

#### After Fix
- Recheck in Jenkins: **Manage Jenkins → Nodes → Built-In Node**
- Temp space should now reflect correctly and node should come online.

---

## **License**
This project is for educational and demonstration purposes.