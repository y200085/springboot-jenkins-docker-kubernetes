# Spring Boot + Git + Jenkins + Docker + Kubernetes (Windows Docker Desktop)

This is a starter CI/CD lab. It builds a Spring Boot REST API, creates a Docker image,
and deploys it to Kubernetes running in Docker Desktop.

## Architecture

GitHub -> Jenkins pipeline -> Maven tests/package -> Docker image -> Docker Desktop Kubernetes
                                                        |
                                                        +-> Kubernetes Deployment + Service

The app exposes:
- `GET /api/hello`
- `GET /actuator/health`

This starter keeps the example small so you can first prove the pipeline end-to-end.

## Prerequisites

- Windows 10/11
- Docker Desktop using **Linux containers**
- Git
- A GitHub repository for this project
- Docker Desktop Kubernetes enabled
- Jenkins already running in Docker

## 1. Enable Kubernetes in Docker Desktop

1. Open Docker Desktop.
2. Go to **Settings -> Kubernetes**.
3. Enable Kubernetes and apply/restart.
4. Wait until Kubernetes reports it is running.
5. In PowerShell, check:
   ```powershell
   kubectl config get-contexts
   kubectl config use-context docker-desktop
   kubectl get nodes
   ```
   If `kubectl` is not installed, install it with:
   ```powershell
   winget install Kubernetes.kubectl
   ```
   Open a new PowerShell window after installation.

## 2. Put this project in GitHub

Extract this ZIP, open PowerShell in the extracted project folder, then:
```powershell
git init
git add .
git commit -m "Initial Spring Boot Kubernetes pipeline"
git branch -M main
git remote add origin https://github.com/YOUR_GITHUB_USER/YOUR_REPO.git
git push -u origin main
```
Create the empty repository on GitHub first. Replace the remote URL with your own.

## 3. Build a Jenkins image with Maven, Docker CLI, and kubectl

This is needed because the existing Jenkins container may not have Maven, Docker CLI,
or kubectl installed. The Docker CLI talks to the Docker Desktop engine through the
mounted Docker socket; it does not run a second Docker daemon.

From the project folder, create/build the Jenkins image:
```powershell
docker build -f jenkins/Dockerfile -t local/jenkins-k8s:latest .
```

Stop the existing Jenkins container and recreate it using the same name/ports/volumes
you already use, but based on `local/jenkins-k8s:latest`. **Keep your existing Jenkins
home volume** so jobs and configuration are retained. Example only (adjust the container
name and volume to match your setup):
```powershell
docker stop jenkins
docker rm jenkins
docker run -d --name jenkins --restart unless-stopped `
  -p 8080:8080 -p 50000:50000 `
  -v jenkins_home:/var/jenkins_home `
  -v /var/run/docker.sock:/var/run/docker.sock `
  -v "$env:USERPROFILE/.kube:/var/jenkins_home/.kube:ro" `
  -e KUBECONFIG=/var/jenkins_home/.kube/config `
  local/jenkins-k8s:latest
```
If your current Jenkins container is named differently, use its name. If your Jenkins home
is a bind mount or a differently named volume, preserve that exact mount instead of
using `jenkins_home`. Do not delete the Jenkins home volume.

### Kubernetes kubeconfig note

Docker Desktop's kubeconfig is usually at `%USERPROFILE%\.kube\config`. Verify it exists:
```powershell
Test-Path "$env:USERPROFILE\.kube\config"
```
The mount above assumes that config works from inside a Linux container and that the
Kubernetes API address is reachable from the Jenkins container. Test from Jenkins:
```powershell
docker exec jenkins kubectl config get-contexts
docker exec jenkins kubectl get nodes
```
If this fails, inspect the server address in `%USERPROFILE%\.kube\config`. Windows-specific
certificate file paths or a host-only API endpoint may need a container-compatible kubeconfig.
Do not disable TLS verification as a permanent fix.

## 4. Configure Jenkins

1. Open `http://localhost:8080`.
2. Install the suggested plugins if prompted. Ensure **Pipeline**, **Git**, and **Credentials Binding**
   plugins are installed.
3. Select **New Item -> Pipeline**.
4. Name it `springboot-kubernetes-pipeline`.
5. Under **Pipeline**, choose **Pipeline script from SCM**.
6. SCM: **Git**. Repository URL: your GitHub repository URL.
7. Branch: `*/main`.
8. Script Path: `Jenkinsfile`.
9. Save.

For a public repository, HTTPS checkout can work without credentials. For a private repository,
add a GitHub token as a Jenkins credential and configure the repository credentials.

## 5. First run

Before running the job, check that Jenkins can access Docker and Kubernetes:
```powershell
docker exec jenkins docker version
docker exec jenkins kubectl get nodes
```
Then click **Build Now** in Jenkins.

The pipeline stages are:
1. Checkout
2. Verify tools and Kubernetes context
3. Unit tests and Maven package
4. Build Docker image
5. Deploy to Kubernetes
6. Wait for rollout and smoke test

## 6. Test the application

After a successful build:
```powershell
kubectl get deployments
kubectl get pods
kubectl get services
kubectl port-forward service/springboot-demo 8081:80
```
Keep `port-forward` running and open:
- http://localhost:8081/api/hello
- http://localhost:8081/actuator/health

## 7. Common troubleshooting

- **`docker: command not found`**: Jenkins is not running the custom image or the image build failed.
- **Docker socket permission/connection error**: verify `/var/run/docker.sock` is mounted and Docker Desktop is running.
- **`kubectl: command not found`**: use the custom Jenkins image from this project.
- **`Unable to connect to the server`**: kubeconfig API address/certificates are not usable from the Jenkins container.
- **ImagePullBackOff**: this local image must be built into the same Docker engine used by Docker Desktop Kubernetes. If your Kubernetes setup uses a separate image store, load/push the image appropriately.
- **`rollout status` timeout**: inspect `kubectl describe pod`, `kubectl logs deployment/springboot-demo`, and `kubectl get events --sort-by=.lastTimestamp`.

## Security note

Mounting the Docker socket gives Jenkins powerful control over the Docker host. This is acceptable
for a personal learning lab, but do not use this exact arrangement for an untrusted multi-user
Jenkins server or production environment.
