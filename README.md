# End-to-End-CI-CD-Pipeline-for-Containerized-Java-Application
# Automated-CI-CD-for-Spring-Boot-Microservice Project with Jenkins, SonarQube, Nexus, OWASP Dependency-Check, Trivy and Docker Deployment

This document explains how I built a complete DevSecOps pipeline on AWS using  Jenkins, SonarQube, Docker, Trivy, OWASP Dependency-Check, S3 artifact storage, and final deployment to Docker-compose.

## Project Overview
- CI/CD: Jenkins Pipeline
- Code Quality: SonarQube
- Security Scanning: OWASP Dependency-Check + Trivy (Image Scanning)
- Artifact Storage: Amazon S3(backup), Nexus
- Container Registry: Docker Hub
- Application: Java Web App (Docker + WAR)
- Final Deployment: Docker-compose

## Prerequisites
- AWS Account with billing enabled
- MobaXterm or any SSH client
- Basic knowledge of AWS, Linux, Jenkins, Docker

## PORT-IP'S
- JENKINS - 8080
- SONARQUBE - 9000
- Nexus - 8081
---

### Step 1: Launch EC2 Instance (Jenkins + Tools Server)

1. Go to AWS Console → EC2 → Launch Instance
2. Name: `jenkins-server` (or any)
3. OS: **Amazon Linux 2023 AMI**
4. Instance Type: **m7i-flex.large** (or t3.large if budget constrained)
5. Key pair: Create or use existing
6. Security Group: Allow **All Traffic** (0.0.0.0/0) → only for learning/lab
7. Storage: 28 GB gp3 EBS volume
8. Launch instance
9. Connect via MobaXterm as `ec2-user`, then switch to root:
     ```bash
    sudo su -
    ```
 10. Launch another with c7i-flex.large for TOMCAT everything same 

---
 
  
### Step 2: Install Jenkins

Open new terminal → SSH into same instance as root

```bash
sudo yum update –y
sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
sudo yum upgrade
sudo yum install java-17-amazon-corretto -y
sudo yum install jenkins git -y
sudo systemctl enable jenkins
sudo systemctl start jenkins
sudo systemctl status jenkins
sudo mkdir -p /var/tmp_disk
sudo chmod 1777 /var/tmp_disk
sudo mount --bind /var/tmp_disk /tmp
echo '/var/tmp_disk /tmp none bind 0 0' | sudo tee -a /etc/fstab
sudo systemctl mask tmp.mount
df -h /tmp
sudo systemctl restart jenkins
```

Access Jenkins: `http://<public-ip>:8080`  
Get initial admin password:
```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Install **Suggested Plugins** → Create admin user (remember username/password)

---

### Step 3: Install Docker on Jenkins Server

```bash
# Docker
yum install docker -y
systemctl start docker
systemctl enable docker
chmod 777 /var/run/docker.sock   # only for lab

# SonarQube
docker run -d --name sonar -p 9000:9000 sonarqube:lts-community
```

Access SonarQube: `http://<public-ip>:9000`  
Login: `admin` / `admin` → Change password →  
My Account → Security → Generate Token → Name: `mytoken` → Copy token

---

### Step 4: Configure Jenkins Plugins

Manage Jenkins → Plugins → Available → Select below listed → Install:
- Pipeline: Stage View
- SonarQube Scanner for Jenkins
- OWASP Dependency-Check
- Docker Pipeline
- Deploy to container
- Nexus artifact uploader

---
- Restart jenkins

  
### Step 5: Configure Global Tools in Jenkins

Manage Jenkins → Tools →

1. **SonarQube Scanner** 
   - Name: `mysonar`
   - Install automatically

2. **Maven**
   - Name: `mymaven`
   - Install automatically (latest)

3. **Dependency-Check**
   - Name: `DP-Check`
   - Install automatically from GitHub

---

### Step 6: Configure SonarQube in Jenkins

Manage Jenkins → System → SonarQube servers
- Name: `mysonar`
- Server URL: `http://<public-ip>:9000`
- Server authentication token → Add → Secret Text → Paste token from SonarQube → ID: `sonar-token`

---
Manage Jenkins → System → Amazon s3 profiles → Profile name: mybucket

### Step 7: Create Jenkins Pipeline Job

New Item → Name: `mydeployment` → Pipeline → OK

#### Pipeline Stages (Add one by one)

**Stage 1: Clean Workspace**
```groovy
cleanWs()
```

**Stage 2: Checkout Code**
```groovy
git 'https://github.com/Sravanikethari/dockerwebapp.git'
```

**Stage 3: SonarQube Analysis**

Use Pipeline Syntax → Snippet Generator → `withSonarQubeEnv` → Select `mysonar` → select → add → jenkins → add credentials → select → Generate:

```groovy
          steps{
                withSonarQubeEnv('mysonar'){
                    sh "mvn clean verify sonar:sonar -Dsonar.projectKey=MyProject"
          }
```

**Stage 4: Quality Gates**

Sonarqube server → project settings → webhooks → create → name: jenkins-integration →  url: jenkinsurl/sonarqube-webhook/ →  save  
Pipeline Syntax → `waitForQualityGate` → select → sonartoken → Generate:

```groovy
  script{
  waitForQualityGate abortPipeline: false, credentialsId: 'sonartoken'
}
```

**Stage 5: Maven Build**

```groovy
sh 'mvn clean package'
sh 'cp -r target Docker-app'
```

**Stage 6: OWASP Dependency-Check**

- Get NVD API Key: https://nvd.nist.gov/developers/request-an-api-key → Apply → Get key from email
- In Jenkins → dependencyCheck → add → jenkins → Add Credentials → Secret Text → Paste API key → ID: `nvd-api-key`

Pipeline Syntax → dependencyCheck → select →  Additional arguments:

```
--scan ./ --disableYarnAudit --disableNodeAudit 
```
Then → dependencyCheckPublisher → Xml report pattern: `**/dependency-check-report.xml`

Generate pipeline →  script should look like this 

    dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', nvdCredentialsId: 'nvd', odcInstallation: 'DP-check'
      dependencyCheckPublisher pattern: '**/dependency-check-report.xml'

**Stage 7: Upload Artifact to S3**

AWS console
- Create IAM User with AmazonS3FullAccess
- Create access key → Copy Access Key & Secret
- In Jenkins → Manage Jenkins → Amazon S3 Profile → Add → Test & Save
- AWS console → s3 → create a bucket → copy bucket name remember region name

Pipeline Syntax → S3 Upload:

- File: `target/vprofile-v2.war` To check war file → cd /var/lib/jenkins/workspace/mydeployment →  ll target
- Bucket: `your-artifact-bucket-name`
- Region: match your bucket region
- opt for no upload on build failure

      s3Upload consoleLogLevel: 'INFO', dontSetBuildResultOnFailure: false, dontWaitForConcurrentBuildCompletion: false, entries: [[bucket: 'bhargavvb', excludedFile: '', flatten: false, gzipFiles: false, keepForever: false, managedArtifacts: false, noUploadOnFailure: true, selectedRegion: 'ap-northeast-1', showDirectlyInBrowser: false, sourceFile: 'target/vprofile-v2.war', storageClass: 'STANDARD', uploadFromSlave: false, useServerSideEncryption: false]], pluginFailureResultConstraint: 'FAILURE', profileName: 'mybucket', userMetadata: []

**Stage 8: Upload Artifact to Nexus**

pipeline syntax → nexusArtifactUploader artifacts → go to pom.xml file (u will find everything u need) → select Nexus3 → Nexus url: copy&paste nexus url → repositary → Nexus repo(created) → generate  

it should look like this 

    nexusArtifactUploader artifacts: [[artifactId: 'vprofile', classifier: '', file: 'target/vprofile-v2.war', type: 'war']], credentialsId: 'nexus-credentials', groupId: 'com.visualpathit', nexusUrl: '44.203.109.210:8081', nexusVersion: 'nexus3', protocol: 'http', repository: 'myproject', version: 'v2'

**Stage 9: Build Docker Images**

```groovy
sh 'docker build -t appimage Docker-app/'
sh 'docker build -t dbimage Docker-db/'
```

**Stage 10: Trivy Image Scan**

First install Trivy on Jenkins server:
```bash
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sudo sh -s -- -b /usr/local/bin v0.67.2
```

In pipeline:

```groovy
sh 'trivy image appimage'
sh 'trivy image dbimage'
```

**Stage 11: Push to Docker Hub**

Add Docker Hub credentials in Jenkins → Pipeline syntax → with docker Registry → Registry credentials → add →  jenkins → Username: Dockerhub username, Password: Dockerhub password → (ID: `dockerhub`) → select → dockerhub

```groovy
docker.withRegistry('https://index.docker.io/v1/', 'dockerhub') {
  sh 'docker tag appimage yourusername/vprofile-app:latest'
  sh 'docker tag dbimage yourusername/vprofile-db:latest'
  sh 'docker push yourusername/vprofile-app:latest'
  sh 'docker push yourusername/vprofile-db:latest'
}
```
It should look like this

      script{
                withDockerRegistry(credentialsId: 'dockerhub') {
                    sh 'docker tag appimage sravanik93/dockerproject:app'
                    sh 'docker tag dbimage sravanik93/dockerproject:db'
                    sh 'docker push sravanik93/dockerproject:app'
                    sh 'docker push sravanik93/dockerproject:db'


**Stage 12: Deploy **

1.  Install Docker-compose 
    
   ```bash
  # Download the current stable release of Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
# Apply executable permissions
sudo chmod +x /usr/local/bin/docker-compose
# Create a symbolic link
sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose

# Install using pip (Python package manager)
sudo yum install -y python3-pip
sudo pip3 install docker-compose
   ```
2. check version
   
       docker-compose --version
   
to deploy our application

    sh 'docker-compose up -d'

---

### Final Pipeline Success!

After building the pipeline:
- Application successfully deployed
- Accessible at: `http://public-ip>:1234`

---

### Final Pipeline Success Screenshots

###Instance

##Instance-1

<img width="1920" height="771" alt="instance-1" src="https://github.com/user-attachments/assets/845fee61-6bab-45b4-a450-656f55a848fc" />

##Instance-2

<img width="1907" height="764" alt="instance-2" src="https://github.com/user-attachments/assets/2ea0b9f0-23f4-4a92-aa45-ddafadeacb7c" />


##Sonarqube

<img width="1905" height="879" alt="sonarqube" src="https://github.com/user-attachments/assets/31094975-1885-4565-b685-69e6b2a7ee06" />

##s3-Bucket

<img width="1920" height="544" alt="s3-bucket" src="https://github.com/user-attachments/assets/8d9e92c8-7644-45ca-a5d1-49dde1cf697b" />

##Nexus

<img width="1920" height="877" alt="nexus" src="https://github.com/user-attachments/assets/2b9fefe2-6a61-4bb6-a242-181eacd0dcc5" />


##DockerHub

<img width="1905" height="873" alt="dockerhub" src="https://github.com/user-attachments/assets/2906094c-1bb2-4db4-8f6b-a661c4df69a1" />


##Jenkins Full-stage-Deploy

<img width="1901" height="636" alt="pipeline-full-stage" src="https://github.com/user-attachments/assets/285d6c0c-0c4d-48e1-808d-e6c8fa42f386" />


##Output-screen

<img width="1920" height="873" alt="output" src="https://github.com/user-attachments/assets/d0f0f907-fd63-44fa-9d7d-02bc5efba014" />



## Project Repository Links

- Application Code: https://github.com/bhargavv62/dockerwebapp.git

---

**Note**: This setup uses open security groups and admin privileges for learning purposes only. In production, follow security best practices (VPC, least privilege IAM, HTTPS, etc.).
