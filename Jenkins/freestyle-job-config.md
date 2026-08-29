# Jenkins FreeStyle Job Configuration

## Project: Java Application Deployment on Tomcat

This document describes how to configure a Jenkins **FreeStyle Job** that builds a Java Maven application, packages it as a WAR file, deploys it to Tomcat, and stores the WAR artifact in Amazon S3.

---

## 1. Jenkins Server Setup

Jenkins runs on a dedicated Amazon Linux EC2 instance with the following installed:

```text
Git
Maven
Java 21
Jenkins
```

Verify the installations:

```bash
git --version
mvn -version
java -version
sudo systemctl status jenkins
```

Access Jenkins at:

```text
http://<JENKINS_PUBLIC_IP>:8080
```

---

## 2. Jenkins Plugins

`Jenkins Dashboard → Manage Jenkins → Plugins`

Install the following plugins if not already present:

| Plugin | Purpose |
|---|---|
| **Git** | Clone source code from GitHub |
| **Maven Integration** | Execute Maven build commands |
| **Deploy to Container** | Deploy the WAR file to Tomcat |
| **S3 Publisher** | Upload the WAR artifact to Amazon S3 |

---

## 3. Create the FreeStyle Job

`Jenkins Dashboard → New Item`

```text
Item name: java-project-tomcat-deployment
Type:      Freestyle project
```

Click **OK**.

---

## 4. Source Code Management

`Source Code Management → Git`

```text
Repository URL: https://github.com/ReyazShaik/java-project-maven-new.git
Branch:         */main
```

Update the branch name if your repository uses a different default branch.

---

## 5. Build Step

`Build Steps → Invoke top-level Maven targets`

```text
Maven installation: <configured Maven version>
Goals:              clean package
```

Build sequence:

```text
GitHub → Checkout Code → mvn clean package → Compile → Test → Package → WAR File
```

The WAR file is generated inside `target/`.

---

## 6. Verify the WAR Artifact

After a successful Maven build, Jenkins should produce a WAR file, for example:

```text
target/
└── java-project.war
```

The exact filename depends on the project's `pom.xml`. Use a wildcard pattern so the job doesn't need to be updated if the filename changes:

```text
**/*.war
```

---

## 7. Deploy WAR to Tomcat

`Post-build Actions → Deploy war/ear to a container`

```text
WAR/EAR files: **/*.war
Context path:  mywebapp
Container:     Tomcat
```

Select the Tomcat container type supported by your installed deployment plugin and Tomcat version.

---

## 8. Configure Tomcat Credentials

Jenkins needs permission to access the Tomcat Manager application.

`Manage Jenkins → Credentials → Add Credentials`

```text
Username: tomcat
Password: <TOMCAT_PASSWORD>
ID:       tomcatcred
```

Select `tomcatcred` in the Tomcat deployment configuration.

> ⚠️ Never commit the real Tomcat password to GitHub.

---

## 9. Tomcat URL

```text
http://<TOMCAT_PUBLIC_IP>:8080
```

Example (placeholder — replace with your own EC2 IP or hostname):

```text
http://<TOMCAT_EC2_PUBLIC_IP>:8080
```

---

## 10. Tomcat Manager Roles

Tomcat Manager must be configured with the required roles in `conf/tomcat-users.xml`:

```xml
<role rolename="manager-gui"/>
<role rolename="manager-script"/>
<user username="tomcat"
      password="<TOMCAT_PASSWORD>"
      roles="manager-gui,manager-script"/>
```

The `manager-script` role is required for automated deployments through the Tomcat Manager API.

---

## 11. Tomcat Security Group

Allow the Jenkins server to reach Tomcat on port `8080`:

```text
Jenkins Server ──(TCP 8080)──▶ Tomcat Server
```

For better security, restrict the inbound source to the Jenkins server or network rather than opening port `8080` to `0.0.0.0/0`.

---

## 12. Publish WAR to Amazon S3

`Post-build Actions → Publish artifacts to S3 bucket`

```text
Artifact:    **/*.war
Destination: <YOUR-S3-BUCKET>/
Region:      ap-south-1
Credential:  s3creds
```

Example bucket:

```text
jen-test-me-reyaz/
```

---

## 13. AWS Credentials

Configure AWS credentials in Jenkins under `Manage Jenkins → Credentials`.

```text
Credential ID: s3creds
```

The Jenkins job uses this credential to upload the WAR file to S3.

Never place credentials directly in project files, including:

```text
README.md
freestyle-job-config.md
tomcat.sh
Jenkinsfile
```

Never commit:

- AWS access keys / secret keys
- Private keys
- Passwords

---

## 14. Complete Post-Build Configuration

```text
Post-build Actions

1. Deploy war/ear to a container
   ├── WAR:          **/*.war
   ├── Context path:  mywebapp
   ├── Container:     Tomcat
   ├── Credential:    tomcatcred
   └── Tomcat URL:    http://<TOMCAT_IP>:8080

2. Publish artifacts to S3
   ├── Source:     **/*.war
   ├── Bucket:     <YOUR-S3-BUCKET>
   ├── Region:     ap-south-1
   └── Credential: s3creds
```

---

## 15. Save and Build

Click **Save**, then click **Build Now**.

---

## 16. Check the Build Console

`Build History → Build Number → Console Output`

You should see the Git checkout, Maven build, WAR creation, and deployment steps. A successful Maven build ends with:

```text
BUILD SUCCESS
```

---

## 17. Verify Tomcat Deployment

After a successful build, open:

```text
http://<TOMCAT_PUBLIC_IP>:8080/mywebapp
```

The Java web application should load.

---

## 18. Verify the S3 Artifact

`AWS Console → S3 → <Your Bucket>`

Example:

```text
jen-test-me-reyaz/
└── java-project.war
```

The WAR artifact should be present in the bucket.

---

## 19. Complete Project Flow

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
                  WAR File
                 /        \
                ▼          ▼
            Tomcat          S3
          Deployment      Artifact
               │
               ▼
           /mywebapp
               │
               ▼
         Web Application
```

---

## 20. Troubleshooting

### Jenkins Build Failed

Check the toolchain versions:

```bash
java -version
mvn -version
git --version
```

Then review `Build History → Console Output` for the failure details.

### WAR File Not Found

Confirm Maven generated the WAR:

```bash
ls -lh target/
```

Make sure the Jenkins artifact pattern is set to:

```text
**/*.war
```

### Tomcat Deployment Failed

Check whether Tomcat is running:

```bash
ps -ef | grep tomcat
ss -lntp | grep 8080
```

Review the Tomcat logs:

```bash
cd apache-tomcat-10.1.33/logs
tail -f catalina.out
```

Also verify:

- Tomcat URL
- Username / password
- Jenkins credential (`tomcatcred`)
- The `manager-script` role is assigned
- Security group / network connectivity between Jenkins and Tomcat

### Cannot Access Tomcat

`AWS EC2 → Security Groups → Inbound Rules`

Confirm port `8080` is reachable from the Jenkins server or network.

### S3 Upload Failed

Verify:

- S3 bucket name and region
- Jenkins AWS credential (`s3creds`)
- IAM permissions for the Jenkins AWS identity
- The S3 Publisher plugin is installed and configured correctly

---

## 21. Learning Outcome

After completing this project, the following concepts were reinforced:

- Jenkins FreeStyle job configuration
- GitHub integration with Jenkins
- Maven builds and WAR artifact generation
- Tomcat deployment from Jenkins
- Managing Jenkins credentials securely
- Jenkins post-build actions
- Amazon S3 artifact storage
- AWS EC2 security groups
- A basic end-to-end CI/CD workflow

---

## 22. Final CI/CD Workflow

```text
CODE → GitHub → Jenkins → BUILD (Maven) → TEST → WAR ARTIFACT
                                                     │
                                        ┌────────────┴────────────┐
                                        ▼                         ▼
                                     Tomcat                       S3
                                        │                         │
                                        ▼                         ▼
                                   /mywebapp                  Artifact
```

### Project Status

```text
✅ Jenkins Installed
✅ Git Installed
✅ Maven Installed
✅ GitHub Integrated
✅ FreeStyle Job Created
✅ Maven Build Configured
✅ WAR Generated
✅ Tomcat Configured
✅ WAR Deployed to Tomcat
✅ WAR Uploaded to S3
```
