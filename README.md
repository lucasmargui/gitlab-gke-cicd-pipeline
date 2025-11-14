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

# ☁️ Authenticate GitLab Runner to Google Cloud

This section explains how to authenticate the GitLab Runner (running on your VM/Bastion Host) with Google Cloud, allowing the runner to interact with GKE or other GCP services during CI/CD pipelines.

## Step 1 – Switch User

Switch to the gitlab-runner user so that all Google Cloud credentials are stored under this user’s home directory.
This ensures correct permissions during pipeline execution.
```
su gitlab-runner
```

## Step 2 – Configure GCloud Account

Set your Google Cloud account and initialize the gcloud CLI.
This step configures the project, region, and other defaults for future commands.
```
gcloud config set account account-name@gmail.com
gcloud init
```

During gcloud init, follow the prompts to:

- Choose or log into your Google account
- Select the GCP project
- Set the default compute region/zone

## Step 3 – Authenticate

Authenticate the GitLab Runner to Google Cloud to enable secure access to GKE or other services.
```
gcloud auth login
```

☸️ Connect to GKE Cluster

This section explains how to configure your GitLab Runner (or any VM acting as a Bastion) to connect securely to your Google Kubernetes Engine (GKE) cluster.

## Step 1 – Retrieve Credentials

From the GKE Console → Your Cluster → Three-dots menu → Connect, copy the authentication command provided by Google Cloud.

Use the command to fetch cluster credentials and update your local kubeconfig:
```
gcloud container clusters get-credentials <cluster-name> --region <region> --project <project-id>
```

This step ensures that your GitLab Runner can authenticate and communicate with the GKE API.

## Step 2 – Install GCloud Components (if required)

Some environments require the GKE gcloud auth plugin for kubectl authentication.

Install the components:
```
gcloud components install gke-gcloud-auth-plugin
sudo apt-get install google-cloud-sdk-gke-gcloud-auth-plugin
```
This plugin enables kubectl to authenticate through gcloud for GKE clusters.

## Step 3 – Validate Access

Verify that your GitLab Runner can successfully communicate with the GKE cluster:

```
kubectl get nodes
```

If you see the list of nodes, the connection is fully working.

# 📦 Deploy Pipeline to a Kubernetes Cluster

This section explains how to build Docker images for multiple microservices, push them to Docker Hub, and then deploy them into a GKE Kubernetes cluster using GitLab CI/CD.

---

## Step 1 – Docker Login on the Bastion Host

Before running the pipeline, the GitLab Runner (running on your GCP bastion VM) must be authenticated to Docker Hub.

This is required to allow the runner to build and push images.

```sh
docker login
```

## Step 2 – Creating Microservice Directories

To demonstrate how microservices are built, packaged, and deployed, we will create two independent microservices.
Each service will run its own HTTP server and contain its own HTML response—simulating isolated, scalable components.

Inside your project directory Bootcamp-DevOps-Project-2/, create the following structure:

Create the directories:
```
app1/
app2/
```

Each directory will contain:

- A Dockerfile running a simple httpd server
- A small HTML file simulating independent microservices

```
app1/
 ├── Dockerfile
 └── index.html

app2/
 ├── Dockerfile
 └── index.html
```

## Step 3 – Create Dockerfiles for Each Microservice

Each microservice will run using a lightweight Apache HTTP Server (httpd) container.

Below are the Dockerfile examples:

### Example Dockerfile for each app:

#### app1/Dockerfile
```
FROM httpd:latest

WORKDIR /usr/local/apache2/htdocs/

COPY * /usr/local/apache2/htdocs/

EXPOSE 80
```

#### app2/Dockerfile
```
FROM httpd:latest

WORKDIR /usr/local/apache2/htdocs/

COPY * /usr/local/apache2/htdocs/

EXPOSE 8080
```

- app1 will serve content on port 80.
- app2 will serve content on port 8080 to illustrate independent services running in different ports.

### Example HTML (index.html):

Each microservice will return a simple web page:

#### app1/index.html
```
<h1>Microservice 1 - Running</h1>
```

#### app2/index.html
```
<h1>Microservice 2 - Running</h1>
```

## Step 4 – Creating the CI Pipeline to Build & Push Images

Inside your root project directory, create the pipeline file:
```
.gitlab-ci.yml
```

This pipeline will:

- Build the Docker image for app1
- Build the Docker image for app2
- Tag each image properly
- Push both images to Docker Hub (or your preferred registry)

This ensures each microservice is built and delivered independently—following the core principles of microservice architecture.

<details><summary>Click to show details</summary> <img width="906" height="405" alt="image" src="https://github.com/user-attachments/assets/1c6fca60-1819-4809-80a5-c56533079ca8" /> </details>



