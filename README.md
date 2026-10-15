# End-to-End CI/CD with Jenkins, Docker, Kubernetes and Argo CD

This project demonstrates an automated delivery workflow for a Java Spring Boot application. Jenkins builds the application, runs Maven and SonarQube analysis, creates a Docker image, publishes it to Docker Hub, and updates the Kubernetes deployment manifest in GitHub. Argo CD watches that Git repository and reconciles the Kubernetes cluster with the manifest stored in Git.

The goal is to connect **Continuous Integration (CI)** with **GitOps-based Continuous Delivery (CD)**, while keeping application code, image versions, and deployment configuration traceable.

![alt text](228301952-abc02ca2-9942-4a67-8293-f76647b6f9d8-1.png)

## Architecture

```text
Application source
      |
      v
    GitHub
      |
      v
    Jenkins
      |
      +--> Checkout / workspace preparation
      +--> Maven build and package
      +--> SonarQube static analysis
      +--> Build Docker image with Jenkins build number
      +--> Push image to Docker Hub
      +--> Update image tag in Kubernetes manifest
      +--> Commit and push manifest change to GitHub
                                      |
                                      v
                           GitHub deployment manifests
                                      |
                                      v
                                   Argo CD
                         (watches repository and path)
                                      |
                           detects desired-state change
                                      |
                                      v
                                 Kubernetes
                                      |
                            Spring Boot Deployment
                                      |
                                Service / NodePort
                                      |
                                      v
                              Spring Boot application
```

### Responsibilities of each component

| Component | Role in this project |
|---|---|
| GitHub | Stores the application source and Kubernetes manifests. |
| Jenkins | Automates the build, analysis, image publishing, and manifest update. |
| Maven | Builds and packages the Spring Boot application. |
| SonarQube | Performs static code analysis. |
| Docker | Packages the application into a container image. |
| Docker Hub | Stores versioned application images. |
| Kubernetes | Runs the application Pods and exposes them through a Service. |
| Argo CD | Implements GitOps delivery by comparing the desired state in Git with the live cluster state. |
| Minikube | Provides the local Kubernetes environment used for deployment and testing. |

## Repository layout

```text
jenkins-end-to-end-pipeline-for-java-based-application/
├── spring-boot-app/
│   ├── src/
│   ├── Dockerfile
│   ├── pom.xml
│   └── JenkinsFile
└── spring-boot-app-manifests/
    ├── deployment.yml
    └── service.yml
```

- `spring-boot-app/` contains the Spring Boot source, Maven configuration, Dockerfile, and Jenkins pipeline.
- `spring-boot-app-manifests/deployment.yml` defines the application Deployment and container image.
- `spring-boot-app-manifests/service.yml` exposes the application through a Kubernetes Service.

The application Deployment is configured for **two replicas** and container port **8080**. The Service uses port **80** and targets container port **8080**. In the local Minikube setup, the Spring Boot Service was exposed through NodePort **30140**.

## CI pipeline

The Jenkins pipeline connects the build and delivery steps:

1. **Workspace and source preparation** — the pipeline prepares the workspace. The checked-in pipeline has used a lightweight `echo passed`/directory-listing step while the explicit Git checkout step was commented out in one iteration; the Jenkins job or agent therefore needs to provide the expected source workspace.
2. **Build and package** — Maven runs `mvn clean package`.
3. **Static code analysis** — Maven invokes SonarQube analysis with the configured server URL and token.
4. **Build the container image** — Docker builds the application image and tags it with the Jenkins build number.
5. **Publish the image** — Jenkins pushes the tagged image to Docker Hub.
6. **Update deployment configuration** — the pipeline replaces the image placeholder/tag in `spring-boot-app-manifests/deployment.yml`.
7. **Commit and push the manifest** — Jenkins commits the changed manifest and pushes it to the `main` branch.
8. **Reconcile through Argo CD** — Argo CD watches the configured Git repository, branch, and manifest directory. With automated sync enabled, it applies a changed desired state to Kubernetes.
9. **Verify the release** — the Deployment, Pods, Service, and application endpoint are checked after reconciliation.

A build-number tag makes each published image identifiable. For example, the deployment was updated to `samarthchinde/spring-boot-app:9` during testing. The manifest in Git and the live Deployment were both checked for that tag.

> **CI/CD distinction:** Jenkins publishes the image and changes the desired deployment configuration in Git. Argo CD does not build the image; it deploys the state represented by the Git manifests.

## Configuration and implementation notes

The pipeline and cluster rely on configuration held outside the application source:

- Jenkins credentials for SonarQube, Docker Hub, and GitHub access.
- A reachable SonarQube server and the configured `SONAR_URL`.
- Docker CLI access to the Docker daemon from the Jenkins agent.
- A Docker Hub repository that accepts pushes from Jenkins.
- An Argo CD Application pointing to the correct Git repository, `main` branch, and `spring-boot-app-manifests` path.
- Kubernetes Deployment and Service manifests that match the image name, container port, labels, and selectors.

Credentials should be stored in Jenkins Credentials and referenced by credential ID. Tokens, passwords, and decoded Kubernetes Secrets are not committed to Git.

## Kubernetes and Argo CD operations

These are the principal commands used to inspect the local deployment and troubleshoot access.

### Inspect cluster resources

```bash
kubectl get pods -A
kubectl get deployments,services
kubectl get svc -n default
kubectl get endpoints -n default
```

`kubectl get endpoints` helps determine whether a Service has ready Pod endpoints behind it.

### Inspect Argo CD Service exposure

```bash
kubectl get svc example-argocd-server -o jsonpath='{.spec.type}{"\n"}'
kubectl get svc example-argocd-server -o yaml
kubectl get svc -n default
```

The Argo CD server Service was inspected when its browser endpoint was not reachable. Its Service type and port mapping need to match the chosen access method. A NodePort change was also investigated using a Kubernetes Service patch; the exact Service name and namespace must match the installed Argo CD resources.

### Access the services in Minikube

```bash
minikube ip
minikube service example-argocd-server -n default
minikube service spring-boot-app-service -n default
```

`minikube service` can open or report a host URL for a Service. The generated local URL can vary between runs and environments. The Spring Boot Service used NodePort `30140`; the Argo CD server had its own Service and port configuration.

### Inspect application rollout and failures

```bash
kubectl get pods -n default -o wide
kubectl describe pod <pod-name> -n default
kubectl logs <pod-name> -n default
kubectl describe svc spring-boot-app-service -n default
kubectl get endpoints spring-boot-app-service -n default
```

These commands expose scheduling events, image-pull errors, container startup failures, readiness issues, Service selectors, and missing endpoints.

### Edit Kubernetes resources on CachyOS

The local workstation ran CachyOS Linux with the Fish shell. The Kubernetes editor was configured to use Vim:

```fish
command -v vim
command -v vi
set -gx KUBE_EDITOR vim
```

To persist the editor setting in Fish:

```fish
echo 'set -gx KUBE_EDITOR vim' >> ~/.config/fish/config.fish
```

This allows `kubectl edit` to open resources in Vim when the editor is available.

## Troubleshooting record

The following issues were encountered during implementation and investigated with the commands below.

### 1. Jenkins pipeline and Git operations

**Observed issue:** The pipeline could not reliably find the Jenkinsfile or access the Git repository in earlier runs. One checkout attempt returned an SSH public-key error (`Permission denied (publickey)`), and the explicit checkout step was commented out during a later pipeline iteration.

**Checks and commands:**

```bash
ls -ltr
git status
git remote -v
git branch
```

The Jenkins job's script path was set in relation to `spring-boot-app/JenkinsFile`. Git access depends on the repository URL and the configured Jenkins credentials; SSH access additionally requires a registered public key and the matching private key available to Jenkins.

### 2. Argo CD did not automatically sync after a Jenkins build

**Observed issue:** Jenkins ran again and pushed a new Git commit, but Argo CD did not immediately show the expected automatic synchronization. During the investigation, the image tag was checked in both places: the Git manifest contained `samarthchinde/spring-boot-app:9`, and the live Kubernetes Deployment also showed `samarthchinde/spring-boot-app:9`.

**What this established:** Matching image tags alone do not prove that Argo CD detected a new desired-state change. If Git already contains the same manifest value, a new Jenkins build may not create a meaningful manifest diff. The Argo CD Application source repository, target revision, manifest path, automated-sync setting, and application sync status are also relevant.

**Checks:**

```bash
git log -n 5 --oneline
grep -n "image:" spring-boot-app-manifests/deployment.yml
kubectl get deployment -n default -o wide
kubectl get pods -n default
```

The Argo CD Application was also inspected in the UI to compare its repository, revision, path, sync status, and health status with the Git repository and live cluster.

### 3. Argo CD browser access and Service type

**Observed issue:** Argo CD was installed in the local Kubernetes environment, but its browser endpoint was not consistently accessible. The server Service configuration was checked, and NodePort exposure and Minikube service access were investigated.

**Commands used:**

```bash
kubectl get svc example-argocd-server -o jsonpath='{.spec.type}{"\n"}'
kubectl get svc example-argocd-server -o yaml
kubectl get svc -n default
kubectl get endpoints -n default
minikube service example-argocd-server -n default
```

Service access depends on the actual Service type, its ports, the health of its backing Pods, and the local cluster's networking. A URL printed by Minikube is environment-specific rather than a permanent public endpoint.

### 4. Spring Boot Service access

**Observed issue:** Argo CD was not the only endpoint that needed validation; the deployed Spring Boot application also had to be reachable outside the Pod.

**Commands used:**

```bash
kubectl get svc -n default
kubectl describe svc spring-boot-app-service -n default
kubectl get endpoints spring-boot-app-service -n default
minikube ip
minikube service spring-boot-app-service -n default
```

The Service mapped port `80` to target port `8080` and used NodePort `30140` in the local setup. Verifying endpoints alongside the Service helped distinguish a networking issue from a Deployment or Pod readiness issue.

### 5. Pods in Pending or Terminating states

**Observed issue:** During the local cluster work, Pods were observed in `Pending` or `Terminating` states. These states can prevent both the Argo CD interface and application from being available even when the Service definition exists.

**Commands used:**

```bash
kubectl get pods -A -o wide
kubectl describe pod <pod-name> -n default
kubectl get events -n default --sort-by=.lastTimestamp
kubectl get nodes
```

The relevant evidence is in Pod events, node status, scheduling messages, image-pull results, and container logs. A Service URL alone does not establish that the workload is healthy.

### 6. SonarQube binary download returned HTTP 403

**Observed issue:** While setting up SonarQube on an Ubuntu server, a direct `wget` request to a SonarSource binary download URL returned `403 Forbidden`, despite the connection being established.

**Troubleshooting context:** This was a download-request rejection rather than proof of a general network outage. The download URL or distribution method needed to be checked before continuing installation.

## Verification checklist

A successful end-to-end run is represented by the following evidence:

- Jenkins completes the Maven build and SonarQube analysis stages.
- Docker builds and pushes the expected versioned image.
- GitHub records the updated image tag in `spring-boot-app-manifests/deployment.yml`.
- Argo CD tracks the correct repository, branch, and manifest path.
- The Argo CD Application reaches `Synced` and `Healthy` when the cluster and workload are healthy.
- Kubernetes has ready application Pods and Service endpoints.
- The Spring Boot application is reachable through the configured Minikube Service route.

## Project evidence

Screenshots and a short demonstration recording accompany this repository to show the pipeline execution, SonarQube analysis, Docker Hub image, Git manifest update, Argo CD application status, and running Kubernetes workload.

## Skills demonstrated

**Jenkins Pipeline · Maven · SonarQube · Docker · Docker Hub · GitHub · Kubernetes · Minikube · Argo CD · GitOps · Linux · Fish shell · CI/CD troubleshooting**

## Repository

[GitHub — samarthchinde/jenkins-end-to-end-pipeline-for-java-based-application](https://github.com/samarthchinde/jenkins-end-to-end-pipeline-for-java-based-application)
