# 🚀DevOps – GitLab Runner + Docker + GKE CI/CD Pipeline

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

- [📐 Architecture Diagram](#-architeture-diagram)
- [📌 Overview](#-overview)
- [🐳 Add Docker to the Instance](#-add-docker-to-the-instance)
- [🧩 GitLab – Bastion VM Setup](#-gitlab--bastion-vm-setup)
- [🚀 Add and Configure GitLab Runner](#-add-and-configure-gitlab-runner)
- [🌥 Authenticate GitLab Runner to Google Cloud](#-authenticate-gitlab-runner-to-google-cloud)
- [🌥 Connect to GKE Cluster](#-connect-to-gke-cluster)  
- [📦 Deploy Pipeline to a Kubernetes Cluster](#-deploy-pipeline-to-a-kubernetes-cluster)  
- [🌥 Publishing Applications to the Kubernetes Cluster](#-publishing-applications-to-the-kubernetes-cluster)  
- [🚦 Creating a LoadBalancer for External Access](#-creating-a-loadbalancer-for-external-access)  
- [🧪 Load Testing Your Microservices with Loader.io](#-load-testing-your-microservices-with-loaderio)  

---



# 📐 Architeture Diagram

<img width="1276" height="677" alt="image" src="https://github.com/user-attachments/assets/499302aa-c119-4e1c-99ec-03d97f5fabaa" />


This diagram represents a **CI/CD (Continuous Integration / Continuous Deployment) workflow** using GitLab, Docker, and GKE (Google Kubernetes Engine). Each component of the workflow is detailed below.

## 🛠 Components

### GitLab
- Acts as the starting point of the workflow.
- Stores the project's source code.
- Contains CI/CD pipelines to automate building, testing, and deployment.

### Bastion VM
- A virtual machine that serves as a secure bridge between GitLab and Google Cloud infrastructure.
- Hosts the **GitLab Runner** to execute pipeline jobs.

### Docker
- Installed on the Bastion VM to build container images from the project code.
- These containers are deployed and run in the production environment.

### Load Balancer
- Distributes incoming network traffic across multiple container instances or GKE nodes.
- Ensures high availability and load balancing.

### GKE Cluster & Nodes
- The **GKE Cluster** is the Kubernetes environment where Docker containers are deployed.
- **GKE Nodes** are the underlying virtual or physical machines running the pods with the project’s containers.

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

---

# 🐳 Add Docker to the Instance

To enable containerized workloads on your VM, Docker must be installed and configured.
In this step, we download Docker using the official convenience script and install it directly on the instance.

<details><summary>Click to show details</summary> 

## Step 1 – Download Docker Installer
```sh
curl -fsSL https://get.docker.com -o get-docker.sh
```

## Step 2 – Install Docker
```sh
sudo sh get-docker.sh
```

</details>

---

# 🧩 GitLab – Bastion VM Setup

Before connecting GitLab CI/CD to your Kubernetes cluster, you must configure a Bastion VM.
This VM acts as a secure bridge between GitLab Runner and your GKE cluster, ensuring controlled access and centralized deployment operations.

- The Bastion VM will:
- Host the GitLab Runner
- Hold Kubernetes credentials (kubeconfig)
- Execute kubectl commands triggered by your GitLab pipeline
- Provide an isolated and secure environment for deployments

<details><summary>Click to show details</summary>  

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

</details>

---

# 🚀 Add and Configure GitLab Runner

This section guides you through installing, registering, and configuring a GitLab Runner. The Runner acts as the bridge between your GitLab CI/CD pipelines and your infrastructure, executing jobs and deploying your applications efficiently. We'll cover adding the Runner to your system, granting necessary permissions, and connecting it to your GitLab project for seamless automation.

<details><summary>Click to show details</summary> 

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

</details>

---

# 🌥 Authenticate GitLab Runner to Google Cloud

This section explains how to authenticate the GitLab Runner (running on your VM/Bastion Host) with Google Cloud, allowing the runner to interact with GKE or other GCP services during CI/CD pipelines.

<details><summary>Click to show details</summary>  

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

</details>

---

# 🌥 Connect to GKE Cluster

This section explains how to configure your GitLab Runner (or any VM acting as a Bastion) to connect securely to your Google Kubernetes Engine (GKE) cluster.

<details><summary>Click to show details</summary> 

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

</details>

---

# 📦 Deploy Pipeline to a Kubernetes Cluster

This section explains how to build Docker images for multiple microservices, push them to Docker Hub, and then deploy them into a GKE Kubernetes cluster using GitLab CI/CD.

<details><summary>Click to show details</summary> 

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

</details>

---

# 🌥 Publishing Applications to the Kubernetes Cluster

Once the CI pipeline completes the build and pushes the Docker images to your registry, the next step is deploying both microservices into the GKE cluster.
This section shows how to create Kubernetes Deployment manifests and automate the deployment through GitLab CI/CD.

<details><summary>Click to show details</summary>  

## Step 1 – Create Kubernetes Deployment Files

Inside the root directory of the repository, create two deployment files:

```
deploy-micro1.yml
deploy-micro2.yml
```

Each file will define the Kubernetes Deployment specification, including:

- The number of replicas (desired number of Pods)
- The container image to pull from Docker Hub
- Labels used for identification and service selection
- Container port exposed inside the Pod


#### Example: deploy-micro1.yml
```
apiVersion: apps/v1 
kind: Deployment 
metadata: 
 name: micro1-deployment
 labels: 
   app: micro1 
spec: 
 replicas: 3 
 selector: 
   matchLabels: 
     app: micro1 
 template: 
   metadata: 
     labels: 
       app: micro1 
   spec: 
     containers: 
     - name: micro1 
       image: <dockerhub-username>/micro1:latest
       ports: 
       - containerPort: 80 
```

#### Example: deploy-micro2.yml

```
apiVersion: apps/v1 
kind: Deployment 
metadata: 
 name: micro2-deployment
 labels: 
   app: micro2 
spec: 
 replicas: 2 
 selector: 
   matchLabels: 
     app: micro2 
 template: 
   metadata: 
     labels: 
       app: micro2 
   spec: 
     containers: 
     - name: micro2 
       image: <dockerhub-username>/micro2:latest
       ports: 
       - containerPort: 8080
```


📌 What Do the “Replicas” and “Pods” Mean?

A Pod is the smallest deployable unit in Kubernetes.
It represents one running instance of your containerized application.

The replicas field determines how many Pods of that microservice Kubernetes should keep running.

In our example:

- micro1 has replicas: 3
  → Kubernetes will create 3 Pods, each running the micro1 container.

- micro2 has replicas: 2
  → Kubernetes will create 2 Pods, each running the micro2 container.

If one Pod stops working, Kubernetes will automatically create a new one to maintain the desired replica count.
This ensures high availability and resilience.

#### Example scenario

Imagine micro1 receives more traffic.
Because it has 3 Pods, Kubernetes can distribute requests across them (load balancing), avoiding overload on a single instance.


## Step 2 – Add Kubernetes Deployment Job to .gitlab-ci.yml

Add the deployment stage to your pipeline so it automatically applies the manifests to your GKE cluster:
```
kubernetes:
  stage: deploy
  needs:
    - criar_imagens
  tags:
    - gcp
  script:
  - kubectl apply -f ./deploy-micro1.yml
  - kubectl apply -f ./deploy-micro2.yml
```

Your pipeline now consists of two main phases:

- Build – Builds and pushes images
- Deploy – Applies Kubernetes manifests to the GKE cluster


## Step 3 – Commit and Push Deployment Configuration

```
git add .
git commit -m "Add Kubernetes deployment pipeline"
git push
```

## Step 4 – Validate the Pods Running in the Cluster

After the deployment job completes, connect to your bastion / GitLab Runner VM and run:
```
kubectl get pods
```

Expected result:

- 3 pods running for microservice 1
- 2 pods running for microservice 2

This confirms both microservices are successfully deployed and running.

</details>

---

# 🚦 Creating a LoadBalancer for External Access

After deploying both microservices to the Kubernetes cluster, they are currently reachable only inside the cluster’s internal network.
To make them accessible from the internet, we need to expose them externally — and the standard way to do this in GKE is by using a Service of type LoadBalancer.

<details><summary>Click to show details</summary> 


## Step 1 — Why We Need a LoadBalancer

We currently have:

 - 3 nodes (VMS) inside the GKE cluster
 - Microservice 1 running with 3 replicas (pods)
 - Microservice 2 running with 2 replicas (pods)

Because a Deployment distributes Pods across multiple nodes, we need one single public entry point capable of forwarding traffic to all replicas.

That’s exactly the job of a LoadBalancer:

- It exposes the service to the public internet
- Kubernetes distributes requests across replica pods
- Each request may hit a different pod in a round-robin style

Client Request → LoadBalancer → Pod 1 / Pod 2 / Pod 3

This ensures availability, scalability, and fault tolerance — if one Pod fails, Kubernetes automatically routes traffic to the remaining replicas.

<img width="2029" height="711" alt="image" src="https://github.com/user-attachments/assets/2b6e9e36-ea58-45ef-a01d-23355f0df3e8" />

## Step 2 — Creating the LoadBalancer for Microservice 1

Create a file named 'lb.yml' and inside it:

```
apiVersion: v1 
kind: Service 
metadata: 
  name: lb-micro1 
spec: 
  selector: 
    app: micro1 
  ports: 
    - port: 80 
      targetPort: 80 
  type: LoadBalancer 
```

- selector.app: micro1
  Must match the app: micro1 label from the deployment, so Kubernetes knows which Pods to route traffic to.

- port / targetPort: 80
  Exposes port 80 externally and forwards it to port 80 inside each Pod.

- type: LoadBalancer
  In GKE, this automatically provisions a Google Cloud external IP, making the service publicly accessible.

## Step 4 — Updating the Pipeline to Apply the LoadBalancer

Edit your .gitlab-ci.yml and ensure the deploy stage includes:
```
kubectl apply -f ./lb.yml
```
This ensures the LoadBalancer is created automatically during the deploy stage.

## Step 5 — Commit and Push to Trigger the Pipeline

```
git add .
git commit -m "Add LoadBalancer service for micro1"
git push
```
Once pushed, the pipeline will deploy the LoadBalancer to the cluster.

## Step 6 — Validating the LoadBalancer

After the pipeline completes, log into your bastion (GitLab Runner VM) and run:
```
kubectl get service
```

You can now access your microservice through the external IP:

```
http://34.94.15.128
```

Every time you refresh this URL:

- Kubernetes forwards each request to one of the available Pods
- You may hit Pod A, B, or C depending on the round-robin behavior
- If you print the Pod name inside the HTML response, you can visually observe load balancing in action

</details>

---

# 🧪 Load Testing Your Microservices with Loader.io

After exposing your microservices through a Kubernetes LoadBalancer, the next crucial step is performance testing.
Load testing validates:

- How your applications behave under heavy traffic
- Whether your infrastructure scales properly
- How replicas and nodes respond to stress
- If your architecture is production-ready

Below is a complete, polished guide on integrating Loader.io into your Kubernetes workflow.

<details><summary>Click to show details</summary> 

## Step 1 — Creating Your Loader.io Account

1. Access https://loader.io
2. Create an account and log in
3 Add a new Target Host
  - This will be the external IP of your LoadBalancer
  - Example: http://34.94.15.128

This is the endpoint Loader.io will stress test.

## Step 2 — Verifying Domain Ownership

Loader.io requires verification to ensure the tested endpoint belongs to you.
You will receive a verification token, for example:

```
loaderio-123456789abcdef.txt
```

### Create the verification file in microservice 1

Place the token file inside microservice 1 (app1):

```
app1/loaderio-123456789abcdef.txt
```
This ensures Loader.io can validate your service.

## Step 3 — Building Test-Specific Docker Images

Since the verification file must be packaged inside the image, you need a new Docker image dedicated to testing.
1. Update your Dockerfile to include the new token file.
2. Build the image with a new tag, for example test.

Update your .gitlab-ci.yml to build your test images:

```
variables:
  VERSION: "test"
```

## Step 4 — Updating Deployments to Use the Test Image

Modify the deployment file for microservice 1:

Update your .deploy-micro1.yml to build your test images:
```
containers: 
- name: micro1 
  image: <dockerhub-username>/micro1:test
  ports: 
  - containerPort: 80
```

Update your .deploy-micro2.yml to build your test images:
```
containers: 
- name: micro2 
  image: <dockerhub-username>/micro2:test
  ports: 
  - containerPort: 8080
```

These new images now contain the Loader.io verification file.

## Step 5 — Commit and Push to Run the Pipeline

```
git add .
git commit -m "Add Loader.io token and test images"
git push
```

GitLab CI/CD will:

- Build the test images
- Push them to Docker Hub
- Deploy updated pods to the Kubernetes cluster

You can confirm new pods were created using:

```
kubectl get pods
```

## Step 6 — Verifying the Loader.io Token

Back on Loader.io:

- Go to Verification
- Click Verify

Loader.io requests your LoadBalancer endpoint and checks the presence of the token file inside the microservice.
Once verified, you can proceed to test execution.

## Step 7 — Running Your Load Tests

Now you can execute load tests using Loader.io profiles such as:
- Linear load
- Spike traffic
- Clients per second
- Clients per minute

Evaluate:
- Response time
- Error rates
- Timeouts
- Throughput

Example test scenarios

### Test #1 — 500 clients per minute
Good for baseline performance validation.

### Test #2 — 10,000 clients per minute
Used to evaluate scaling behavior and infrastructure resilience.

If your application maintains low latency and zero errors under high load, your cluster is correctly handling traffic.

## Step 8 — Scaling if Needed

After running your load tests, you may determine that your current setup is not handling traffic efficiently.
Kubernetes allows two primary scaling strategies:

- Scaling Pods (application-level scaling)
- Scaling Nodes (infrastructure-level scaling)

Both serve different purposes and should be used depending on the performance bottleneck observed.

### 🟦 Option A — Increase Pod Replicas (Horizontal Scaling of Workloads)

This option scales your application, not the cluster.

If your microservice needs to handle more concurrent requests, you can increase the number of Pods running the same container image.

Pods are the actual running instances of your application — adding more replicas increases your parallel processing capacity.

#### 🔧 How to Scale Pods

Update the Deployment file (deploy-micro1.yml):

```
spec:
  replicas: 6
```

Then apply your standard Git flow:

```
git add .
git commit -m "Increase replicas for microservice 1"
git push
```

Kubernetes will then:

- Create additional Pods (from 3 → 6, in this example)
- Distribute them across available nodes
- Update the LoadBalancer to route traffic evenly across all Pods

#### 📌 When to Use This Option

Choose Pod scaling when:

- CPU usage per Pod is high
- Requests are timing out even though nodes have available resources
- Your application needs more "instances" to handle traffic

This is typically the first and most common scaling action.

### 🟩 Option B — Increase Kubernetes Nodes (Scaling the Cluster Infrastructure)

Scaling nodes expands the infrastructure capacity of your cluster, not the number of Pods directly.

#### 🛑 Important Clarification

Adding more nodes does NOT automatically increase the number of Pods.
It only increases the available compute resources (CPU, RAM) the cluster can use.

Pods will only increase if:

- You manually increase replica counts
or
- You use Horizontal Pod Autoscaler (HPA)

So a node is NOT a “container” — it's a VM that hosts Pods.

#### 🔧 How to Scale Nodes

If you determine the cluster is running out of CPU/RAM capacity:

- Add more nodes to the node pool
- Or enable cluster autoscaling to let GKE add/remove nodes automatically

#### 📌 When to Use This Option

Choose node scaling when:

- Pods are stuck in Pending state due to insufficient resources
- Your node(s) frequently hit CPU/memory limits
- You need higher availability across more physical machines
- You want room for future Pod scaling
  
More nodes = more room for Pods, but does not create Pods by itself.

</details

---
