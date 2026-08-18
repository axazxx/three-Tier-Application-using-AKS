# Three Tier Architecture Deployment on Azure AKS

Stan's Robot Shop is a sample microservice application you can use as a sandbox to test and learn containerised application orchestration and monitoring techniques. It is not intended to be a comprehensive reference example of how to write a microservices application, although you will better understand some of those concepts by playing with Stan's Robot Shop. To be clear, the error handling is patchy and there is not any security built into the application.

Forked from https://github.com/instana/robot-shop
Here you can get more detailed information about this sample microservice application.

This sample microservice application has been built using these technologies:
- NodeJS ([Express](http://expressjs.com/))
- Java ([Spring Boot](https://spring.io/))
- Python ([Flask](http://flask.pocoo.org))
- Golang
- PHP (Apache)
- MongoDB
- Redis
- MySQL ([Maxmind](http://www.maxmind.com) data)
- RabbitMQ
- Nginx
- AngularJS (1.x)

The various services in the sample application already include all required Instana components installed and configured. The Instana components provide automatic instrumentation for complete end to end [tracing](https://docs.instana.io/core_concepts/tracing/), as well as complete visibility into time series metrics for all the technologies.

To see the application performance results in the Instana dashboard, create an Instana account.

## Build from Source
To optionally build from source (you will need a newish version of Docker to do this) use Docker Compose. Optionally edit the `.env` file to specify an alternative image registry and version tag; see the official [documentation](https://docs.docker.com/compose/env-file/) for more information.

To download the tracing module for Nginx, it needs a valid Instana agent key. Set this in the environment before starting the build.

```shell
$ export INSTANA_AGENT_KEY="<your agent key>"
```

Now build all the images.

```shell
$ docker-compose build
```

If you modified the `.env` file and changed the image registry, you need to push the images to that registry. When deploying to AKS this registry will typically be **Azure Container Registry (ACR)**:

```shell
$ docker-compose push
```

## Run Locally
You can run it locally for testing.

If you did not build from source, don't worry all the images are on Docker Hub. Just pull down those images first using:

```shell
$ docker-compose pull
```

Fire up Stan's Robot Shop with:

```shell
$ docker-compose up
```

If you want to fire up some load as well:

```shell
$ docker-compose -f docker-compose.yaml -f docker-compose-load.yaml up
```

If you are running it locally on a Linux host you can also run the Instana [agent](https://docs.instana.io/quick_start/agent_setup/container/docker/) locally, unfortunately the agent is currently not supported on Mac.

There is also only limited support on ARM architectures at the moment.

## Azure Kubernetes Service (AKS)

You can run Kubernetes locally using [minikube](https://github.com/kubernetes/minikube), or deploy straight to **Azure Kubernetes Service (AKS)**, Azure's fully managed Kubernetes offering.

### 1. Prerequisites

- An active Azure subscription
- The [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli) installed and logged in (`az login`)
- `kubectl` and `helm` installed on your workstation/bastion host
- (Optional) [`eksctl`-style convenience is not needed here — AKS clusters are created and managed via `az aks` commands]

### 2. Create a Resource Group and AKS Cluster

```shell
$ az group create --name robot-shop-rg --location eastus

$ az aks create \
    --resource-group robot-shop-rg \
    --name robot-shop-aks \
    --node-count 3 \
    --node-vm-size Standard_D2s_v3 \
    --generate-ssh-keys \
    --enable-managed-identity
```

> If you scale down to a smaller/cheaper node size, keep an eye on `kubectl get pods -n robot-shop` — under-provisioned nodes are a common cause of pods staying `Pending` due to insufficient CPU/memory.

### 3. Connect `kubectl` to Your Cluster

```shell
$ az aks get-credentials --resource-group robot-shop-rg --name robot-shop-aks

$ kubectl get nodes
```

### 4. (Optional) Push Images to Azure Container Registry

If you built your own images rather than using the public Docker Hub images, push them to ACR instead of a generic registry:

```shell
$ az acr create --resource-group robot-shop-rg --name robotshopacr --sku Basic

$ az aks update --resource-group robot-shop-rg --name robot-shop-aks --attach-acr robotshopacr

$ az acr login --name robotshopacr
$ docker-compose push
```

Update the image registry values in your `.env` file or Helm `values.yaml` to point at `robotshopacr.azurecr.io` instead of Docker Hub.

## Kubernetes Deployment

The Docker container images are all available on [Docker Hub](https://hub.docker.com/u/robotshop/).

Install Stan's Robot Shop to your AKS cluster using the [Helm](K8s/helm/README.md) chart:

```shell
$ kubectl create namespace robot-shop

$ cd K8s/helm
$ helm install robot-shop --namespace robot-shop .
```

Check that everything came up cleanly:

```shell
$ kubectl get pods -n robot-shop
$ kubectl get svc -n robot-shop
$ kubectl describe pods -n robot-shop   # useful if pods are stuck Pending/CrashLoopBackOff
```

## Accessing the Store

If you are running the store locally via `docker-compose up` then, the store front is available on localhost port 8080: [http://localhost:8080](http://localhost:8080/)

If you are running the store on **AKS**, the `web` service is typically exposed as a `LoadBalancer` type, which provisions an Azure Load Balancer with a public IP automatically:

```shell
$ kubectl get svc web -n robot-shop
```

Wait for the `EXTERNAL-IP` column to populate (this can take a minute or two), then browse to `http://<EXTERNAL-IP>:8080`.

For more control over routing (path-based rules, TLS termination, custom domains), consider deploying an **Ingress Controller** (e.g. NGINX Ingress or the [AGIC — Application Gateway Ingress Controller](https://learn.microsoft.com/en-us/azure/application-gateway/ingress-controller-overview)) instead of a plain LoadBalancer service:

```shell
$ kubectl apply -f ingress.yaml
$ kubectl get ingress -n robot-shop
```

With Ingress configured, the application will be reachable via the DNS name or IP assigned to your Ingress Controller / Application Gateway.

## Load Generation
A separate load generation utility is provided in the `load-gen` directory. This is not automatically run when the application is started. The load generator is built with Python and [Locust](https://locust.io). The `build.sh` script builds the Docker image, optionally taking `push` as the first argument to also push the image to the registry. The registry and tag settings are loaded from the `.env` file in the parent directory. The script `load-gen.sh` runs the image, it takes a number of command line arguments. You could run the container inside AKS as well if you want to — an example descriptor is provided in the K8s directory. For End-user Monitoring, load is not automatically generated but by navigating through the Robot Shop from the browser. For more details see the [README](load-gen/README.md) in the load-gen directory.

## Website Monitoring / End-User Monitoring

### Docker Compose

To enable Website Monitoring / End-User Monitoring (EUM) see the official [documentation](https://docs.instana.io/website_monitoring/) for how to create a configuration.

### Kubernetes / AKS

The Helm chart for installing Stan's Robot Shop supports setting the key and endpoint url required for website monitoring, see the [README](K8s/helm/README.md). This is set the same way regardless of the underlying cloud provider.

## Prometheus

The cart and payment services both have Prometheus metric endpoints. These are accessible on `/metrics`. The cart service provides:

* Counter of the number of items added to the cart

The payment services provides:

* Counter of the number of items purchased
* Histogram of the total number of items in each cart
* Histogram of the total value of each cart

To test the metrics use:

```shell
$ curl http://<host>:8080/api/cart/metrics
$ curl http://<host>:8080/api/payment/metrics
```
