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