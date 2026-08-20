Your steps are mostly correct for setting up **Jenkins on Amazon Linux 2023**, but there are a few important corrections—especially around **Java 21, Docker permissions, Jenkins repository setup, and security group configuration**.

## Jenkins EC2 Configuration

### 1. EC2 instance

Recommended configuration:

| Setting        | Recommendation                           |
| -------------- | ---------------------------------------- |
| AMI            | Amazon Linux 2023                        |
| Instance       | `t3.medium` minimum                      |
| RAM            | 4 GB on t3.medium / 8 GB on t3.large     |
| Storage        | 80 GB gp3 SSD                            |
| Security Group | SSH 22 + Jenkins 8080                    |
| Key pair       | `jenkins.pem`                            |
| Public IP      | Required unless using VPN/private access |

For a learning/DevOps lab, **t3.medium is sufficient**. If you plan to run Docker builds, Maven builds, SonarQube, multiple agents, etc., use **t3.large**.

### 2. Connect from Windows

```powershell
cd Downloads

chmod 400 jenkins.pem

ssh -i "jenkins.pem" ec2-user@<EC2-PUBLIC-DNS>
```

For example:

```powershell
ssh -i "jenkins.pem" ec2-user@ec2-107-20-105-95.compute-1.amazonaws.com
```

### 3. Update Amazon Linux

```bash
sudo dnf update -y
sudo dnf upgrade -y
```

> On Amazon Linux 2023, `dnf` is preferred. `yum` may still work as a compatibility command.

### 4. Install Git and Docker

```bash
sudo dnf install git docker -y

git --version
docker --version
```

Configure Git:

```bash
git config --global user.name "Atul Kamble"
git config --global user.email "atul_kamble@hotmail.com"

git config --list
```

Start Docker:

```bash
sudo systemctl start docker
sudo systemctl enable docker
sudo systemctl status docker
```

Add your EC2 user to Docker group:

```bash
sudo usermod -aG docker $USER
newgrp docker
```

Test:

```bash
docker ps
```

### 5. Install Java 21

For current Jenkins installations, Java 21 is a good choice.

Search:

```bash
sudo dnf search java-21
```

Install:

```bash
sudo dnf install java-21-amazon-corretto-devel -y
```

Verify:

```bash
java --version
```

You should see Java 21.

### 6. Install Maven

```bash
sudo dnf install maven -y

mvn -v
```

You should see Maven using Java 21.

### 7. Install Jenkins

Download the Jenkins repository file:

```bash
sudo wget -O /etc/yum.repos.d/jenkins.repo \
  https://pkg.jenkins.io/rpm-stable/jenkins.repo
```

Then:

```bash
sudo dnf upgrade -y
sudo dnf install jenkins -y
```

Check:

```bash
jenkins --version
```

### 8. Start Jenkins

```bash
sudo systemctl daemon-reload

sudo systemctl start jenkins
sudo systemctl enable jenkins

sudo systemctl status jenkins
```

If Jenkins is running, you should see:

```text
Active: active (running)
```

### 9. Give Jenkins access to Docker

This is **very important** if Jenkins will build Docker images.

```bash
sudo usermod -aG docker jenkins
```

Restart Jenkins:

```bash
sudo systemctl restart jenkins
```

Check:

```bash
id jenkins
```

You should see `docker` in the groups.

You can also verify:

```bash
sudo -u jenkins docker ps
```

If that works, Jenkins can execute Docker commands.

### 10. Open Jenkins port in AWS Security Group

Your EC2 Security Group should have:

| Type       | Port | Source   |
| ---------- | ---: | -------- |
| SSH        |   22 | Your IP  |
| Custom TCP | 8080 | Your IP  |
| HTTP       |   80 | Optional |
| HTTPS      |  443 | Optional |

For a lab you can temporarily use:

```text
8080 → 0.0.0.0/0
```

but **do not leave Jenkins 8080 open to the entire internet for a production installation**.

### 11. Open Jenkins

From your Windows browser:

```text
http://<EC2-PUBLIC-IP>:8080
```

Example:

```text
http://107.20.105.95:8080
```

### 12. Get Jenkins initial password

On EC2:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Copy the generated password and paste it into the Jenkins setup screen.

**Don't reuse or publish the password shown in your notes.** Since it was exposed in the conversation, if that was a real active Jenkins credential, regenerate/reset it.

### 13. Jenkins initial setup

Select:

**Install suggested plugins**

Then create the administrator account.

For example:

```text
Username: admin
Password: <strong unique password>
Email: <your email>
```

---

## 14. Verify the complete Jenkins environment

After Jenkins is installed, run:

```bash
java --version
git --version
docker --version
mvn -v
jenkins --version
```

Then:

```bash
sudo systemctl status docker
sudo systemctl status jenkins
```

And:

```bash
sudo -u jenkins docker ps
```

You want all of these to work.

### Recommended final architecture

```text
                    Internet
                       |
                       |
                AWS Security Group
                  |           |
                SSH 22      Jenkins 8080
                  |           |
                  v           v
             +----------------------+
             |    Jenkins EC2       |
             |  Amazon Linux 2023   |
             |                      |
             |  Jenkins             |
             |  Java 21             |
             |  Git                 |
             |  Maven               |
             |  Docker              |
             |                      |
             |  80 GB gp3           |
             +----------------------+
                       |
                       v
                 Docker Builds
                       |
                       v
                     ECR
                       |
                       v
                    EKS
```

For your **Hospital Management DevOps project**, this is a good Jenkins foundation. The natural next step would be to configure **Jenkins → GitHub → Maven/Docker → ECR → EKS**, with a Jenkins Pipeline (`Jenkinsfile`) for the application.
