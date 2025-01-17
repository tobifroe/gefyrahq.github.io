---
layout: page
title: Remote Development on Google Kubernetes Engine
permalink: /usecases/remote-development-with-gke/
nav_order: 4
parent: Use Cases and Demos
---
# Developing a backend application with Django on the Google Kubernetes Engine (GKE) 
{:.no_toc}

Intermediate Usecase
{: .label .label-blue }

This guide will show you how to use Gefyra for the local development of a Kubernetes
Application running in GKE. Gefyra is able to spin up a local container which
behaves like it was part of the cluster already. This way you can run services
inside the cluster without abstaining from features like hot code reload. Sounds
good? Lets go!
{: .fs-6 .fw-300 }

This is more of an advanced use-case, if you just want an easy example of how Gefyra works, check
out the [getting started guide](https://gefyra.dev/getting-started/).

<hr />

### What you will learn
{:.no_toc}
* How to set up a Kubernetes cluster with the Google Kubernetes Engine
* Deploy a Django-based demo application 
* Set out with Gefyra to run a local container instance as part of the remote cluster
<hr />
## Prerequisites

To follow this guide, you need to have the following tools installed:

You need to have the following tools installed:

* [gcloud](https://cloud.google.com/sdk/docs/install-sdk)
* [kubectl](https://kubernetes.io/docs/tasks/tools/)
* [Helm](https://helm.sh/docs/intro/install/)
* [Gefyra](https://gefyra.dev/installation/)

Additionally you need an account for the Google Cloud Platform including the
permission to create a new cluster. Make sure your gcloud is using the right
project configuration. Googles documentation is available
[here](https://cloud.google.com/docs/get-started).

## Setup a cluster

In this guide we will spin up a small demo application called spacecrafts,
featuring Django Hurricane. If you already have a cluster running, you are free
to use this as well.

The easiest way to create a new cluster is using `gcloud`:

`gcloud container clusters create spacecraft`.

This may take a few minutes, there will be 3 VM instances running a kubernetes
cluster ready to serve your applications. `gcloud` will set your
kubectl context to the created cluster, nothing to worry about!

The last thing we need to do is open a port in the firewall. This allows gefyra
to connect to the cluster using wireguard:

`gcloud compute firewall-rules create gefyra --allow udp:31820`

## Running the Spacecrafts Demo

To have some actual application to develop against, we want to deploy our
spacecrafts demo to our cluster. Clone the repository and deploy it using helm:

```
git clone https://github.com/django-hurricane/spacecrafts-demo.git
cd spacecrafts-demo/helm
helm install spacecrafts spacecrafts/
```

Check if everything is up and running with `kubectl get pods`.

Now we need to deploy a load balancer so Google exposes our service on a public
IP.  Use the following service definition with `kubectl apply -f
service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello
spec:
  type: LoadBalancer
  selector:
    app.kubernetes.io/name: spacecrafts
  strategy:
  ports:
  - port: 80
    targetPort: 8080
---
```

It takes some time to actually get an IP. You can check the state running
`kubectl get service`. There will be a service named `hello` of the type
`LoadBalancer`. As soon as an IP is available, feel free to visit it in your
browser. If you can see a 404 error from Django, everything is alright!

## Setting up Gefyra

Now we can run `gefyra up`. This way gefyra sets up all requirements
to enjoy local development supported by services in the cluster.

At first, we need an endpoint IP of one of our compute instances. You can get
them with running `gcloud compute instances list`. Pick one of them.

Now you can run `gefyra up --endpoint <IP>:31820`.

Gefyra now sets up a wireguard connection into the cluster and prepares
everything to allow us to run local containers linked to the cluster.

## Local development backed by the cluster

Lets say we want to debug and fix the 404 on the index page. It's possible to do
this in the remote cluster, but Gefyra enables us to do this on your machine!

The following command will spin your local development container for spacecrafts
up. Please note you need to adjust the name of the container, run `kubectl get
pods` to get the right identifier.

```bash
export podname=$(kubectl get pods --no-headers -o custom-columns=":metadata.name" | grep -v postgres)
gefyra run --env-from $podname/spacecrafts \
 -i quay.io/django-hurricane/spacecrafts-demo \
 -N myspacecraft \
 -v src/:/app \
 -c "python manage.py serve 
  --autoreload 
  --static 
  --command 'collectstatic 
  --noinput' 
  --command 'migrate'" 
```

Some explanation: We pass the image for the container with `-i`, name the container with `-N`, mount our 
source directory inside the container for hot-reloading using `-v` and specify the command to be executed 
on startup.

Thats it. To get the IP of the container, run 

`gefyra list --containers`

Now you can open your browser at <IP>:8000 and get the same `404-Error`. You can watch the logs using 
`docker logs -f myspacecraft`.
  
The reason why we get a 404 not found is simply a missing route. We should add one in `src/configuration/urls.py`:

```diff
 urlpatterns = [
     # django-admin:
+    path("", csrf_exempt(GraphQLView.as_view(graphiql=True))),
     path("admin/doc/", include(admindocs_urls)),  # noqa: DJ05
     path("admin/", admin.site.urls),
     path("graphql/", csrf_exempt(GraphQLView.as_view(graphiql=True))),
```
  
If you now reload your browser tab, you should see a graphql input field!
