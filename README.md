# CI/CD DevOps Pipeline — `cicd-app`

A complete CI/CD pipeline for a Spring Boot application using **GitHub, Jenkins, Maven, SonarQube, Trivy, Docker, Docker Hub, SSH, and Oracle Cloud VM**.

## 🚀 Project Overview

This project automates the complete software delivery process:

```text
Developer
   |
   | git push
   v
GitHub
   |
   | Webhook
   v
Jenkins
   |
   +--> Checkout
   +--> Build & Test
   +--> SonarQube Analysis
   +--> Quality Gate
   +--> Package
   +--> Docker Build
   +--> Trivy Security Scan
   +--> Docker Hub Push
   |
   | SSH
   v
Oracle Cloud VM
   |
   +--> Docker Pull
   +--> Stop old container
   +--> Run new container
   +--> Health Check
   +--> Rollback if unhealthy
   |
   v
Spring Boot Application :8081
```

---

## 🛠️ Technologies Used

| Technology | Purpose |
|---|---|
| **Java / Spring Boot** | Application |
| **Maven** | Build and test |
| **Git / GitHub** | Source control |
| **Jenkins** | CI/CD automation |
| **SonarQube** | Code quality analysis |
| **Trivy** | Container vulnerability scanning |
| **Docker** | Containerization |
| **Docker Hub** | Container image registry |
| **SSH** | Secure remote deployment |
| **Oracle Cloud VM** | Remote deployment server |
| **Cloudflare Tunnel** | Expose local Jenkins for GitHub webhook during development |

---

# 🔄 CI/CD Pipeline

The Jenkins pipeline contains the following stages:

### 1. Checkout

Jenkins checks out the latest source code from GitHub.

```groovy
stage('Checkout') {
    steps {
        checkout scm
    }
}
```

---

### 2. Build & Test

The Maven Wrapper is used to build the application and execute tests.

```bash
./mvnw clean test
```

This ensures that the application passes its automated tests before continuing.

---

### 3. SonarQube Analysis

Jenkins sends the project to SonarQube for static/code-quality analysis.

```groovy
withSonarQubeEnv('SonarQube') {
    sh './mvnw sonar:sonar'
}
```

The analysis report is uploaded to SonarQube.

---

### 4. Quality Gate

Jenkins waits for the SonarQube Quality Gate result.

```groovy
timeout(time: 5, unit: 'MINUTES') {
    waitForQualityGate abortPipeline: true
}
```

If the quality gate fails, the pipeline stops before packaging and deployment.

> **Note:** During development, a SonarQube Compute Engine task remained `PENDING` long enough for Jenkins' five-minute timeout. This is a known follow-up item to investigate.

---

### 5. Package

The Spring Boot application is packaged into a JAR.

```bash
./mvnw package -DskipTests
```

Tests are skipped here because they were already executed during the Build & Test stage.

---

### 6. Docker Build

A Docker image is created using the Dockerfile.

```bash
docker build -t cicd-app:${BUILD_NUMBER} .
```

The Jenkins build number is used as the image version.

For example:

```text
cicd-app:25
```

---

# 🐳 Dockerfile

A multi-stage Docker build is used.

```dockerfile
# ---------- Build stage ----------
FROM eclipse-temurin:21-jdk AS build

WORKDIR /app

COPY .mvn/ .mvn/
COPY mvnw pom.xml ./

RUN chmod +x mvnw
RUN ./mvnw dependency:go-offline

COPY src ./src

RUN ./mvnw clean package -DskipTests


# ---------- Runtime stage ----------
FROM eclipse-temurin:21-jre

WORKDIR /app

COPY --from=build /app/target/*.jar app.jar

EXPOSE 8081

ENTRYPOINT ["java", "-jar", "app.jar"]
```

### Why multi-stage Docker?

The build stage contains the JDK and Maven requirements.

The runtime stage contains only the JRE and application JAR.

This keeps the final runtime image cleaner and avoids including build tools in the production container.

---

# 🔐 Trivy Security Scan

After building the Docker image, Trivy scans it for vulnerabilities.

```bash
trivy image \
  --timeout 20m \
  --scanners vuln \
  --severity HIGH,CRITICAL \
  --exit-code 1 \
  cicd-app:${BUILD_NUMBER}
```

### Configuration

- Vulnerability scanning enabled
- HIGH and CRITICAL vulnerabilities selected
- 20-minute timeout
- `--exit-code 1` causes Jenkins to fail if matching vulnerabilities are detected

During development, Trivy initially encountered a scanning timeout. The timeout was increased and scanning was restricted to vulnerability scanning.

A later scan completed but reported HIGH vulnerabilities associated with the `pebble` binary, which correctly stopped the pipeline because `--exit-code 1` was configured.

---

# 📦 Docker Hub

If the Trivy scan succeeds, Jenkins logs into Docker Hub and pushes the image.

Example:

```text
iamabhshek/cicd-app:25
```

The Docker Hub credentials are stored in Jenkins Credentials using:

```text
credentialsId: dockerhub-credentials
```

Credentials are not stored directly in the Jenkinsfile.

---

# ☁️ Oracle Cloud Deployment

The application is deployed to an Oracle Cloud VM.

The Oracle VM is used only as the **remote deployment server**.

```text
Oracle VM
├── Docker service
└── cicd-app container
```

Jenkins, SonarQube, Maven and Trivy remain on the Jenkins/WSL environment.

### Oracle VM

The Always Free ARM A1 Flex shape was initially attempted but was unavailable due to host capacity.

An Always Free E2.1.Micro VM was used instead.

The VM was configured with Docker and verified manually before automated deployment.

---

# 🔑 SSH Deployment

Jenkins connects to Oracle using an SSH credential:

```text
Credential ID: oracle-vm-ssh
Username: ubuntu
```

The Oracle public IP is stored in Jenkins as:

```text
ORACLE_HOST
```

The IP is therefore not hardcoded in the GitHub repository.

---

# 🚀 Remote Deployment Process

Jenkins performs the following operations on Oracle:

### Pull image

```bash
docker pull iamabhshek/cicd-app:${BUILD_NUMBER}
```

### Save previous image

Before replacing the running container, Jenkins checks:

```bash
docker inspect cicd-app
```

The previous image is stored in:

```text
/tmp/cicd-previous-image.txt
```

### Stop old container

```bash
docker stop cicd-app || true
docker rm cicd-app || true
```

### Start new container

```bash
docker run -d \
  --name cicd-app \
  --restart unless-stopped \
  -p 8081:8081 \
  iamabhshek/cicd-app:${BUILD_NUMBER}
```

---

# ❤️ Health Check

After deployment, Jenkins checks the application remotely.

```bash
curl --fail --silent http://localhost:8081/hello
```

The check is repeated up to 12 times with a 5-second interval.

That provides approximately 60 seconds for the application to start.

This is important because the application takes time to initialize on the small Oracle VM.

---

# 🔙 Rollback

If the new deployment fails the health check:

1. Jenkins connects to Oracle.
2. Reads the previous image.
3. Stops the failed container.
4. Removes the failed container.
5. Starts the previous image.
6. Reports the deployment as failed after restoring the previous version.

Example previous image:

```text
iamabhshek/cicd-app:24
```

New image:

```text
iamabhshek/cicd-app:25
```

If version `25` fails:

```text
25 ❌
 ↓
rollback
 ↓
24 ✅
```

---

# 🧹 Docker Cleanup

After a successful pipeline, Jenkins removes dangling Docker images on the Jenkins host:

```bash
docker image prune -f
```

This cleanup is performed on the Jenkins machine.

It does **not** remove the currently deployed application image from Oracle.

---

# 🔐 Jenkins Credentials

The project uses Jenkins Credentials instead of putting secrets inside the repository.

| Credential | Purpose |
|---|---|
| `dockerhub-credentials` | Docker Hub authentication |
| `oracle-vm-ssh` | SSH authentication to Oracle VM |

The Oracle VM address is stored separately as:

```text
ORACLE_HOST
```

### Never commit

Do not commit:

- SSH private keys
- Docker Hub passwords
- API tokens
- Cloud credentials
- `.env` files containing secrets

---

# 🌐 GitHub Webhook

GitHub triggers Jenkins after a push.

The webhook endpoint is:

```text
https://YOUR-JENKINS-URL/github-webhook/
```

During development, a Cloudflare Quick Tunnel was used to expose the local Jenkins server.

### Important

Cloudflare Quick Tunnel URLs can change when the tunnel is recreated.

If the URL changes, update:

1. GitHub → Repository → Settings → Webhooks
2. Jenkins → Manage Jenkins → System → Jenkins Location → Jenkins URL

A future improvement would be using a stable named Cloudflare Tunnel with a custom domain.

---

# 💻 Developer Workflow

Once the CI/CD system is configured, a developer does not need to manually deploy the application.

The normal workflow becomes:

```bash
git clone <repository-url>

cd ci-cd-Devops

# Make code changes

git add .
git commit -m "update application"
git push origin master
```

Then:

```text
git push
   ↓
GitHub
   ↓
Webhook
   ↓
Jenkins
   ↓
Automated CI/CD
   ↓
Oracle VM
```

---

# 🧪 Manual Verification Commands

### Check Docker

```bash
docker ps
```

### View application logs

```bash
docker logs cicd-app
```

### Test application

```bash
curl http://localhost:8081/hello
```

### SSH into Oracle

```bash
ssh -i ~/.ssh/ssh-key-2026-08-16.key ubuntu@<ORACLE_PUBLIC_IP>
```

### Pull image manually

```bash
docker pull iamabhshek/cicd-app:20
```

### Run manually

```bash
docker run -d \
  --name cicd-app \
  --restart unless-stopped \
  -p 8081:8081 \
  iamabhshek/cicd-app:20
```

---

# 📊 Final Architecture

```text
                         ┌─────────────┐
                         │   GitHub    │
                         └──────┬──────┘
                                │
                              Webhook
                                │
                                ▼
                     ┌────────────────────┐
                     │ Jenkins / WSL      │
                     │                    │
                     │ Maven              │
                     │ SonarQube          │
                     │ Docker             │
                     │ Trivy              │
                     └─────────┬──────────┘
                               │
                        Docker Hub Push
                               │
                               ▼
                       ┌───────────────┐
                       │  Docker Hub   │
                       └───────┬───────┘
                               │
                         SSH + Pull
                               │
                               ▼
                    ┌────────────────────┐
                    │ Oracle Cloud VM    │
                    │                    │
                    │ Docker             │
                    │                    │
                    │ cicd-app           │
                    │ :8081              │
                    └─────────┬──────────┘
                              │
                         Health Check
                              │
                              ▼
                         /hello
```

---

# 🎯 What Was Achieved

The project progressed from a local application to an automated remote CI/CD deployment system.

### Completed

- [x] GitHub repository
- [x] Jenkins pipeline
- [x] Maven build and tests
- [x] SonarQube analysis
- [x] SonarQube Quality Gate integration
- [x] Docker multi-stage build
- [x] Trivy vulnerability scanning
- [x] Docker Hub image publishing
- [x] Jenkins credentials
- [x] SSH authentication
- [x] Oracle Cloud VM
- [x] Remote Docker deployment
- [x] Remote health check
- [x] Rollback mechanism
- [x] Docker cleanup
- [x] GitHub webhook integration

### Follow-up improvements

- [ ] Investigate SonarQube Quality Gate tasks that remain `PENDING`
- [ ] Improve vulnerability remediation for Trivy findings
- [ ] Clean old Docker images on the Oracle VM with a retention policy
- [ ] Replace Cloudflare Quick Tunnel with a stable named tunnel/domain
- [ ] Improve deployment strategy toward zero/minimal downtime
- [ ] Add monitoring and application metrics

---

## 📌 Project Summary

This repository demonstrates a practical CI/CD implementation in which code pushed to GitHub can automatically progress through testing, quality analysis, security scanning, containerization, image publishing and remote deployment.

The final deployment target is an Oracle Cloud VM running Docker, while Jenkins remains responsible for orchestrating the CI/CD workflow.
