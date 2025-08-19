# backstage-local
This repo guides a developer on running a local instance of backstage

# Prerequisites
* Docker Desktop
* Docker Hub Account
* Have node and yarn installed
* GitHub Authentication App ID and Secret
* Azure App Registration ID and secret

# Useful links
* https://github.com/ricardoandre97/backstage

# Run backstage
If you are not using the folder in the repo, install backstage:
```
npx @backstage/create-app@latest
```
This creates a blank backstage with the guest provider.
We will add the [GitHub Auth Provider](https://backstage.io/docs/auth/github/provider/)
Follow the instruction on creating a GitHub app.

Install the GitHub plugin from the backstage folder:
```
yarn --cwd packages/backend add @backstage/plugin-auth-backend-module-github-provider
```

Add the following files under a new path: catalog/entities (users.yaml and groups.yaml)
```
apiVersion: backstage.io/v1alpha1
kind: User
metadata:
  name: RushkaVenter # your github username
spec:
  profile:
    # Intentional no displayName for testing
    email: rushka.venter@entelect.co.za
    picture: https://api.dicebear.com/7.x/avataaars/svg?seed=Luna&backgroundColor=transparent
  memberOf: [development]
```
```
apiVersion: backstage.io/v1alpha1
kind: Group
metadata:
  name: development
  description: Development Team
spec:
  type: team
  profile:
    # Intentional no displayName for testing
    email: dev@example.com
    picture: https://api.dicebear.com/7.x/identicon/svg?seed=Fluffy&backgroundType=solid,gradientLinear&backgroundColor=ffd5dc,b6e3f4
  children: []
```

In the app-config.local.yaml file, add the auth configuration:
```
auth:
  environment: development
  providers:
    github:
      development:
        clientId: ${AUTH_GITHUB_CLIENT_ID}
        clientSecret: ${AUTH_GITHUB_CLIENT_SECRET}
        ## uncomment if using GitHub Enterprise
        # enterpriseInstanceUrl: ${AUTH_GITHUB_ENTERPRISE_INSTANCE_URL}
        ## uncomment to set lifespan of user session
        # sessionDuration: { hours: 24 } # supports `ms` library format (e.g. '24h', '2 days'), ISO duration, "human duration" as used in code
        signIn:
          resolvers:
            # See https://backstage.io/docs/auth/github/provider#resolvers for more resolvers
            - resolver: usernameMatchingUserEntityName
catalog:
  rules:
    - allow: [Component, System, API, Resource, Location]
  locations:
    # Local example data, file locations are relative to the backend process, typically `packages/backend`
    - type: file
      target: ../../catalog/entities/users.yaml
      rules:
        - allow: [User]

    - type: file
      target: ../../catalog/entities/groups.yaml
      rules:
        - allow: [Group]
```

Modify the file in packages/backend/src/index.ts with the following line and remove the guest provider.
```
backend.add(import('@backstage/plugin-auth-backend-module-github-provider'));
```

For the front end, ensure these changes on the packages/app/src/App.tsx file
```
import { githubAuthApiRef } from '@backstage/core-plugin-api';
import { SignInPage } from '@backstage/core-components';

const app = createApp({
  components: {
    SignInPage: props => (
      <SignInPage
        {...props}
        auto
        provider={{
          id: 'github-auth-provider',
          title: 'GitHub',
          message: 'Sign in using GitHub',
          apiRef: githubAuthApiRef,
        }}
      />
    ),
  },
  // ..
});
```

From the backstage folder, you can run backstage with:
```
yarn start
```
Currently this does not connect to a db

# Run backstage on Docker
[See: Getting started prerequisites section](https://backstage.io/docs/getting-started/)

## Add Azure Authentication and Catalogue
To install the plugins:
```
yarn workspace backend add @backstage/plugin-catalog-backend-module-msgraph
yarn workspace backend add @backstage/plugin-auth-backend-module-microsoft-provider
```

Add the plugins to the backend:
```
backend.add(import('@backstage/plugin-catalog-backend-module-msgraph'));
backend.add(import('@backstage/plugin-auth-backend-module-microsoft-provider'));
```

Add the auth provider in auth section in the app-config.production.yaml:
```
microsoft:
  development:
    clientId: ${AUTH_MICROSOFT_CLIENT_ID}
    clientSecret: ${AUTH_MICROSOFT_CLIENT_SECRET}
    tenantId: ${AUTH_MICROSOFT_TENANT_ID}
    signIn:
      resolvers: #resolvers must be enabled, or else throws errors that Microsoft SSO is not configured correctly "NotFoundError"
          - resolver: emailMatchingUserEntityProfileEmail
          - resolver: emailLocalPartMatchingUserEntityName
          - resolver: emailMatchingUserEntityAnnotation
          - resolver: userIdMatchingUserEntityAnnotation
```

Add catalog section for Azure in the app-config.production.yaml
```
catalog:
  rules:
    - allow: [Component, System, API, Resource, Location, User, Group]

  providers:
    microsoftGraphOrg: 
      providerId:
        target: https://graph.microsoft.com/v1.0
        authority: https://login.microsoftonline.com
        tenantId: ${AUTH_MICROSOFT_TENANT_ID}
        clientId: ${AUTH_MICROSOFT_CLIENT_ID}
        clientSecret: ${AUTH_MICROSOFT_CLIENT_SECRET}
        schedule:
          frequency: PT3M
          timeout: PT15M
```

Add the following Application permissions to your application registration:
* User.Read.All
* Group.Read.All
* Directory.Read.All

Update the signin page section:
```
SignInPage: props => <SignInPage {...props} auto providers={[{
          id: 'github-auth-provider',
          title: 'GitHub',
          message: 'Sign in using GitHub',
          apiRef: githubAuthApiRef,
        }} />,
        },
        {
          id: 'microsoft-auth-provider',
          title: 'Microsoft',
          message: 'Sign in using Microsoft',
          apiRef: microsoftAuthApiRef,
        }]} />,
  },
```

## Setup network for Backstage
```
docker network create backstage
```

## Create Database for backstage
```
docker run -d --name psql -e POSTGRES_PASSWORD=backstage -e POSTGRES_DB=backstage -e POSTGRES_USER=backstage -e PGDATA=/var/lib/postgresql/data/pgdata -v /tmp/psql:/var/lib/postgresql/data -p 5432:5432 --network backstage postgres:16
```

Access the db with pgadmin
```
docker run -d --name pgadmin -e PGADMIN_DEFAULT_EMAIL=admin@example.com -e PGADMIN_DEFAULT_PASSWORD=admin -p 8080:80 --network backstage dpage/pgadmin4
```

Ensure that your container is up and running with docker ps.
Check that the logs have "database system is ready to accept connections" with the below
```
docker logs -f <CONTAINER ID>
```

## Dockerfile
Add a docker file to the backstage folder
```
FROM node:20-bookworm-slim

RUN corepack enable && corepack prepare yarn@4.4.1 --activate

# Install isolate-vm dependencies, these are needed by the @backstage/plugin-scaffolder-backend.
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt-get update && \
    apt-get install -y --no-install-recommends python3 python3-pip python3-venv g++ build-essential

ENV PYTHON=/usr/bin/python3
ENV VIRTUAL_ENV=/opt/venv
RUN python3 -m venv $VIRTUAL_ENV
ENV PATH="$VIRTUAL_ENV/bin:$PATH"
RUN pip3 install mkdocs-techdocs-core

RUN mkdir -p /home/node/.cache/node/corepack && chown -R node:node /home/node/.cache

# From here on we use the least-privileged `node` user to run the backend.
USER node

WORKDIR /app

# Copy files needed by Yarn
COPY --chown=node:node .yarn ./.yarn
COPY --chown=node:node .yarnrc.yml ./
COPY --chown=node:node backstage.json ./

# This switches many Node.js dependencies to production mode.
ENV NODE_ENV=production

# This disables node snapshot for Node 20 to work with the Scaffolder
ENV NODE_OPTIONS="--no-node-snapshot"

# Copy repo skeleton first, to avoid unnecessary docker cache invalidation.
# The skeleton contains the package.json of each package in the monorepo,
# and along with yarn.lock and the root package.json, that's enough to run yarn install.
COPY --chown=node:node yarn.lock package.json packages/backend/dist/skeleton.tar.gz ./
RUN tar xzf skeleton.tar.gz && rm skeleton.tar.gz

RUN --mount=type=cache,target=/home/node/.cache/yarn,sharing=locked,uid=1000,gid=1000 \
    yarn workspaces focus --all --production && rm -rf "$(yarn cache clean)"

# Then copy the rest of the backend bundle, along with any other files we might want.
COPY --chown=node:node packages/backend/dist/bundle.tar.gz app-config*.yaml ./
RUN tar xzf bundle.tar.gz && rm bundle.tar.gz

COPY --chown=node:node ./catalog .

CMD ["node", "packages/backend", "--config", "app-config.yaml", "--config", "app-config.production.yaml"]
```

## Build the docker image
From the backstage folder build the distributables and then build the image
```
yarn build:backend 
docker build -t backstage_production .
```

## Run the docker image
docker run -d --name backstage_production -e POSTGRES_HOST=psql -e POSTGRES_PORT=5432 -e POSTGRES_USER=backstage -e POSTGRES_PASSWORD=backstage -e AUTH_GITHUB_CLIENT_ID=githubclientid -e AUTH_GITHUB_CLIENT_SECRET=githubclientsecret -e AUTH_MICROSOFT_CLIENT_ID=AppRegId -e AUTH_MICROSOFT_CLIENT_SECRET=AppRegSecret -e AUTH_MICROSOFT_TENANT_ID=AppRegTenant -p 3000:3000 -p 7007:7007 --network backstage backstage_production

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

## Postgres DB on kubernetes
Create the a helm values file:
```
global:
  postgresql:
    auth:
      postgresPassword: "backstage"
      username: "backstage"
      password: "backstage"
      database: "backstage"
primary:
  name: primary
  resources:
    requests:
      memory: 256Mi
      cpu: 100m
  persistence:
    enabled: true
    size: 8Gi
```

Install the chart repo:
```
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

Install the chart
```
helm install psql bitnami/postgresql --version 16.7.26 -n backstage --create-namespace -f values-postgres.yaml
```

You can check that this worked to deploy the backstage image by:
```
kubectl exec -ti psql-postgresql-0 -n backstage -- sh
psql -U backstage
```

You should see "backstage=>" on success

## Backstage on Kubernetes
Build a new docker image with the following commands and push the image to Docker Hub
```
docker build -t backstage:v1 .
docker tag backstage:v1 <account name>/backstage:v1
docker login
docker push <account name>/backstage:v1
```

Create the helm chart for backstage and apply it to kubernetes
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backstage
  labels:
    app: backstage
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backstage
  template:
    metadata:
      labels:
        app: backstage
    spec:
      containers:
      - name: backstage
        image: rushkaventervector/backstage:v1
        ports:
        - containerPort: 7007
        env:
          - name: APP_CONFIG_app_baseUrl
            value: https://backstage.test.com
          - name: APP_CONFIG_backend_baseUrl
            value: https://backstage.test.com
          - name: POSTGRES_HOST
            value: psql-postgresql
          - name: POSTGRES_USER
            value: backstage
          - name: POSTGRES_PASSWORD
            value: backstage
          - name: GITHUB_TOKEN
            value: some_secret_value(DO NOT PUSH SECRETS TO GITHUB)
          - name: AUTH_GITHUB_CLIENT_ID
            value: some_secret_value(DO NOT PUSH SECRETS TO GITHUB)
          - name: AUTH_GITHUB_CLIENT_SECRET
            value: some_secret_value(DO NOT PUSH SECRETS TO GITHUB)
          - name: AUTH_MICROSOFT_CLIENT_ID
            value: some_secret_value(DO NOT PUSH SECRETS TO GITHUB)
          - name: AUTH_MICROSOFT_CLIENT_SECRET
            value: some_secret_value(DO NOT PUSH SECRETS TO GITHUB)
          - name: AUTH_MICROSOFT_TENANT_ID
            value: some_secret_value(DO NOT PUSH SECRETS TO GITHUB) 

---
apiVersion: v1
kind: Service
metadata:
  name: backstage
spec:
  selector:
    app: backstage
  ports:
    - protocol: TCP
      port: 7007
      targetPort: 7007
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: backstage
spec:
  rules:
  - host: "backstage.test.com"
    http:
        paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: backstage
              port:
                number: 7007
```

```
kubectl apply -f backstage-k8s.yaml -n backstage
```

You would need to add the following to your hosts file:
```
127.0.0.1 backstage.test.com
```

You will need to add the following redirect url to your Azure App Registration:
https://backstage.test.com/api/auth/microsoft/handler/frame

You will need to update your GitHub Application's Home URL to 
https://backstage.test.com
Change the backend url to:
https://backstage.test.com/api/auth/github/handler/frame

You should be able to access the backstage through https://backstage.test.com/
You can connect to the DB by using pgadmin with the following details:
Host: psql-postgresql
Port: 5432
Username: backstage
Password: backstage

NOTE: if you are experiencing issues connecting to the port when something else on your system is using it, you can forward a different port to the pod:
```
kubectl port-forward svc/psql-postgresql 15432:5432 -n backstage
```

To remove your k8s deployment, you can run the following:
```
kubectl delete -f .\backstage-k8s.yaml -n backstage
helm uninstall psql -n backstage
kubectl delete namespace backstage
```