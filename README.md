# Jenkins-Tomcat-Deployment
# Day 6 — Jenkins FreeStyle Job: WAR Deployment on Tomcat

## Project Overview

This project demonstrates how to build and deploy a Java Maven web application using a **Jenkins FreeStyle Job**.

Jenkins pulls the source code from GitHub, builds the application with Maven, packages it as a `.war` artifact, deploys the WAR file to an Apache Tomcat server, and finally stores the WAR artifact in an Amazon S3 bucket for centralized artifact storage.

### Project Flow

```text
GitHub
  │
  ▼
Jenkins
  │
  ├── Checkout Source Code
  ├── Maven Build (clean package)
  ├── Generate WAR File
  ├── Deploy WAR to Tomcat
  └── Upload WAR to Amazon S3
```

---

## Architecture

```text
                    ┌─────────────────┐
                    │      GitHub      │
                    │  Java Maven App  │
                    └────────┬─────────┘
                             │ git clone
                             ▼
                    ┌─────────────────┐
                    │     Jenkins      │
                    │  FreeStyle Job   │
                    └────────┬─────────┘
                             │ mvn clean package
                             ▼
                    ┌─────────────────┐
                    │     WAR File      │
                    │      *.war        │
                    └────────┬─────────┘
                             │
                 ┌───────────┴───────────┐
                 │                       │
          Deploy WAR                Upload Artifact
                 │                       │
                 ▼                       ▼
        ┌────────────────┐      ┌─────────────────┐
        │  Tomcat Server  │      │  Amazon S3      │
        │     :8080       │      │    Bucket       │
        └────────┬────────┘      └─────────────────┘
                 │
                 ▼
            /mywebapp
```

---

## Technologies Used

- AWS EC2
- Amazon Linux 2023
- Jenkins
- Git / GitHub
- Apache Maven
- Java (Amazon Corretto 21)
- Apache Tomcat 10.x
- Amazon S3
- Jenkins FreeStyle Job

---

## Repository Structure

```text
jenkins-freestyle-tomcat-deployment/
│
├── README.md
│
├── scripts/
│   └── tomcat.sh
│
├── jenkins/
│   └── freestyle-job-config.md
│
├── docs/
│   ├── tomcat-setup.md
│   ├── jenkins-setup.md
│   ├── deployment.md
│   └── s3-artifact-storage.md
│
└── screenshots/
    ├── 01-jenkins-job.png
    ├── 02-maven-build.png
    ├── 03-tomcat-server.png
    ├── 04-deployment-success.png
    └── 05-s3-artifact.png
```

---

## CI/CD Workflow

```text
CODE → BUILD → TEST → ARTIFACT → DEPLOY → TOMCAT → S3
```

### 1. Code

The Java web application source code is maintained in GitHub.

Example repository:

```text
https://github.com/ReyazShaik/java-project-maven-new.git
```

### 2. Build

Jenkins builds the application using Maven:

```bash
mvn clean package
```

### 3. Artifact

Maven produces a WAR file inside the `target/` directory:

```text
target/
└── application.war
```

### 4. Deployment

Jenkins deploys the generated WAR file to the Tomcat server.

### 5. Artifact Storage

Jenkins uploads the WAR file to an Amazon S3 bucket for long-term artifact storage.

---

## AWS Infrastructure

Two EC2 instances are used for this project:

```text
Server 1 → Jenkins Server
Server 2 → Tomcat Server
```

```text
Jenkins EC2 ──(deploy WAR)──▶ Tomcat EC2 (port 8080)
```

---

## Tomcat Server Setup

Launch an **Amazon Linux 2023** EC2 instance for Tomcat.

### Step 1 — Install Java

```bash
sudo dnf install java-21-amazon-corretto -y
java -version
```

### Step 2 — Install wget

```bash
sudo dnf install wget -y
```

### Step 3 — Download Tomcat

Apache Tomcat releases are available at [https://dlcdn.apache.org/tomcat/](https://dlcdn.apache.org/tomcat/).

```bash
wget https://dlcdn.apache.org/tomcat/tomcat-10/v10.1.33/bin/apache-tomcat-10.1.33.tar.gz
```

### Step 4 — Extract Tomcat

```bash
tar -xzvf apache-tomcat-10.1.33.tar.gz
cd apache-tomcat-10.1.33
```

### Step 5 — Configure Tomcat Users

```bash
cd apache-tomcat-10.1.33/conf
vi tomcat-users.xml
```

Add the required roles and a user before the closing `</tomcat-users>` tag:

```xml
<role rolename="manager-gui"/>
<role rolename="manager-script"/>
<user username="tomcat"
      password="<CHANGE_ME>"
      roles="manager-gui,manager-script"/>
```

> ⚠️ Never commit real credentials to GitHub. Replace `<CHANGE_ME>` with a strong password on the server only.

### Step 6 — Configure Tomcat Manager Access

Tomcat Manager restricts access by IP address by default.

```bash
cd apache-tomcat-10.1.33/webapps/manager/META-INF
vi context.xml
```

If Jenkins needs to reach the Manager application remotely, update the access restriction to allow only the Jenkins server's IP.

> ⚠️ Avoid removing IP restrictions entirely on an internet-facing server. Restrict access to the Jenkins server's private/public IP or a secured network path.

### Step 7 — Start Tomcat

```bash
cd apache-tomcat-10.1.33/bin
./startup.sh
```

Verify Tomcat is running:

```bash
ps -ef | grep tomcat
```

---

## AWS Security Group

Tomcat listens on port `8080`. Allow inbound TCP `8080` in the Tomcat EC2 security group.

For better security, restrict the source to the Jenkins server's IP rather than `0.0.0.0/0`.

---

## Accessing Tomcat

```text
http://<TOMCAT_PUBLIC_IP>:8080
http://<TOMCAT_PUBLIC_IP>:8080/manager/html
```

Use the credentials configured in `conf/tomcat-users.xml`.

---

## Automated Tomcat Installation Script

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

```bash
chmod +x tomcat.sh
./tomcat.sh

## Jenkins Server Setup

Launch a separate Amazon Linux EC2 instance for Jenkins and install Jenkins following the [official Jenkins installation guide](https://www.jenkins.io/doc/book/installing/).
Access Jenkins at:
```text
http://<JENKINS_PUBLIC_IP>:8080
```

---

## Jenkins Plugins

`Dashboard → Manage Jenkins → Plugins`

| Plugin | Purpose |
|---|---|
| **Git** | Clone source code from GitHub |
| **Maven Integration** | Run Maven builds |
| **Deploy to Container** | Deploy WAR/EAR files to Tomcat |
| **S3 Publisher** | Publish build artifacts to Amazon S3 |
| **Pipeline: AWS Steps** | Optional, for Pipeline-based AWS integrations |

---

## Creating the Jenkins FreeStyle Job

`New Item → Enter item name → Freestyle project → OK`

Example job name: `java-project-tomcat-deployment`

### 1. Source Code Management

`Source Code Management → Git`

```text
Repository URL: https://github.com/ReyazShaik/java-project-maven-new.git
```

If the repository is private, add the appropriate GitHub credentials.

### 2. Build Step

`Build Steps → Invoke top-level Maven targets`

```text
Goals: clean package
```

Maven generates the WAR file under `target/`.

### 3. Deploy WAR to Tomcat

`Post-build Actions → Deploy war/ear to a container`

```text
WAR/EAR files: **/*.war
Context path:  mywebapp
Container:     Tomcat
```

Select the Tomcat version supported by your deployment plugin.

### Jenkins Tomcat Credentials

Create Jenkins credentials for the Tomcat Manager user:

```text
Username: tomcat
Password: <TOMCAT_PASSWORD>
ID:       tomcatcred
```

```text
Credentials:  tomcatcred
Tomcat URL:   http://<TOMCAT_PUBLIC_IP>:8080
```

> ⚠️ Never commit `tomcatcred`, passwords, or any credentials to GitHub.

---

## Running the Job

Click **Build Now**. Jenkins will:

1. Clone the GitHub repository
2. Run the Maven build
3. Generate the WAR file
4. Deploy the WAR file to Tomcat

Check **Console Output** for build and deployment logs.

---

## Testing the Application

After a successful deployment, open:

```text
http://<TOMCAT_PUBLIC_IP>:8080/mywebapp
```

## Storing the WAR Artifact in Amazon S3

```text
Jenkins → target/*.war → Amazon S3 (jen-test-me-reyaz/)
```

### Step 1 — Create an S3 Bucket

```text
Bucket name: jen-test-me-reyaz
Region:      ap-south-1
> Bucket names must be globally unique — use your own unique name.

 ### Step 2 — Install the S3 Publisher Plugin
Manage Jenkins → Plugins → search "S3 Publisher" → Install`, then restart Jenkins if prompted.

### Step 3 — Configure AWS Credentials
Store AWS credentials in the Jenkins credentials store — never in the GitHub repository.

```text
Jenkins → Credentials → AWS Credentials → Access Key / Secret Key
Credential ID: s3creds

For Jenkins hosted on AWS, prefer an IAM role over long-lived access keys where possible.

### Step 4 — Configure the S3 Publisher Post-Build Step

`Jenkins Job → Configure → Post-build Actions → Publish artifacts to S3 bucket`

```text
Source:            **/*.war
Destination bucket: jen-test-me-reyaz/
Bucket region:      ap-south-1
Credential:         s3creds
```

Save the configuration.
---

## Final Pipeline Flow

Although this is a FreeStyle job, the logical workflow is:

```text
GitHub
  │
  ▼
Jenkins (FreeStyle)
  │
  ▼
Git Checkout
  │
  ▼
Maven Build (mvn clean package)
  │
  ▼
WAR File
  │
  ├──────────────┬──────────────┐
  ▼                              ▼
Tomcat Deployment           S3 Artifact
  │                              │
  ▼                              ▼
/mywebapp                      *.war
---

## Verification Checklist

**GitHub**
- [ ] Repository is accessible
- [ ] Jenkins can clone the repository

**Maven**
- [ ] Build succeeds
- [ ] Tests execute successfully
- [ ] WAR file is generated

**Tomcat**
- [ ] Tomcat is running
- [ ] Port 8080 is reachable from Jenkins
- [ ] Manager credentials are correct
- [ ] WAR deployment succeeds

**Application**
- [ ] `/mywebapp` is accessible
- [ ] Application loads successfully

**S3**
- [ ] Bucket exists
- [ ] Jenkins has upload permission
- [ ] WAR file appears in the bucket

---

## Security Best Practices

Do **not** commit the following to GitHub:

- AWS access keys / secret keys
- Tomcat or Jenkins passwords
- Private SSH keys / `.pem` files
- Production credentials

Instead, use:

- Jenkins Credentials store
- AWS IAM Roles
- Environment variables
- AWS Secrets Manager / SSM Parameter Store

Avoid exposing the Tomcat Manager to the public internet. Prefer routing Jenkins → Tomcat traffic over a private network and restricting security group rules accordingly.

---

## Useful Tomcat Commands

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

## Useful Maven Commands

```bash
# Build the application
mvn clean package

# Skip tests
mvn clean package -DskipTests

# Clean previous build files
mvn clean
```

## What I Learned

- Launching and configuring EC2 instances on AWS
- Installing Java on Amazon Linux
- Installing and configuring Apache Tomcat manually and via script
- Configuring the Tomcat Manager application
- Creating a Jenkins FreeStyle job and integrating it with GitHub
- Building a Maven application and generating a WAR artifact
- Deploying a WAR file to Tomcat from Jenkins
- Configuring Jenkins credentials securely
- Publishing build artifacts to Amazon S3
- Fundamentals of AWS security groups and artifact management

---
## Project Objective

To understand a basic CI/CD deployment workflow, end to end:

```text
Developer → GitHub → Jenkins → Maven Build → WAR Artifact → Tomcat → Web App
                                                     │
                                                     └──▶ Amazon S3

**Stack:** 
AWS · Jenkins · Git · GitHub · Maven · Java · Tomcat · S3

---

## Conclusion

This project demonstrates how a Java web application can be taken from source code to a deployed application using a Jenkins FreeStyle Job:
```text
CODE → BUILD → TEST → ARTIFACT → DEPLOY → TOMCAT → STORE IN S3
It provides a foundational, practical understanding of a Jenkins-based CI/CD deployment workflow.
