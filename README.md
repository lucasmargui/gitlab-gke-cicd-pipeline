# 🚀 Bootcamp DevOps – GitLab Runner + Docker + GKE CI/CD Pipeline

A complete DevOps workflow demonstrating how to configure Docker, GitLab Runner, Google Cloud authentication, and deployment to a Kubernetes cluster on GKE using GitLab CI/CD.

---

## 📛 Badges

![Status](https://img.shields.io/badge/Status-Active-brightgreen)
![GitLab CI](https://img.shields.io/badge/GitLab-CI%2FCD-orange)
![GCP](https://img.shields.io/badge/Google%20Cloud-Enabled-blue)
![Docker](https://img.shields.io/badge/Docker-Ready-2496ED)
![Kubernetes](https://img.shields.io/badge/Kubernetes-GKE-326CE5)
![License](https://img.shields.io/badge/License-MIT-lightgrey)

---

# 📘 Table of Contents

- [📌 Overview](#-overview)
- [🖼️ Architecture Diagram](#️-architecture-diagram)
- [🐳 Add Docker to the Instance](#-add-docker-to-the-instance)
- [🧩 GitLab – Bastion VM Setup](#-gitlab--bastion-vm-setup)
- [🚀 Add and Configure GitLab Runner](#-add-and-configure-gitlab-runner)
- [☁️ Authenticate GitLab Runner to Google Cloud](#️-authenticate-gitlab-runner-to-google-cloud)
- [☸️ Connect to GKE Cluster](#️-connect-to-gke-cluster)
- [📦 CI/CD Pipeline Structure](#-cicd-pipeline-structure)
- [📜 Full .gitlab-ci.yml Example](#-full-gitlab-ciyml-example)
- [📂 Project Structure](#-project-structure)
- [🧾 License](#-license)

---

# 📌 Overview

This repository demonstrates a complete workflow:

- Install Docker on a GCP VM  
- Create a GitLab project  
- Install and configure GitLab Runner on GCP  
- Assign permissions (sudo, docker groups, sudoers file)  
- Authenticate GitLab Runner to Google Cloud  
- Connect to a GKE Kubernetes cluster  
- Deploy containers through GitLab CI/CD  

This repository is ideal for **DevOps bootcamps, CI/CD tutorials, and hands-on cloud workshops**.

---

# 🖼️ Architecture Diagram

> 🔧 *Replace with an actual image later — placeholder below.*

![Architecture Diagram](https://via.placeholder.com/1200x500.png?text=Architecture+Diagram+Here)

---

# 🐳 Add Docker to the Instance

## Step 1 – Download Docker Installer
```sh
curl -fsSL https://get.docker.com -o get-docker.sh
```

## Step 2 – Install Docker
```sh
sudo sh get-docker.sh
```

# 🧩 GitLab – Bastion VM Setup

## Step 1 – Create GitLab Project

Create a public GitLab project named:
```
Bootcamp-DevOps-Project-2
```

## Step 2 – Initialize Local Directory
Create a folder locally and include:
```
instruções.txt   (empty file)
```

## Step 3 – Push Files to GitLab
```
git add .
git commit -m "Initial commit"
git push
```

# 🚀 Add and Configure GitLab Runner

## Step 1 – Install GitLab Runner in Bastion

We use a Google VM as a bastion so the GitLab Runner can deploy to the cluster without exposing the Kubernetes API to the public internet.
The Runner runs inside this bastion VM, which already has private network access to the cluster.
This keeps the cluster private, secure, and only reachable from inside the VPC.

Install on the Instance 1 (called Bastion):

```
curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh | sudo bash
sudo apt-get install gitlab-runner -y
```

## Step 2 – Register the Runner in Bastion

In your GitLab project, navigate to:
Settings → CI/CD → Runners
Open the options menu (three dots) and copy the project’s registration token.

Use this token to register the GitLab Runner installed on the Bastion VM.
This links the runner on the bastion to your GitLab project and allows it to execute your CI/CD pipelines.

```
gitlab-runner register
```

## Step 3 – Grant Permissions

To ensure the GitLab Runner can build and run Docker containers, as well as execute privileged operations when required, you must assign the proper system permissions.

### Add the Runner to the Docker Group  

This allows the `gitlab-runner` user to execute Docker commands without requiring `sudo`.
```bash
sudo /sbin/usermod -G docker gitlab-runner
```

### Add the Runner to the Sudo Group

This grants permission for the runner to perform administrative tasks when needed during CI/CD jobs.
```
sudo /sbin/usermod -G sudo gitlab-runner
```

### Update Sudoers File

You need to explicitly allow the gitlab-runner user to run any command with sudo privileges.

Open the sudoers file:
```
sudo nano /etc/sudoers
```

Add the following line at the end:
```
gitlab-runner ALL=(ALL:ALL) ALL
```

### Set a Password for the Runner

Some operations may require authentication; define a password for the gitlab-runner account:
```
sudo passwd gitlab-runner
```

