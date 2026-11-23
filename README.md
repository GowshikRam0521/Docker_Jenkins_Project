# Kanban Dashboard -- Docker, Jenkins & AWS CI/CD

A containerized Kanban Task Manager deployed on **AWS EC2** with
**Docker** and an automated **Jenkins CI/CD pipeline**. The project
demonstrates building a Docker image, publishing versioned images to
Docker Hub, and deploying the application to an EC2 instance.

## Project Overview

The application is a Kanban Task Manager with three workflow columns:

-   **To Do**
-   **In Progress**
-   **Done**

Users can create tasks with details such as a title, description,
status, tags, priority, and optional deadline, then organize tasks
across the Kanban workflow. The application is based on the [Kanban
Dashboard source
project](https://github.com/Vennilavanguvi/kanban-dashboard).

The deployment project focuses on automating the application delivery
lifecycle and applying containerization and deployment best practices.

## Architecture

``` text
Developer
   |
   | git push
   v
GitHub Repository
   |
   | Webhook
   v
Jenkins on AWS EC2
   |
   | Checkout source
   | Build and tag Docker image
   | Push image
   v
Docker Hub
   |
   | Pull versioned image
   v
Docker Container on AWS EC2
   |
   | Health check and HTTP validation
   v
Live Kanban Dashboard
```

## Technology Stack

  Technology         Purpose
  ------------------ ------------------------------------------------
  Kanban Dashboard   Web application
  Docker             Application containerization
  Docker Hub         Container image registry
  Jenkins            CI/CD automation
  AWS EC2            Jenkins host and application deployment server
  GitHub             Source control and webhook trigger

## Repository Structure

The deployment repository is intended to contain the application
deployment configuration, including files such as:

``` text
.
├── Dockerfile
├── .dockerignore
├── Jenkinsfile
├── nginx.conf              # If used by the runtime image
└── README.md
```

The exact application source tree and supporting configuration may vary
with the checked-out source revision.

## CI/CD Workflow

The Jenkins pipeline is designed to automate these stages:

1.  **Checkout** -- retrieve the source code from GitHub.
2.  **Build** -- build a Docker image from the Dockerfile.
3.  **Tag** -- assign a version using the Jenkins build number or Git
    commit SHA. The deployment does not rely on the `latest` tag alone.
4.  **Publish** -- authenticate to Docker Hub using Jenkins-managed
    credentials and push the tagged image.
5.  **Deploy** -- connect to the EC2 deployment host, pull the image,
    and start/update the application container.
6.  **Validate** -- verify the container state, Docker health status,
    and application HTTP response.
7.  **Result / recovery** -- report pipeline success or failure; the
    deployment workflow specifies rollback on a failed deployment.

A GitHub webhook can trigger the Jenkins pipeline when changes are
pushed, avoiding a manual build trigger.

## Deployment Details

  Item                          Value
  ----------------------------- -------------------------------
  Cloud platform                AWS
  Compute                       EC2
  Container runtime             Docker
  CI/CD server                  Jenkins
  Jenkins UI port               `8080`
  Application host port         `8081`
  Docker Hub image repository   `gowshikram/kanban-dashboard`
  Public application URL        http://your_ec2_public_ip:8081/

The application is accessed through host port `8081`. The container's
internal listening port and the exact port mapping should be checked
against the current Docker run/deployment configuration.

## Run Locally

Clone the project repository:

``` bash
git clone https://github.com/GowshikRam0521/Docker_Jenkins_Project.git
cd Docker_Jenkins_Project
```

Install and run the application using the package scripts in the
checked-out source:

``` bash
npm install
npm run dev
```

For this project, the local Vite development server was shown running at
`http://localhost:5173`. Use the URL printed by the development server
if it differs.

## Build and Run with Docker

Build the image from the directory containing the Dockerfile:

``` bash
docker build -t kanban-dashboard:local .
```

Run it:

``` bash
docker run -d \
  --name kanban-dashboard \
  -p 8081:8080 \
  kanban-dashboard:local
```

> The example mapping above assumes the runtime image listens on
> container port `8080` (for example, an Nginx-based static frontend
> image). If your Dockerfile/runtime listens on another port, change the
> right-hand side of the mapping to that port.

Check the container:

``` bash
docker ps
docker logs kanban-dashboard
docker inspect kanban-dashboard
```

Check its health status:

``` bash
docker inspect --format='{{.State.Health.Status}}' kanban-dashboard
```

Test the application:

``` bash
curl -I http://localhost:8081/
```

## Docker Image Registry

The published image repository is:

[gowshikram/kanban-dashboard on Docker
Hub](https://hub.docker.com/repository/docker/gowshikram/kanban-dashboard/general)

Example commands (replace the tag with the version produced by your
build):

``` bash
docker login
docker build -t gowshikram/kanban-dashboard:<tag> .
docker push gowshikram/kanban-dashboard:<tag>
```

To verify a published version on another machine or on EC2:

``` bash
docker pull gowshikram/kanban-dashboard:<tag>
```

Use immutable build-number or Git-SHA tags to identify releases and make
rollbacks reproducible.

## Jenkins Configuration

The Jenkins setup uses the following integrations:

-   Git
-   Docker Pipeline
-   Credentials Binding

Configure the pipeline job to retrieve the `Jenkinsfile` from the GitHub
repository. Store Docker Hub and deployment/SSH credentials in **Jenkins
Credentials** and reference credential IDs from the pipeline. Do not
place passwords, access tokens, private keys, or other secrets directly
in source files or image layers.

Configure a GitHub webhook to notify Jenkins when a commit is pushed.
Ensure the Jenkins endpoint is reachable by GitHub and that the webhook
trigger is enabled for the job.

## AWS EC2 and Network Access

The EC2 instance hosts Jenkins and the deployed application container.

Configure the EC2 security group to allow only the access required by
your environment. Typical ports for this setup are:

  Port     Purpose              Recommended access
  -------- -------------------- -----------------------------------------
  `22`     SSH administration   Restrict to trusted IP addresses
  `8080`   Jenkins UI           Restrict to trusted IP addresses or VPN
  `8081`   Kanban application   Allow intended application users

Do not expose administrative services broadly to the public internet.
For production use, consider HTTPS through a reverse proxy or load
balancer and secure Jenkins behind appropriate network controls.

## Container Security and Reliability

The project requirements and implementation approach include:

-   Use a minimal base image such as Alpine or slim where appropriate.
-   Use a multi-stage build so the runtime image contains only the
    required production artifacts.
-   Run the application as a non-root user where supported.
-   Add a `.dockerignore` file to exclude items such as `node_modules`,
    `.git`, and local environment files.
-   Keep secrets out of Dockerfiles, source control, build arguments,
    and image layers.
-   Add a Docker `HEALTHCHECK` and validate application availability
    rather than relying only on the container's running state.
-   Apply CPU and memory limits appropriate to the EC2 instance and
    workload.

Example resource limits (adjust to the host capacity and application
needs):

``` bash
docker update \
  --memory=512m \
  --memory-reservation=256m \
  --cpus=0.5 \
  kanban-dashboard
```

Docker may reject an update if the requested limit conflicts with limits
already configured on the container. Inspect the current settings before
changing them.

## Troubleshooting and Validation

Useful commands:

``` bash
# Running containers
docker ps

# Container logs
docker logs kanban-dashboard

# Detailed container configuration and health
docker inspect kanban-dashboard

# Image list
docker images

# HTTP response
curl -I http://localhost:8081/

# Jenkins service status (Ubuntu EC2)
sudo systemctl status jenkins

# Docker service status (Ubuntu EC2)
sudo systemctl status docker
```

A deployment should be considered successful only after the new
container is running, its health check is passing, and the application
endpoint responds as expected. Review the Jenkins console output and
Docker logs when a pipeline stage fails.

## Project Goals Demonstrated

-   Containerize a web application with Docker.
-   Build and publish versioned Docker images.
-   Provision AWS EC2 as a deployment environment.
-   Configure Jenkins to build and deploy from a Jenkinsfile.
-   Automate deployments through a GitHub webhook.
-   Validate deployments using Docker health checks and HTTP requests.
-   Apply container security, resource limits, and operational
    troubleshooting practices.


------------------------------------------------------------------------

*This README documents the project details and deployment evidence
provided with the project. Confirm the current EC2 public IP, Docker
container port, image tag, and Jenkins job settings before reusing the
deployment commands in another environment.*
