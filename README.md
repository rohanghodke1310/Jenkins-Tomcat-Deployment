# Jenkins Tomcat Deployment - Freestyle Job
# 🚀 Jenkins FreeStyle CI/CD – Maven WAR Deployment to Tomcat & Amazon S3

## 📌 Project Overview

This project demonstrates a basic CI/CD workflow using a **Jenkins FreeStyle Job** to build and deploy a Java Maven web application.

The source code is stored in GitHub. Jenkins checks out the source code, builds the application using Maven, executes tests, generates a WAR artifact, deploys the WAR file to an Apache Tomcat server, and stores the WAR artifact in an Amazon S3 bucket.

The project uses separate AWS EC2 instances for Jenkins and Tomcat.

---

## 🎯 Project Objective

The main objective of this project is to understand how a Java web application can be moved from source code to a deployed application using Jenkins.

The complete workflow is:

```text
CODE
  ↓
BUILD
  ↓
TEST
  ↓
ARTIFACT
  ↓
DEPLOYMENT
  ↓
TOMCAT
  ↓
AMAZON S3
```

---

## 🏗️ Architecture

```text
                         ┌─────────────────────┐
                         │       GitHub        │
                         │   Java Maven App    │
                         └──────────┬──────────┘
                                    │
                                    │ Git Checkout
                                    ▼
                         ┌─────────────────────┐
                         │      Jenkins        │
                         │    FreeStyle Job    │
                         └──────────┬──────────┘
                                    │
                                    │ Maven
                                    │ clean package
                                    ▼
                         ┌─────────────────────┐
                         │     WAR Artifact     │
                         │        *.war         │
                         └──────────┬──────────┘
                                    │
                         ┌──────────┴──────────┐
                         │                     │
                         │ Deploy              │ Upload
                         ▼                     ▼
                ┌──────────────────┐   ┌──────────────────┐
                │  Apache Tomcat   │   │    Amazon S3     │
                │    EC2 Server    │   │ Artifact Storage │
                │      :8080       │   │                  │
                └────────┬─────────┘   └──────────────────┘
                         │
                         ▼
                    /mywebapp/
                         │
                         ▼
                  Web Application
```

---

## 🛠️ Technologies Used

| Technology | Purpose |
|---|---|
| GitHub | Source Code Management |
| Git | Source Code Checkout |
| Jenkins | CI/CD Automation |
| Jenkins FreeStyle Job | Build and Deployment |
| Java 21 (Amazon Corretto) | Application Runtime |
| Maven | Build and Package |
| Apache Tomcat 10.x | Application Server |
| AWS EC2 | Jenkins and Tomcat Servers |
| Amazon S3 | WAR Artifact Storage |
| Linux Shell Script | Server Installation |

---

## ☁️ AWS Infrastructure

Two separate EC2 instances are used.

```text
┌──────────────────────────────┐
│        Jenkins EC2           │
│                               │
│ Java 21                      │
│ Git                          │
│ Maven                        │
│ Jenkins                      │
└──────────────┬────────────────┘
               │
               │ Deploy WAR
               ▼
┌──────────────────────────────┐
│         Tomcat EC2           │
│                               │
│ Java 21                      │
│ Apache Tomcat                │
│ Port 8080                    │
└──────────────────────────────┘

               +

┌──────────────────────────────┐
│          Amazon S3           │
│                               │
│       WAR Artifacts          │
└──────────────────────────────┘
```

---

## 📂 Repository Structure

```text
jenkins-freestyle-tomcat-deployment/
│
├── README.md
│
├── scripts/
│   ├── jenkins-install.sh
│   └── tomcat.sh
│
├── jenkins/
│   └── freestyle-job-config.md
│
└── screenshots/
    ├── 01-jenkins-job.png
    ├── 02-maven-build.png
    ├── 03-aws-ec2-servers.png
    ├── 04-tomcat-manager.png
    ├── 05-deployment-success.png
    └── 06-s3-bucket.png
```

> Note: A `Jenkinsfile` is not used in this project because it uses a Jenkins **FreeStyle Job**, not a Jenkins Pipeline.

---

## 🖥️ 1. Jenkins Server Installation

Launch an Amazon Linux EC2 instance for Jenkins.

**Install Git and Maven**

```bash
sudo dnf install git maven -y
```

Verify:

```bash
git --version
mvn -version
```

**Add the Jenkins Repository**

```bash
sudo wget -O /etc/yum.repos.d/jenkins.repo \
  https://pkg.jenkins.io/redhat-stable/jenkins.repo
```

Import the Jenkins repository key:

```bash
sudo rpm --import \
  https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
```

**Install Java 21**

```bash
sudo dnf install java-21-amazon-corretto -y
java -version
```

**Install Jenkins**

```bash
sudo dnf install jenkins -y
```

**Start Jenkins**

```bash
sudo systemctl start jenkins
sudo systemctl status jenkins
```

**Enable Jenkins at Boot**

```bash
sudo systemctl enable jenkins
```

Jenkins URL:

```text
http://<JENKINS_PUBLIC_IP>:8080
```

**Get the Initial Jenkins Password**

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Copy the password and complete the Jenkins initial setup wizard.

### 📜 Jenkins Installation Script

An automated installation script is stored at `scripts/jenkins-install.sh`.

```bash
chmod +x jenkins-install.sh
sudo ./jenkins-install.sh
```

---

## 🐱 2. Tomcat Server Installation

Launch a **separate** Amazon Linux EC2 instance for Tomcat.

**Install Java**

```bash
sudo dnf install java-21-amazon-corretto -y
java -version
```

**Install wget**

```bash
sudo dnf install wget -y
```

**Download Apache Tomcat**

Example version: `10.1.33`

```bash
wget https://dlcdn.apache.org/tomcat/tomcat-10/v10.1.33/bin/apache-tomcat-10.1.33.tar.gz
```

**Extract Tomcat**

```bash
tar -xzvf apache-tomcat-10.1.33.tar.gz
cd apache-tomcat-10.1.33
```

---

## 🔐 3. Configure Tomcat Manager

Edit the users file:

```bash
vi conf/tomcat-users.xml
```

Add the required roles and a user before the closing `</tomcat-users>` tag:

```xml
<role rolename="manager-gui"/>
<role rolename="manager-script"/>

<user username="tomcat"
      password="<CHANGE_ME>"
      roles="manager-gui,manager-script"/>
```

The `manager-script` role allows Jenkins to deploy the WAR file through the Tomcat Manager interface.

> ⚠️ Never commit the real Tomcat password to GitHub. Replace `<CHANGE_ME>` with a strong password on the server only.

**Restrict Manager access (recommended)**

Tomcat Manager restricts access by IP address by default:

```bash
cd webapps/manager/META-INF
vi context.xml
```

If Jenkins needs to reach the Manager remotely, restrict access to the Jenkins server's IP rather than removing the restriction entirely.

---

## ▶️ 4. Start Tomcat

```bash
cd apache-tomcat-10.1.33/bin
./startup.sh
```

Verify:

```bash
ps -ef | grep tomcat
ss -lntp | grep 8080
```

**Tomcat URLs**

```text
http://<TOMCAT_PUBLIC_IP>:8080
http://<TOMCAT_PUBLIC_IP>:8080/manager/html
```

### 📜 Tomcat Installation Script

`scripts/tomcat.sh`:

```bash
#!/bin/bash
set -e

TOMCAT_VERSION="10.1.33"
TOMCAT_FILE="apache-tomcat-${TOMCAT_VERSION}.tar.gz"
TOMCAT_URL="https://dlcdn.apache.org/tomcat/tomcat-10/v${TOMCAT_VERSION}/bin/${TOMCAT_FILE}"

sudo dnf install java-21-amazon-corretto wget -y

wget "${TOMCAT_URL}"
tar -xzvf "${TOMCAT_FILE}"

TOMCAT_DIR="apache-tomcat-${TOMCAT_VERSION}"
echo "Tomcat downloaded and extracted to ${TOMCAT_DIR}"
echo "Configure tomcat-users.xml and manager context.xml before production use."

sh "${TOMCAT_DIR}/bin/startup.sh"
```

Run it:

```bash
chmod +x tomcat.sh
./tomcat.sh
```

---

## 🔥 5. AWS Security Groups

**Jenkins Server** — allow inbound TCP `8080` for Jenkins UI access.

**Tomcat Server** — allow inbound TCP `8080` for Jenkins/Tomcat communication.

```text
Jenkins EC2
     │
     │ TCP 8080
     ▼
Tomcat EC2
```

For production environments, restrict the source to trusted IP addresses (e.g., the Jenkins server's IP) instead of `0.0.0.0/0`.

---

## 🔌 6. Jenkins Plugins

`Manage Jenkins → Plugins`

| Plugin | Purpose |
|---|---|
| **Git** | Clone source code from GitHub |
| **Maven Integration** | Run Maven builds |
| **Deploy to Container** | Deploy WAR/EAR files to Tomcat |
| **S3 Publisher** | Publish build artifacts to Amazon S3 |
| **Pipeline: AWS Steps** | Optional — for Pipeline-based AWS integrations |

---

## 👷 7. Create the Jenkins FreeStyle Job

`Jenkins Dashboard → New Item`

- Item name: `deploytomcat`
- Type: **Freestyle project**
- Click **OK**

---

## 📦 8. Source Code Management

`Source Code Management → Git`

```text
Repository URL: https://github.com/ReyazShaik/java-project-maven-new.git
Branch:         */main
```

If the repository is private, add the appropriate GitHub credentials. If your repository uses a different default branch, select it accordingly.

---

## 🔨 9. Configure the Maven Build

`Build Steps → Invoke top-level Maven targets`

```text
Goals: clean package
```

Build process:

```text
GitHub → Checkout → Compile → Test → Package → WAR Artifact
```

The WAR file is generated inside `target/`:

```text
target/
└── myapp.war
```

---

## 🚀 10. Deploy WAR to Tomcat

`Post-build Actions → Deploy war/ear to a container`

```text
WAR/EAR files: **/*.war
Context path:  mywebapp
Container:     Tomcat
Tomcat URL:    http://<TOMCAT_PUBLIC_IP>:8080
```

Select the Tomcat version supported by your installed deployment plugin.

---

## 🔑 11. Jenkins Tomcat Credentials

`Manage Jenkins → Credentials`

```text
Username: tomcat
Password: <TOMCAT_PASSWORD>
ID:       tomcatcred
```

Select `tomcatcred` in the Tomcat deployment configuration.

> ⚠️ Never commit the actual password to GitHub.

---

## 🪣 12. Amazon S3 Artifact Storage

The generated WAR artifact is stored in Amazon S3.

```text
Bucket name: tomcat-warfile
Region:      ap-south-1
```

> S3 bucket names must be globally unique — use your own bucket name.

---

## 🔐 13. Configure AWS Credentials in Jenkins

`Manage Jenkins → Credentials → AWS Credentials`

```text
Access Key / Secret Key
Credential ID: s3creds
```

Use the Jenkins credentials store rather than placing AWS keys directly inside scripts or source code.

> For AWS-hosted Jenkins, prefer an IAM role over long-lived access keys where practical.

---

## 📤 14. Publish the WAR Artifact to S3

`Jenkins Job → Configure → Post-build Actions → Publish artifacts to S3 bucket`

```text
Source:             **/*.war
Destination bucket: tomcat-warfile/
Bucket region:      ap-south-1
Credential:         s3creds
```

Save the configuration.

---

## 🚀 15. Build the Jenkins Job

Click **Build Now**. Jenkins will:

1. Checkout source code
2. Run Maven
3. Execute tests
4. Generate the WAR file
5. Deploy the WAR file to Tomcat
6. Upload the WAR file to S3

Check `Build History → Build Number → Console Output`. A successful build shows:

```text
BUILD SUCCESS
```

---

## 🌐 16. Verify Tomcat Deployment

After a successful deployment, open:

```text
http://<TOMCAT_PUBLIC_IP>:8080/mywebapp/
```

The application should load successfully.

---

## 🪣 17. Verify the S3 Artifact

`AWS Console → S3 → Your Bucket`

```text
tomcat-warfile/
└── target/
    └── myapp.war
```

The exact WAR filename depends on the Maven project's `pom.xml`.

---

## 📸 Project Screenshots

| # | Screenshot | Description |
|---|---|---|
| 01 | Jenkins FreeStyle Job | Dashboard showing the `deploytomcat` job and a successful build |
| 02 | Maven Build | Build output showing the generated `myapp.war` artifact |
| 03 | AWS EC2 Infrastructure | Console showing separate Jenkins and Tomcat instances running |
| 04 | Tomcat Manager | Tomcat Web Application Manager accessible at `/manager/html` |
| 05 | Deployment Success | Application deployed and accessible at `/mywebapp/` |
| 06 | Amazon S3 Bucket | S3 bucket configured to store the WAR artifact |

---

## 🔒 Security Best Practices

Do **not** commit the following to GitHub:

- ❌ AWS access keys / secret keys
- ❌ Tomcat or Jenkins passwords
- ❌ Private SSH keys / `.pem` files
- ❌ Production credentials

Instead, use:

- ✅ Jenkins Credentials store
- ✅ AWS IAM Roles
- ✅ AWS Secrets Manager / SSM Parameter Store
- ✅ Environment variables

Restrict AWS Security Group access whenever possible, and avoid exposing the Tomcat Manager directly to the public internet.

---

## 🧹 Useful Jenkins Commands

```bash
# Check status
sudo systemctl status jenkins

# Start
sudo systemctl start jenkins

# Stop
sudo systemctl stop jenkins

# Restart
sudo systemctl restart jenkins

# Enable at boot
sudo systemctl enable jenkins
```

## 🐱 Useful Tomcat Commands

```bash
# Start Tomcat
./startup.sh

# Stop Tomcat
./shutdown.sh

# Check Tomcat process
ps -ef | grep tomcat

# Check port 8080
ss -lntp | grep 8080

# View logs
cd apache-tomcat-10.1.33/logs
tail -f catalina.out
```

## 🔨 Useful Maven Commands

```bash
# Build the application
mvn clean package

# Skip tests
mvn clean package -DskipTests

# Clean previous build files
mvn clean

# Check Maven version
mvn -version
```

---

## 🔍 Troubleshooting

**Jenkins build failed**

```bash
java -version
mvn -version
git --version
```

Then check `Jenkins → Build History → Build Number → Console Output`.

**WAR file not generated**

```bash
ls -lh target/
```

Expected:

```text
target/
└── myapp.war
```

Make sure the Maven project is configured correctly in `pom.xml`.

**Tomcat deployment failed**

```bash
ps -ef | grep tomcat
ss -lntp | grep 8080
tail -f apache-tomcat-10.1.33/logs/catalina.out
```

Verify: Tomcat URL, Tomcat username/password, Jenkins credential, `manager-script` role, AWS security group, and network connectivity.

**Application not accessible**

```bash
ss -lntp | grep 8080
```

Check `http://<TOMCAT_PUBLIC_IP>:8080/mywebapp/` and confirm the Tomcat EC2 security group allows inbound traffic on port 8080 from the required source.

**S3 upload failed**

Verify: S3 bucket name, AWS region, Jenkins AWS credential, IAM permissions, and that the S3 Publisher plugin is installed. The AWS identity used by Jenkins must have permission to upload the WAR artifact.

---

## 📚 Key Learnings

- Launching and configuring AWS EC2 instances
- Installing Java on Amazon Linux
- Installing Git and Maven
- Installing and configuring Jenkins
- Creating a Jenkins FreeStyle job and integrating it with GitHub
- Building a Maven application and executing tests
- Generating a WAR artifact
- Installing and configuring Apache Tomcat and its Manager application
- Deploying a WAR file from Jenkins to Tomcat
- Configuring Jenkins credentials securely
- Publishing build artifacts to Amazon S3
- Fundamentals of AWS security groups and artifact management

---

## 🎯 Complete Project Flow

```text
                         GitHub
                           │
                           ▼
                  Jenkins FreeStyle
                           │
                           ▼
                     Git Checkout
                           │
                           ▼
                  Maven clean package
                           │
                           ▼
                         Tests
                           │
                           ▼
                      myapp.war
                       /       \
                      /         \
                     ▼           ▼
                 Tomcat          S3
                Deployment     Storage
                     │           │
                     ▼           ▼
                /mywebapp/      *.war
                     │
                     ▼
               Web Application
```

---

## ✅ Project Checklist

- [x] AWS Jenkins EC2
- [x] AWS Tomcat EC2
- [x] Java 21
- [x] Git
- [x] Maven
- [x] Jenkins installed and configured
- [x] Jenkins FreeStyle Job
- [x] GitHub integration
- [x] Maven build
- [x] Test execution
- [x] WAR artifact generated
- [x] Tomcat Manager configured
- [x] WAR deployed
- [x] Application running
- [x] Amazon S3 bucket created
- [x] S3 artifact storage working

---

## 👨‍💻 Author

**Rohan Ghodke**
DevOps Learning 

**Technologies:** AWS · Jenkins · Git · GitHub · Maven · Java · Apache Tomcat · Amazon S3 · Linux

---

## ⭐ Conclusion

This project demonstrates a practical CI/CD workflow using a Jenkins FreeStyle Job. The Java web application is taken from source code in GitHub, built with Maven, packaged as a WAR file, deployed to Apache Tomcat, and stored as an artifact in Amazon S3.

```text
CODE → BUILD → TEST → ARTIFACT → DEPLOYMENT → TOMCAT → S3 ARTIFACT STORAGE
```

This project provides a strong foundation for understanding Jenkins automation, Java application deployment, Apache Tomcat, AWS EC2, and artifact management.
