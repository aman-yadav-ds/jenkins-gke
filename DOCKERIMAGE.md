
# 🚀 Setting up a Jenkins Docker-in-Docker (DinD) Container

This guide walks you through the steps to create a custom Jenkins Docker image that includes the necessary tools to run Docker commands from within Jenkins pipelines.

---

## 📁 1. Create the Dockerfile

### Step 1: Set up the project directory

Create a new folder for your project (e.g., `custom_jenkins`) and a `Dockerfile` inside it.

```bash
mkdir custom_jenkins
cd custom_jenkins
touch Dockerfile
```

### Step 2: Add the following content to your `Dockerfile`

```dockerfile
# Use the official Jenkins LTS image as the base
FROM jenkins/jenkins:lts

# Switch to the root user to install Docker and other dependencies
USER root

# Install necessary prerequisites and the Docker CLI
RUN apt-get update -y &&     apt-get install -y apt-transport-https ca-certificates curl gnupg software-properties-common &&     curl -fsSL https://download.docker.com/linux/debian/gpg | apt-key add - &&     echo "deb [arch=amd64] https://download.docker.com/linux/debian bullseye stable" > /etc/apt/sources.list.d/docker.list &&     apt-get update -y &&     apt-get install -y docker-ce docker-ce-cli containerd.io &&     apt-get clean

# Add the Jenkins user to the Docker group
RUN groupadd -f docker &&     usermod -aG docker jenkins

# Create Docker directory and volume for DinD
RUN mkdir -p /var/lib/docker
VOLUME /var/lib/docker

# Switch back to the Jenkins user for best practices
USER jenkins
```

---

## 🛠️ 2. Build and Verify the Docker Image

### Step 1: Open your terminal and navigate to the project directory

```bash
cd custom_jenkins
```

### Step 2: Build the Docker image

```bash
docker build -t jenkins-dind-recommender .
```

### Step 3: Verify the image was created

```bash
docker images
```

Look for `jenkins-dind-recommender` in the list of images. If it's there, your custom Jenkins DinD image is ready to use! 🎉

---

## ✅ Summary

You've now:

- Created a custom Jenkins Dockerfile
- Installed Docker inside Jenkins
- Built and verified the image

You can now use this image in your pipelines to run Docker commands from within Jenkins safely.

---
