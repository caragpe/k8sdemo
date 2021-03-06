# K8s Demo

## Intent
This repo shows a basic setup for a single application, based on a theoretical API container (usersapi) sending requests to a database (usersdb).

## Components

### Deployments

There are two main deployment files:
- **usersapi.deployment.yaml**
  Its purpose is to work as an API service layer sending requests to a pod containing the required database.
- **usersdb.deployment.yaml**
  Its purpose is to work as a pod database receiving requests from the API pod. 

All variables have been extracted to a secret or config map component for privacy and readability.

### Secrets and ConfigMap

The **usersdb.secret.yaml** component stores secret values related to the database as the root password, and an admin user that will be created upon pod creationg. In a real scenario, we wan to keep this file as private as possible and avoid publishing it to a repo.

The **variables.configmap.yaml** component stores information regarding the pod database url (to allow connecting to it) and the database name to be created when the database pod spins up.

### Services

There are two services, integrated in each deployment file to facilitate reading:

- **usersdb-service**
  It allows connectivity to the database pod on specified port (3306) as per its docker image description
- **usersapi-service**
  Similarly to above, it allows connectivity to the API pod(s) based on a public ip address and a public port, different from the port that our API application is listening to. This is acting as a load balancer.

## Horizontal Pod Autoscaler

We want to be able to smoothly scale up/down based on CPU for our API pod(s). We achieve this by creating an HPA component that fluctuates between 1 and 10 pods, trying to keep an average usage of all CPUs below 70%.

Once that average usage goes over the limit, our HPA will try to spin up a new usersapi pod (if possible) to load balance the requests.

Additionally, the deployment for usersapi has a cap on 80% CPU usage, in order to try to keep our pods alive.

# Notes

## Local setup

This demo was prepared using the following tools in my local environment:

- **minikube** 
  - v1.23.1
- **hyperkit** 
  - v0.20210107-9-gacbc2d
- **kubectl** (as part of minikube)
  ```
  Client Version: version.Info{
    Major:"1", 
    Minor:"21", 
    GitVersion:"v1.21.3",
    GitCommit:"ca643a4d1f7bfe34773c74f79527be4afd95bf39,
    GitTreeState:"clean", BuildDate:"2021-07-15T21:04:39Z",
    GoVersion:"go1.16.6",
    Compiler:"gc",
    Platform:"darwin/amd64"}
  
  Server Version: version.Info{
    Major:"1",
    Minor:"22",
    GitVersion:"v1.22.1",
    GitCommit:"632ed300f2c34f6d6d15ca4cef3d3c7073412212,
    GitTreeState:"clean",
    BuildDate:"2021-08-19T15:39:34Z",
    GoVersion:"go1.16.7",
    Compiler:"gc",
    Platform:"linux/amd64"}
  ```


## How to apply this infra
It is recommended to deploy first the `usersdb.secret.yaml` and `variables.configmap.yaml`, prior to anything else, as some secrets/variables are required by the rest of the components.

You can do it using the following steps:

```
kubectl apply -f usersdb.secret.yaml
kubectl apply -f variables.configmap.yaml
```

Then, you can apply the rest of the components:

```
kubectl apply -f usersdb.deployment.yaml
kubectl apply -f usersapi.deployment.yaml
kubectl apply -f usersapi.autoscale.yaml
```
## Rolling deployments and rollbacks
At this stage, updating the corresponding deployment file will allow to perform rollbacks and rolling deployments smoothly by modifying the container image name.
## Limitation on the database component
As you can see, there is no **description about the persistance of the database component**. For now, this may be out of scope for this demo, but the end goal would be to either remove the database deployment and use a regular database instance, or keep the database pod and connect it to a volume component where data persists in case the pod/replica set/database deployment is deleted.

## Limitations on the load balancer component
The use case describe here is mean for local/development usage. In a real scenario we would like to access `https://my-example-api.com`, for example, and access our API service directly, without needing to know the port.

This production-ready scenario requires the setup of an `ingress` component and some ingress rules about what we do when we get a request to our URL (path, port to redirect to, erros, etc).
