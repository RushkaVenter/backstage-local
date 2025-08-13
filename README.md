# backstage-local
This repo guides a developer on running a local instance of backstage

# Prerequisites
* Docker Desktop
* Docker Hub Account
* Have node and yarn installed
* GitHub Authentication App ID and Secret
* Azure App Registration ID and secret

# Run backstage
Install the GitHub plugin from the backstage folder:
```
yarn --cwd packages/backend add @backstage/plugin-auth-backend-module-github-provider
```

From the backstage folder, you can run backstage with:
```
yarn start
```
Currently this does not connect to a db

# Run backstage on Docker
[See: Getting started prerequisites section](https://backstage.io/docs/getting-started/)

## Setup network for Backstage
```
docker network create backstage
```

## Create Database for backstage
```
docker run -d --name psql -e POSTGRES_PASSWORD=backstage -e POSTGRES_DB=backstage -e POSTGRES_USER=backstage -e PGDATA=/var/lib/postgresql/data/pgdata -v /tmp/psql:/var/lib/postgresql/data --network backstage postgres:16
```

Ensure that your container is up and running with docker ps.
Check that the logs have "database system is ready to accept connections" with the below
```
docker logs -f <CONTAINER ID>
```

## Create a backstage docker file
The backstage folder will have a sample backstage app, but if you want the latest version, you can follow these steps:
* From within the project folder, run the following ```npx @backstage/create-app@latest```
* Ok to proceed? (y) -> press Enter
* Just press enter when prompted for a name
* Copy the app-config.yaml, app-config.local.yaml and app-config.production.yaml files in this repo
* Copy the docker file from the backstage folder in this repo
* Copy the catalog folder from this repo

## Build the docker image
docker build -t backstage_production .

# Run backstage on Kubernetes
## Setup Local Kubernetes Cluster
We will be installing a local kind cluster on a Windows machine.
[Quick Start Guide](https://kind.sigs.k8s.io/docs/user/quick-start/)

### Install kind
```
curl.exe -Lo kind-windows-amd64.exe https://kind.sigs.k8s.io/dl/v0.29.0/kind-windows-amd64
Move-Item .\kind-windows-amd64.exe c:\some-dir-in-your-PATH\kind.exe
```
Alternatively
```
winget install Kubernetes.kind
```

### Create Kind Cluster
```
kind create cluster --config kind-config.yaml
```

### Setup ingress-nginx
ingress-nginx is an Ingress controller for Kubernetes using NGINX as a reverse proxy and load balancer.
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
```

## Install kubectl
[kubectl binaries](https://kubernetes.io/releases/download/#binaries)

# Clean Up Steps if needed
## Deleting previous kubernetes
```
kind delete cluster --name kind
```