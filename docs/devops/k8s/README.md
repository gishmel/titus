# Deploy on Kubernetes

To set up a [Titus] deployment on an [Kubernetes] using [Helm] and [kinD], there are few steps to be performed.

## Requirements:
- Helm installed locally.
- A local running kinD cluster with a custom configuration, including ingress controller and port mappings.
- Kubectl command line tool installed.
- Docker installed locally to build the images.


## Infrastructure Stack

The stack is built with minimum number of services to make Titus work on Azure:
- **titus-postgresql**: Postgres SQL server and a "titus" DB in it.
- **titus-backend**: Container Instances for the backend.
- **titus-frontend**: Container Instances for the frontend.
- **titus-db-manager**: Container Instances for the db-manager.


## Helm and KinD

You can find instructions on how to install helm and kinD on you specific opertion system at [Install Helm](https://helm.sh/docs/helm/helm_install/) and [Install kinD](https://kind.sigs.k8s.io/docs/user/quick-start/).

If you not already enabled and run an ingress controller inside your kinD cluster you can use the following config ([Ingress for kinD](https://kind.sigs.k8s.io/docs/user/ingress/)).
```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP

```

This is not mandatory to expose your services and let the frontend work as expected by requesting the backend service at it's external URL.
Instead of using a ingress controler you can switch your services from ClusterIP to NodePort as described below.


## Update the Container images

If you use kinD you need to upload your custom docker images to the cluster or provide an docker registry. 

For simplicity let's build and upload all images to kinD itself.

```bash
$ cd packages/titus-backend && docker build -t titus-backend:latest .
$ cd packages/titus-db-manager && docker build -t titus-db-manager:latest .
$ cd packages/titus-frontend && cp .env.sample .env && docker build -t titus-frontend:latest .
```
```bash
$ kind load docker-image titus-backend:latest 
$ kind load docker-image titus-frontend:latest 
$ kind load docker-image titus-db-manager:latest 
```

## Config and Deploy

If you want to expose your services for testing porposes you can just switch from clusterIP to NodePort inside values.yaml

You can enable ingress for the frontend and backend service to expose your services for production use inside the values.yaml. Be sure you already have installed an ingress controller.
```yaml
frontend:
  ingress:
    enabled: true
    ...
backend:
  ingress:
    enabled: true
```

By default the ingress settings configure titus-frontend.localhost and titus-backend.localhost subdomains listen for. Make sur your frontend image was build with the right backend API url configured.

For Database use the helm chart deploy postgresql subchart by default and creates a secret for you. If you want to use your own (external) database and/or your own already existing secret for the db password you can config that in the values.yaml as well.

```yaml
postgresql:
  enabled: true
  postgresqlUsername: titus
  postgresqlPassword: ""
  postgresqlDatabase: titus
  existingSecret: ""
externalDatabase:
  host: "postgresql.example.com"
  port: 5432
  user: titus
  password: ""
  database: titus
  existingSecret: "titus-postgresql"
```

If you decide to use your own secret please make sure it contains at least the keys named 'postgresql-password' for the unprevileged user titus and postgresql-postgres-password for root user.

If you fully configured the chart and you are happy let's deploy Titus
```bash
$ cd infra/k8s/helm && helm upgrade -i -n titus --create-namespace --kubeconfig {PATH_TO_YOUR_KUBECONFIG} -f values.yaml titus .
```

## Verifying Kubernetes deployment

- To check if your Frontend is working as expected open you browser and go to http://titus-frontend.localhost
- For the backend you can curl directly the healthcheck endpoint 
```bash
$ curl -kL http://titus-backend.localhost/healthcheck
```

## Cleanup Kubernetes deployment

If you don't want anymore Titus to be deployed to you cluster just let helm delete your deployment
```bash
$ helm delete -n titus --kubeconfig {PATH_TO_YOUR_KUBECONFIG} titus
```

Helm will not delete persistsnce volumes. If you are sure you dont want your database data and configuration anymore you have to delete it manually via kubectl

To delete you fully kinD cluster ust type
```bash
$ kind delete cluster
```

[Helm]: https://helm.sh/
[Kubernetes]: https://kubernetes.io/
[Titus]: https://github.com/nearform/titus
[kinD]: https://kind.sigs.k8s.io/