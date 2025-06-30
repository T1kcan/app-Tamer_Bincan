# Case Step II

A statefull application was choosen for the second phase of the Devops Case. Below are the step to run Github Action & Git Ops approach for application deployment. At the end of the document Cat vs Dogs Voting Application properties are listed.

## Step 1: Install Argo CD

### 1.1 Create `argocd` namespace and apply official manifest
01. Installing Argo CD 
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd wait deploy --all --for condition=Available --timeout 2m
kubectl get pods -n argocd
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath="{.data.password}" | base64 -d && echo
# -d1TcgILt62Hk6Z5
kubectl port-forward --address 0.0.0.0 svc/argocd-server -n argocd 8080:443
#or
kubectl edit svc -n argocd argocd-server:
#underneath spec:ports:http
#NodePort: 30080
#underneath spec:ports:https
#NodePort: 30443
#type: NodePort
```
02. Installing Argo CD CLI
Install Argo CD CLI, use the below commands to start with the installation
```bash
curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/download/v2.4.8/argocd-linux-amd64
chmod +x /usr/local/bin/argocd
```
Verify that CLI is installed
```bash
argocd
```
03. Through Personal Manifests:
```bash
kubectl create ns argocd
git clone https://github.com/T1kcan/argocd-example-apps.git
cd argocd-example-apps
kubectl apply -f install.yaml -n argocd
kubectl -n argocd wait deploy --all --for condition=Available --timeout 2m
kubectl get pods -n argocd
```
Wait until the pods are running and ready
Initial admin password is stored as a secret in argocd namespace:
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
# OkYifesK7KdIMGAM
```

### 1.2 Install Argo CD using Helm
```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm install argocd argo/argo-cd \
  --namespace argocd \
  --set server.service.type=NodePort

  
NAME: argocd
LAST DEPLOYED: Wed Jun 25 19:59:57 2025
NAMESPACE: argocd
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
In order to access the server UI you have the following options:

1. kubectl port-forward service/argocd-server -n argocd 8080:443

    and then open the browser on http://localhost:8080 and accept the certificate

2. enable ingress in the values file `server.ingress.enabled` and either
      - Add the annotation for ssl passthrough: https://argo-cd.readthedocs.io/en/stable/operator-manual/ingress/#option-1-ssl-passthrough
      - Set the `configs.params."server.insecure"` in the values file and terminate SSL at your ingress: https://argo-cd.readthedocs.io/en/stable/operator-manual/ingress/#option-2-multiple-ingress-objects-and-hosts


After reaching the UI the first time you can login with username: admin and the random password generated during the installation. You can find the password by running:

kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

(You should delete the initial secret afterwards as suggested by the Getting Started Guide: https://argo-cd.readthedocs.io/en/stable/getting_started/#4-login-using-the-cli)
```

### 1.3 Verify Pods
<!-- The Workflow Controller is responsible for running workflows:
```bash
kubectl -n argocd get deploy workflow-controller
``` -->
And the Argo Server provides a user interface and API:
```bash
kubectl -n argocd get deploy argo-server
```
All pods should be in `Running` state.
```bash
kubectl -n argocd wait deploy --all --for condition=Available --timeout 2m
kubectl get pods -n argocd
```
---

### 1.4 Access Web UI

01. Access Web UI

In this environment we exposed Argo CD server externally using node port.
```bash
controlplane:~$ kubectl get svc -n argocd argocd-server -o wide
NAME            TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE     SELECTOR
argocd-server   NodePort   10.111.162.189   <none>        80:32073/TCP,443:30164/TCP   4m49s   app.kubernetes.io/name=argocd-server
```
You can access Argo CD web ui using admin user and initial admin 'password'. 

---

02. Installing Argo CD CLI
Install Argo CD CLI, use the below commands to start with the installation
```bash
curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/download/v2.4.8/argocd-linux-amd64
chmod +x /usr/local/bin/argocd
```
Verify that CLI is installed
```bash
argocd
```

### Login using CLI and start using commands

In order to interact with Argo CD using CLI or web UI, Argo CD server need to be reached. In this environment we exposed Argo CD server externally using node port.
- Get the initial admin user password.
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
#wWsgk6eSY4XDYzai
```
- Login to Argo CD using CLI, use argocd login command.
```bash
example: argocd login argocdHost --grpc-web 
```
```bash
controlplane:~$ kubectl get nodes -o wide
NAME           STATUS   ROLES           AGE    VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
controlplane   Ready    control-plane   7d3h   v1.32.1   172.30.1.2    <none>        Ubuntu 24.04.1 LTS   6.8.0-51-generic   containerd://1.7.27
node01         Ready    <none>          7d3h   v1.32.1   172.30.2.2    <none>        Ubuntu 24.04.1 LTS   6.8.0-51-generic   containerd://1.7.27
controlplane:~$ kubectl get svc -n argocd argocd-server -o wide
NAME            TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE     SELECTOR
argocd-server   NodePort   10.109.112.76   <none>        80:32073/TCP,443:30164/TCP   8m15s   app.kubernetes.io/name=argocd-server
```
Since Argo CD server service is exposed using node port 32073, you can use the internal node IP and Argo CD server service port. 172.30.1.2:32073

Execute this command to login to Argo CD:
```bash
argocd login 172.30.1.2:32073 --grpc-web
#
Username: admin           
Password: 
'admin:login' logged in successfully
Context '172.30.1.2:32073' updated

```
Enter admin as user and enter the password that you got previously.

After successful login you can start using other commands such as listing apps or clusters.
```bash
argocd cluster list --grpc-web
```
----------

## Create applications
### Create ArgoCD Application Declaratively
- Check the application definition and create an Argo CD application declaratively using below Yaml format with specs and apply it using kubectl:

- application-dev.yaml
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: voting-dev
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/T1kcan/example-voting-app
    targetRevision: dev
    path: k8s/overlays/dev
  destination:
    server: https://kubernetes.default.svc
    namespace: dev
  syncPolicy:
    automated:
      selfHeal: true
      prune: true
```
-- applicatin-prod.yaml
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: voting-prod
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/T1kcan/example-voting-app
    targetRevision: main
    path: k8s/overlays/prod
  destination:
    server: https://kubernetes.default.svc
    namespace: prod
  syncPolicy:
    automated:
      selfHeal: true
      prune: true
```
Create the application:
```bash
kubectl apply -f application-dev.yaml
kubectl apply -f application-prod.yaml
```
Verify application is created
```bash
kubectl get application -n argocd
```

Open web UI and Access ArgoCD


## Creating Argo CD Application using Web UI

Create Github Action Workflow as follows:
- This pipeline will be triggered each code commit on dev branch and pull requests againts main and master branches. 
- Static Code Analysis and Container Image Building jobs will be running by parallel.
- Artifact container images will be tagged as GH build short SHA and will be pushed into Github Container Registry. 
- Each tag will be updated on environment's own kustomization yaml file.

- .github/workflows/.ci.yaml
```yaml
name: CI - Voting App Dev

on:
  push:
    branches:
      - dev
  pull_request:
    branches:
      - main
      - master

jobs:
  sast:
    name: Static Code Analysis (CodeQL)
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Initialize CodeQL
        uses: github/codeql-action/init@v3
        with:
          languages: javascript

      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@v3
                
  push-images:
    name: Build and Push Docker Images
    runs-on: ubuntu-latest
    env:
      COMMIT_SHA: ${{ github.sha }}
    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Login to GHCR
        uses: docker/login-action@v2
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build & Push Docker Images
        run: |
          docker build -t ghcr.io/t1kcan/vote:dev-${COMMIT_SHA} ./vote
          docker push ghcr.io/t1kcan/vote:dev-${COMMIT_SHA}

          docker build -t ghcr.io/t1kcan/result:dev-${COMMIT_SHA} ./result
          docker push ghcr.io/t1kcan/result:dev-${COMMIT_SHA}

      - name: Set ENV_NAME from branch
        run: |
          if [[ "${GITHUB_REF_NAME}" == "master" ]]; then
            echo "ENV_NAME=prod" >> $GITHUB_ENV
          else
            echo "ENV_NAME=dev" >> $GITHUB_ENV
          fi

      - name: Update kustomization.yaml image tags
        run: |
          sed -i "s|newTag:.*|newTag: ${ENV_NAME}-${COMMIT_SHA}|" k8s/overlays/${ENV_NAME}/kustomization.yaml

      - name: Commit updated kustomization.yaml
        run: |
          git config user.name "GitHub Actions"
          git config user.email "actions@github.com"
          git add k8s/overlays/${ENV_NAME}/kustomization.yaml
          git commit -m "Update image tag to ${ENV_NAME}-${COMMIT_SHA}"
          git push
```
Any code commit will trigger deployment of the application on dev namespace and pull request againts main/master brunch will deploy application on prod namespace.



# Example Voting App

A simple distributed application running across multiple Docker containers.

## Getting started

Download [Docker Desktop](https://www.docker.com/products/docker-desktop) for Mac or Windows. [Docker Compose](https://docs.docker.com/compose) will be automatically installed. On Linux, make sure you have the latest version of [Compose](https://docs.docker.com/compose/install/).

This solution uses Python, Node.js, .NET, with Redis for messaging and Postgres for storage.

Run in this directory to build and run the app:

```shell
docker compose up
```

The `vote` app will be running at [http://localhost:8080](http://localhost:8080), and the `results` will be at [http://localhost:8081](http://localhost:8081).

Alternately, if you want to run it on a [Docker Swarm](https://docs.docker.com/engine/swarm/), first make sure you have a swarm. If you don't, run:

```shell
docker swarm init
```

Once you have your swarm, in this directory run:

```shell
docker stack deploy --compose-file docker-stack.yml vote
```

## Run the app in Kubernetes

The folder k8s-specifications contains the YAML specifications of the Voting App's services.

Run the following command to create the deployments and services. Note it will create these resources in your current namespace (`default` if you haven't changed it.)

```shell
kubectl create -f k8s-specifications/
```

The `vote` web app is then available on port 31000 on each host of the cluster, the `result` web app is available on port 31001.

To remove them, run:

```shell
kubectl delete -f k8s-specifications/
```

## Architecture

![Architecture diagram](architecture.excalidraw.png)

* A front-end web app in [Python](/vote) which lets you vote between two options
* A [Redis](https://hub.docker.com/_/redis/) which collects new votes
* A [.NET](/worker/) worker which consumes votes and stores them in…
* A [Postgres](https://hub.docker.com/_/postgres/) database backed by a Docker volume
* A [Node.js](/result) web app which shows the results of the voting in real time

## Notes

The voting application only accepts one vote per client browser. It does not register additional votes if a vote has already been submitted from a client.

This isn't an example of a properly architected perfectly designed distributed app... it's just a simple
example of the various types of pieces and languages you might see (queues, persistent data, etc), and how to
deal with them in Docker at a basic level.
