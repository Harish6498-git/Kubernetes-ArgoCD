🚀 SonarQube Installation on Ubuntu (with PostgreSQL)

⚠️ Requirements:

Ubuntu 20.04 or 22.04 LTS

At least 4 vCPUs + 8 GB RAM (SonarQube is heavy)

Java 17 (required)

PostgreSQL ≥ 12 (only supported DB)

1. Install Java 17 (Temurin / OpenJDK)
sudo apt-get update -y
sudo apt-get install -y wget unzip apt-transport-https gnupg

# Add Adoptium repo for Temurin JDK 17
wget -O- https://packages.adoptium.net/artifactory/api/gpg/key/public | sudo gpg --dearmor -o /usr/share/keyrings/adoptium.gpg
echo "deb [signed-by=/usr/share/keyrings/adoptium.gpg] https://packages.adoptium.net/artifactory/deb $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/adoptium.list
sudo apt-get update -y
sudo apt-get install -y temurin-17-jdk

# verify
java -version

2. Install & Configure PostgreSQL
sudo apt-get install -y postgresql postgresql-contrib

# login as postgres
sudo -u postgres psql


Inside psql:

CREATE USER sonar WITH ENCRYPTED PASSWORD 'sonarpass';
CREATE DATABASE sonarqube OWNER sonar;
GRANT ALL PRIVILEGES ON DATABASE sonarqube TO sonar;
\q

3. Download & Configure SonarQube
cd /opt
sudo wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-10.6.0.92116.zip
sudo apt-get install -y unzip
sudo unzip sonarqube-10.6.0.92116.zip
sudo mv sonarqube-10.6.0.92116 sonarqube
sudo useradd --system --create-home --shell /bin/bash sonarqube
sudo chown -R sonarqube:sonarqube /opt/sonarqube

4. Configure DB Connection

Edit config:

sudo nano /opt/sonarqube/conf/sonar.properties


Uncomment & set:

sonar.jdbc.username=sonar
sonar.jdbc.password=sonarpass
sonar.jdbc.url=jdbc:postgresql://localhost:5432/sonarqube


Also set:

sonar.web.host=0.0.0.0

5. Systemd Service for SonarQube
sudo nano /etc/systemd/system/sonarqube.service


Paste:

[Unit]
Description=SonarQube service
After=network.target postgresql.service

[Service]
Type=forking
User=sonarqube
Group=sonarqube
ExecStart=/opt/sonarqube/bin/linux-x86-64/sonar.sh start
ExecStop=/opt/sonarqube/bin/linux-x86-64/sonar.sh stop
LimitNOFILE=65536
LimitNPROC=4096
Restart=always

[Install]
WantedBy=multi-user.target

6. Start SonarQube
sudo systemctl daemon-reload
sudo systemctl enable sonarqube
sudo systemctl start sonarqube
sudo systemctl status sonarqube


Logs:

tail -f /opt/sonarqube/logs/sonar.log

7. Access SonarQube

Open: http://<your-server-ip>:9000

Default login: admin / admin → change password

8. Integrate with Jenkins

In SonarQube UI → My Account → Security → Generate Token (copy it).

Jenkins → Manage Jenkins → Plugins → install SonarQube Scanner for Jenkins.

Jenkins → Manage Jenkins → System → SonarQube servers:

Name: MySonarQube

URL: http://<server-ip>:9000

Credentials: add token as Secret text.

SonarQube UI → Administration → Webhooks → add:

URL: http://<jenkins-ip>:8080/sonarqube-webhook/

✅ Done — you now have SonarQube running natively on Ubuntu with PostgreSQL, managed as a system service, and ready to integrate into Jenkins pipelines.
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------
🚀 Jenkins Installation on Ubuntu
1. Update system
sudo apt-get update -y
sudo apt-get upgrade -y

2. Install Java (Jenkins needs Java 17+ now)
sudo apt-get install -y fontconfig openjdk-17-jdk
java -version


(You should see something like openjdk version "17.x")

3. Add Jenkins Repository
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null

echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null

4. Install Jenkins
sudo apt-get update -y
sudo apt-get install -y jenkins

5. Start and Enable Jenkins Service
sudo systemctl start jenkins
sudo systemctl enable jenkins
sudo systemctl status jenkins


✅ Service should be active (running).

6. Open Firewall (if enabled)
sudo ufw allow 8080
sudo ufw status

7. Setup Jenkins in Browser

Open: http://<your-server-ip>:8080

First-time setup asks for an Admin Password:

sudo cat /var/lib/jenkins/secrets/initialAdminPassword


Copy that password → paste in browser.

Choose Install Suggested Plugins (recommended).

Create first admin user.

8. Verify

Jenkins is now running on port 8080.

Default home dir: /var/lib/jenkins

Configs/logs: /var/log/jenkins/                                                                                                                                     
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
🔹 Steps to Add SonarQube in Jenkins

Go to Jenkins Dashboard → Manage Jenkins → System.
Scroll to SonarQube servers.

Check: “Environment variables” (this allows you to use $SONAR_HOST_URL inside pipelines).

Click Add SonarQube → fill fields:

Name:
e.g. MySonarQube
(this must match the string you use in Jenkinsfile → withSonarQubeEnv('MySonarQube'))

Server URL:
If installed locally on Ubuntu server:
http://localhost:9000
Or, if accessed from another server:
http://<your-sonarqube-server-ip>:9000

Server authentication token:

Go to SonarQube Web UI → My Account → Security → Generate Tokens.

Copy the token.

In Jenkins, click Add → Jenkins → Secret text.

Paste the token. Give it an ID (e.g. sonar-token).

Select that credential here.

Click Save (bottom of page).

In SonarQube UI → Administration → Webhooks → Add

Name: Jenkins

URL: http://<jenkins-server>:8080/sonarqube-webhook/

This ensures Jenkins knows when analysis is done.
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Install Maven via apt (good for most cases)
# 1) Update & install Java 17 (required for modern Maven)
sudo apt-get update -y
sudo apt-get install -y openjdk-17-jdk

# 2) Install Maven from Ubuntu repos
sudo apt-get install -y maven

# 3) Verify
mvn -version
java -version
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
You’ll create two permission statements (click Add more permissions to add the second).

Statement 1 — allow ListBucket (bucket-level)

Service: S3

Actions: ListBucket

Resources → bucket: Specific → Add ARN → Bucket name: jenkins-s3-artifacts

Request conditions → Add condition:

Key: s3:prefix

Operator: StringLike

Value: datastore/*
(Adjust datastore/ to your chosen folder. If you’re not using a folder, use *.)

Statement 2 — allow puts (object-level)

Service: S3

Actions:

PutObject

AbortMultipartUpload

ListMultipartUploadParts
(add PutObjectAcl only if your bucket requires bucket-owner-full-control)

Resources → object: Specific → Add ARN

Bucket: jenkins-s3-artifacts

Object: datastore/*.jar (or *.jar at bucket root if no folder)

Request conditions (optional but good):

Key: aws:SecureTransport → Bool = true (forces HTTPS)

Do not add anything under accesspoint or accesspointobject. Leave those empty.

Equivalent JSON (ready to paste)

(Folder-scoped to datastore/ and only *.jar files.)

{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ListBucketForPrefixOnly",
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::jenkins-s3-artifacts",
      "Condition": { "StringLike": { "s3:prefix": ["datastore/*"] } }
    },
    {
      "Sid": "PutJarsAndMultipartInPrefix",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:AbortMultipartUpload",
        "s3:ListMultipartUploadParts"
      ],
      "Resource": "arn:aws:s3:::jenkins-s3-artifacts/datastore/*.jar",
      "Condition": { "Bool": { "aws:SecureTransport": "true" } }
    }
  ]
}

If you use SSE-KMS on the bucket

Add a 3rd statement (replace KMS key ARN), and pass KMS flags in your Jenkins step:

{
  "Sid": "AllowKmsForS3Put",
  "Effect": "Allow",
  "Action": ["kms:Encrypt", "kms:GenerateDataKey*"],
  "Resource": "arn:aws:kms:us-east-1:123456789012:key/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}


Jenkins CLI example:

aws s3 cp ./target/*.jar s3://jenkins-s3-artifacts/datastore/ \
  --sse aws:kms --sse-kms-key-id arn:aws:kms:us-east-1:123456789012:key/...

Final Jenkins step (match the prefix you allowed)
aws s3 cp ./target/*.jar s3://jenkins-s3-artifacts/datastore/


That’s it—least privilege to list only that folder and put only JARs under it. If your actual bucket/prefix differs, tell me the names and I’ll tailor the ARNs precisely.
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Here’s the clean, official way to install AWS CLI v2 on Ubuntu (works on 20.04/22.04/24.04). This installs to /usr/local/aws-cli and puts aws in your PATH at /usr/local/bin/aws.

Install (x86_64 / Intel-AMD)
sudo apt-get update -y
sudo apt-get install -y curl unzip

cd /tmp
curl -sSLo awscliv2.zip https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip
unzip -q awscliv2.zip
sudo ./aws/install

aws --version
# => aws-cli/2.x.x ...

Install (ARM / Graviton / Raspberry Pi)
sudo apt-get update -y
sudo apt-get install -y curl unzip

cd /tmp
curl -sSLo awscliv2.zip https://awscli.amazonaws.com/awscli-exe-linux-aarch64.zip
unzip -q awscliv2.zip
sudo ./aws/install

aws --version

Upgrade later
cd /tmp
curl -sSLo awscliv2.zip https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip   # or aarch64
unzip -q -o awscliv2.zip
sudo ./aws/install --update
aws --version

Uninstall (if needed)
sudo /usr/local/aws-cli/v2/current/dist/aws --version  # optional check
sudo rm -rf /usr/local/aws-cli
sudo rm -f /usr/local/bin/aws

Make sure Jenkins can use it

If Jenkins runs on the same box:

sudo -u jenkins aws --version            # should print the version
# If not found, restart Jenkins so PATH picks up /usr/local/bin
sudo systemctl restart jenkins

Configure credentials (pick one)

EC2 instance role (best): nothing to configure; the CLI will use the attached role.

Access keys (if you must):

aws configure
# AWS Access Key ID [None]: AKIA...
# AWS Secret Access Key [None]: ********
# Default region name [None]: us-east-1
# Default output format [None]: json


Tip: For your Jenkins pipeline that uploads to S3, set the region once:

aws configure set default.region us-east-1


(or export AWS_DEFAULT_REGION in the pipeline env).

That’s it—AWS CLI v2 is installed and ready for your Jenkins S3 step.
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Here’s the clean, official way to install Docker Engine on Ubuntu (20.04 / 22.04 / 24.04). Includes Compose v2 and Jenkins tips.

1) Remove old versions (if any)
sudo apt-get remove -y docker docker-engine docker.io containerd runc || true

2) Prereqs
sudo apt-get update -y
sudo apt-get install -y ca-certificates curl gnupg lsb-release

3) Add Docker’s GPG key & repo
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release; echo $VERSION_CODENAME) stable" \
| sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

4) Install Docker Engine + Compose
sudo apt-get update -y
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

5) Enable and verify
sudo systemctl enable --now docker
docker --version
docker compose version

6) Run without sudo (your user)
sudo usermod -aG docker $USER
# Re-login to your shell (or run: newgrp docker)
docker run --rm hello-world

Jenkins on the same host? (important)

Add the jenkins service user to the docker group and restart Jenkins:

sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
# Optional: verify Docker works for jenkins
sudo -u jenkins -H docker version

Upgrade later
sudo apt-get update -y
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

Uninstall (if needed)
sudo apt-get purge -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo rm -rf /var/lib/docker /var/lib/containerd
sudo rm -f /etc/apt/sources.list.d/docker.list /etc/apt/keyrings/docker.gpg


That’s it—Docker Engine + Compose v2 are ready to go. If you want, I can also share a minimal hardening snippet (daemon.json) or set up rootless mode.
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Here’s the clean, official way to install Trivy on Ubuntu (20.04/22.04/24.04) and make sure Jenkins can use it.

Install via apt (recommended)
# 1) Prereqs
sudo apt-get update -y
sudo apt-get install -y curl gnupg lsb-release

# 2) Add Aqua Security’s Trivy repo (no deprecated apt-key)
sudo mkdir -p /usr/share/keyrings
curl -fsSL https://aquasecurity.github.io/trivy-repo/deb/public.key \
  | sudo gpg --dearmor -o /usr/share/keyrings/trivy.gpg

echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] \
https://aquasecurity.github.io/trivy-repo/deb \
stable main" | sudo tee /etc/apt/sources.list.d/trivy.list > /dev/null

# 3) Install
sudo apt-get update -y
sudo apt-get install -y trivy

# 4) Verify
trivy --version


If your machine is ARM (Graviton/RPi), the repo handles it automatically—no change needed.

First-time DB download (faster scans)
# Pull vulnerability DB once (global flag)
sudo trivy --download-db-only


(Trivy will refresh the DB automatically; you can also set a cron if you want.)

Make sure Jenkins can run Trivy

If Jenkins is on the same host:

sudo -u jenkins trivy --version || sudo systemctl restart jenkins


If you scan Docker images, ensure Jenkins can access Docker:

sudo usermod -aG docker jenkins
sudo systemctl restart jenkins

Quick usage examples

Scan a Docker image (what you use in your pipeline):

trivy image datastore:"${App_Version}"


Fail the build on HIGH/CRITICAL vulns:

trivy image --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 datastore:"${App_Version}"


Scan local filesystem (app deps, binaries):

trivy fs .


Speed up CI by pre-warming cache (add before the scan stage):

trivy --download-db-only || true


That’s it—Trivy is installed and ready for your Jenkins pipeline. If you want, I can drop a hardened Trivy step into your Jenkinsfile (e.g., fail on HIGH/CRITICAL with a neat summary).
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
