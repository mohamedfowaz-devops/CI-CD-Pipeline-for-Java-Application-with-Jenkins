# Java CI/CD Pipeline with Jenkins, Docker, AWS ECR and Ansible

An end-to-end CI/CD pipeline for a Java (Spring Boot) application. Jenkins builds and tests the code with Maven, creates a versioned Docker image and pushes it to Amazon ECR, then uses Ansible to deploy the application JAR to two Ubuntu EC2 application servers and verify it with a health check.
<img width="994" height="502" alt="image" src="https://github.com/user-attachments/assets/51a3fea9-b6e2-4781-acf3-990816c49f84" />


## Highlights

- Declarative Jenkins pipeline (`Jenkinsfile`) covering checkout, build, test, image build, ECR push and deployment
- Docker images tagged with the Jenkins build number and stored in a private AWS ECR repository
- Ansible playbook that installs Java 21, deploys the JAR to the app servers and restarts the application
- Automated port check and `/actuator/health` verification after each deployment
- Documented break-fix scenarios covering failures encountered while building and running the pipeline

## Tools & Technologies

| Area | Tools |

Language & Build - Java 21, Maven 
Source Control - Git, GitHub 
CI/CD - Jenkins 
Containers - Docker 
Cloud - AWS EC2, AWS ECR, AWS IAM 
Configuration Management - Ansible 
OS & Scripting - Ubuntu Linux, Shell scripting 

## CI/CD Workflow

```
GitHub
   ↓
Jenkins
   ↓
Maven Build, Test & Package
   ↓
Docker Build  →  Docker Image Check
   ↓
Push Image to AWS ECR
   ↓
Ansible Deployment (JAR)
   ↓
App Server 1 / App Server 2
   ↓
Port Check + Health Check
```

## Project Structure

```
CI-CD-Pipeline-for-Java-Application-with-Jenkins/
│
├── java-cicd/
│   ├── src/
│   ├── pom.xml
│   ├── Dockerfile
│   └── Jenkinsfile
│
├── ansible/
│   ├── inventory
│   └── deploy.yml
│
└── README.md
```

## Pipeline Stages

 Stage | What it does |

| 1 | **Checkout** - Jenkins pulls the source code from GitHub 
| 2 | **Build and Test** - Maven runs `clean package`, executes unit tests and publishes the artifact with `mvn deploy` 
| 3 | **Docker Build** -Builds the image `java-cicd-demo:<BUILD_NUMBER>` from an `eclipse-temurin:21-jre` base image 
| 4 | **Docker Image Check** - Confirms the image exists with the expected tag 
| 5 | **Push to ECR** - Logs in to ECR, tags the image and pushes it to the `java-cicd-demo` repository (`us-west-2`) 
| 6 | **Deploy with Ansible** - Runs the playbook against the `app_servers` inventory group 

## Deployment with Ansible

The inventory defines two application servers (`app1`, `app2`) in the `app_servers` group. For each server the playbook:

1. Installs Java 21 (`openjdk-21-jre`)
2. Creates the `/opt/java-cicd` application directory
3. Stops the existing application process
4. Copies the new JAR from the Jenkins workspace
5. Starts the application with `java -jar`, logging to `/opt/java-cicd/app.log`
6. Waits for port `8080` to open
7. Verifies `http://127.0.0.1:8080/actuator/health` returns HTTP 200

Run manually:

```bash
ansible-playbook -i ansible/inventory ansible/deploy.yml
```

## AWS Components
**EC2** – hosts Jenkins and the two application servers
**ECR** – private Docker image repository (`java-cicd-demo`)
**IAM** – permissions that allow Jenkins to authenticate to ECR and push images

## Break-Fix Scenarios
### 1. Maven build or unit tests failed
**Issue:** The Jenkins pipeline stopped at *Build and Test* with `BUILD FAILURE` and failing tests.
 **Root cause:** A code or test error, a Java/Maven version mismatch on the Jenkins server, or a dependency download failure.
 **Resolution:** Reviewed the console output, verified `java -version` and `mvn -version` against the project requirements, reproduced the failure locally with `mvn clean test`, fixed the cause and re-triggered the build.

### 2. Jenkins could not run Docker (permission denied)
**Issue:** The *Docker Build* stage failed with `permission denied while trying to connect to the Docker daemon socket`.
**Root cause:** The `jenkins` user was not in the `docker` group, or the Docker service was stopped.
**Resolution:** Added the `jenkins` user to the `docker` group, restarted Docker and Jenkins, and verified the build ran successfully.
  ```bash
  sudo usermod -aG docker jenkins
  sudo systemctl restart docker
  sudo systemctl restart jenkins
  ```

### 3. Docker push to ECR failed (unauthorized)
**Issue:** The *Push to ECR* stage failed with `no basic auth credentials` or `denied: not authorized`.
**Root cause:** The ECR login token had expired, the IAM permissions were missing, or the region or account ID was wrong.
**Resolution:** Verified the region and account ID, confirmed the IAM policy included the ECR push actions, re-authenticated to ECR and re-ran the pipeline.
  ```bash
  aws ecr get-login-password --region us-west-2 | \
    docker login --username AWS --password-stdin <account-id>.dkr.ecr.us-west-2.amazonaws.com
  ```

### 4. Ansible could not reach the app servers (SSH failure)
**Issue:** Ansible reported `UNREACHABLE! Permission denied (publickey)` or a connection timeout.
**Root cause:** A wrong key or user in the inventory, key file permissions that were too open, or a security group that did not allow SSH from the Jenkins server.
**Resolution:** Verified the inventory user and key path, set `chmod 400` on the key file, allowed port 22 from the Jenkins server in the security group, and confirmed connectivity with `ansible all -i ansible/inventory -m ping`.

### 5. JAR file not found during deployment
**Issue:** The *Copy application JAR* task failed because the source file did not exist.
**Root cause:** The Maven build did not produce `java-cicd-demo-1.0.0.jar`, or the workspace path or JAR version in the playbook did not match the build output.
**Resolution:** Checked the `target/` directory in the Jenkins workspace, confirmed the artifact name and version in `pom.xml`, corrected the path in `deploy.yml` and re-ran the pipeline.

### 6. Application started but the health check failed
**Issue:** The wait-for-port or `/actuator/health` task failed after the application was started.
**Root cause:** An application crash on startup, the wrong Java version, the Actuator dependency missing, or port 8080 blocked by a security group.
**Resolution:** Checked the application log and running process, tested the endpoint locally, corrected the cause and re-ran the pipeline.
  ```bash
  tail -n 100 /opt/java-cicd/app.log
  ps -ef | grep app.jar
  curl -i http://127.0.0.1:8080/actuator/health
  ```

### 7. Port 8080 already in use
**Issue:** The new application failed to start because port 8080 was already occupied.
**Root cause:** The previous Java process was not stopped before the new one started.
**Resolution:** Identified the process holding the port, stopped it, and confirmed the playbook's stop task matched the JAR path.
  ```bash
  sudo ss -tulpn | grep 8080
  sudo pkill -f '/opt/java-cicd/app.jar'
  ```

### 8. Server disk full from old images and logs
**Issue:** Docker builds failed with `no space left on device` on the Jenkins server.
**Root cause:** Old build-number images and build artifacts had accumulated.
**Resolution:** Checked disk usage, pruned unused Docker images and cleaned old workspaces.
  ```bash
  df -h
  docker image prune -a -f
  ```

### 9. Jenkinsfile syntax error
**Issue:** The pipeline failed before running any stage with a declarative syntax error.
**Root cause:** A stage was nested inside another stage's block, which declarative pipelines do not allow.
**Resolution:** Moved the stage to the same level as the others, validated the Jenkinsfile and re-ran the pipeline.

## Key Learning

- Writing Jenkins declarative pipelines
- Maven builds, testing and packaging
- Building and versioning Docker images
- Pushing images to AWS ECR with IAM-based access
- Ansible playbooks, inventories and application deployment
- Application health checks as a deployment gate
- Linux server administration
- Diagnosing and fixing failures at each stage of a CI/CD pipeline

## Future Improvements

- Deploy servers one at a time with `serial: 1` for a true rolling deployment
- Run the application as a systemd service so it survives reboots
- Deploy the ECR image as a container on the app servers instead of a JAR
- Store the AWS account ID and region as Jenkins environment variables
