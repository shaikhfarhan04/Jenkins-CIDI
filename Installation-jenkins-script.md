Yes. Since you're using **Amazon Linux 2023**, I recommend using `dnf` and Java 21. The script below automates the Jenkins EC2 setup, including **Git, Docker, Java 21, Maven, Jenkins, Docker permissions, and Jenkins startup**.

Save it as `install-jenkins.sh`.

```bash
#!/bin/bash

set -e

echo "=========================================="
echo " Jenkins EC2 Installation Script"
echo " Amazon Linux 2023"
echo "=========================================="

# --------------------------------------------------
# 1. Update system
# --------------------------------------------------

echo "[1/9] Updating system packages..."

sudo dnf update -y
sudo dnf upgrade -y

# --------------------------------------------------
# 2. Install Git and Docker
# --------------------------------------------------

echo "[2/9] Installing Git and Docker..."

sudo dnf install -y git docker wget

echo "Git version:"
git --version

echo "Docker version:"
docker --version

# --------------------------------------------------
# 3. Configure Docker
# --------------------------------------------------

echo "[3/9] Configuring Docker..."

sudo systemctl start docker
sudo systemctl enable docker

sudo systemctl status docker --no-pager

# Add current user to docker group
sudo usermod -aG docker "$USER"

echo "Docker configured."

# --------------------------------------------------
# 4. Configure Git
# --------------------------------------------------

echo "[4/9] Configuring Git..."

read -p "Enter Git username: " GIT_USERNAME
read -p "Enter Git email: " GIT_EMAIL

git config --global user.name "$GIT_USERNAME"
git config --global user.email "$GIT_EMAIL"

echo ""
echo "Git configuration:"
git config --list

# --------------------------------------------------
# 5. Install Java 21
# --------------------------------------------------

echo "[5/9] Installing Java 21..."

sudo dnf install -y java-21-amazon-corretto-devel

echo "Java version:"
java --version

# --------------------------------------------------
# 6. Install Maven
# --------------------------------------------------

echo "[6/9] Installing Maven..."

sudo dnf install -y maven

echo "Maven version:"
mvn -v

# --------------------------------------------------
# 7. Install Jenkins repository
# --------------------------------------------------

echo "[7/9] Configuring Jenkins repository..."

sudo wget -O /etc/yum.repos.d/jenkins.repo \
https://pkg.jenkins.io/rpm-stable/jenkins.repo

sudo dnf clean all

sudo dnf makecache

# --------------------------------------------------
# 8. Install Jenkins
# --------------------------------------------------

echo "[8/9] Installing Jenkins..."

sudo dnf install -y jenkins

sudo systemctl daemon-reload

sudo systemctl start jenkins
sudo systemctl enable jenkins

# Give Jenkins access to Docker
sudo usermod -aG docker jenkins

# Restart Jenkins so configuration takes effect
sudo systemctl restart jenkins

echo "Jenkins version:"
jenkins --version

echo "Jenkins service status:"
sudo systemctl status jenkins --no-pager

# --------------------------------------------------
# 9. Display Jenkins initial password
# --------------------------------------------------

echo ""
echo "=========================================="
echo " Jenkins Installation Completed"
echo "=========================================="

echo ""
echo "Java:"
java --version

echo ""
echo "Git:"
git --version

echo ""
echo "Docker:"
docker --version

echo ""
echo "Maven:"
mvn -v

echo ""
echo "Jenkins:"
jenkins --version

echo ""
echo "=========================================="
echo " Jenkins Initial Admin Password"
echo "=========================================="

if [ -f /var/lib/jenkins/secrets/initialAdminPassword ]; then
    sudo cat /var/lib/jenkins/secrets/initialAdminPassword
else
    echo "Initial password file not found."
    echo "Check Jenkins status:"
    echo "sudo systemctl status jenkins"
fi

echo ""
echo "=========================================="
echo " Jenkins URL"
echo "=========================================="

PUBLIC_IP=$(curl -s http://169.254.169.254/latest/meta-data/public-ipv4 || true)

if [ -n "$PUBLIC_IP" ]; then
    echo "http://$PUBLIC_IP:8080"
else
    echo "http://<EC2-PUBLIC-IP>:8080"
fi

echo ""
echo "=========================================="
echo " IMPORTANT"
echo "=========================================="
echo "1. Open TCP port 8080 in the EC2 Security Group."
echo "2. Open Jenkins using http://<PUBLIC-IP>:8080"
echo "3. Copy the initial admin password shown above."
echo "4. Select 'Install suggested plugins'."
echo "5. Create your Jenkins admin user."
echo ""
echo "Docker access for Jenkins:"
echo "sudo -u jenkins docker ps"
echo ""
echo "You may need to log out and log back in"
echo "for the current user's Docker group membership."
echo "=========================================="
```

### Run it

On the Jenkins EC2 instance:

```bash
nano install-jenkins.sh
```

Paste the script, save it, then:

```bash
chmod +x install-jenkins.sh
```

Run:

```bash
./install-jenkins.sh
```

Or:

```bash
bash install-jenkins.sh
```

### After installation

Verify Docker access for Jenkins:

```bash
sudo -u jenkins docker ps
```

Verify Jenkins:

```bash
sudo systemctl status jenkins
```

Get the password again if required:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Then open:

```text
http://<EC2-PUBLIC-IP>:8080
```

**One important difference from your original commands:** I would not put the Git username/email directly into the script. The script asks for them interactively, so you don't accidentally hard-code personal information into a reusable installation script.

Also, for your project, I would **not install AWS CLI, Terraform, kubectl, Helm, eksctl, or SonarQube in this base script yet**. Those can be added as a separate **Jenkins DevOps tools installation script** once the Jenkins server itself is working.
