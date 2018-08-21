
# Intro

This Workshop assumes [minikube](https://github.com/kubernetes/minikube/blob/v0.28.2/README.md) is installed (along with kubectl + kubernetes) and has only been tested in a Mac environment
In addition, you will need a Datadog Account and have access to an API key -- Start a Free Trial [Here](https://www.datadoghq.com/lpg6/)!

This repo contains a minikube based pathing to deploy a simple flask app container that returns some sample text contained in a separate postgres container. 
The goal of this repo is to demonstrate the pathing involved in installing a [Datadog
(datadoghq.com/) agent to demonstrate the product's [Infrastructure Monitoring](https://www.datadoghq.com/server-monitoring/), [Application Performance Monitoring](https://www.datadoghq.com/blog/announcing-apm/), [Live Process/Container Monitoring](https://www.datadoghq.com/blog/live-process-monitoring/), and [Log Monitoring Capabilities](https://www.datadoghq.com/blog/announcing-logs/) in a Kubernetes x Docker based environment.

This repo makes no accommodations for proxy scenarios or situations where machines are unable to pull from the internet to download packages

# Steps to Success

The gist of the setup portion is 
## Set up 
Start Minikube instance 
```
minikube start
```
In Mac's case, there may be a need to use localkube as a bootstrapper

```
minikube start --bootstrapper=localkube
```

Use minikube's docker
```eval $(minikube docker-env)```

## build images
docker build -t sample_flask:007 ./flask_app/
docker build -t sample_postgres:007 ./postgres/

## deploy things

deploy the application container and turn it into a service
```
kubectl create -f app_deployment.yaml
kubectl create -f app_service.yaml
```

deploy the postgres container
```
kubectl create -f postgres_deployment.yaml
```

deploy the Datadog agent container
**important** edit the agent_daemon.yaml file and include the API KEY for `DD_API_KEY` [environment variable](https://cl.ly/2q3U3l1b240v)
```
kubectl create -f agent_daemon.yaml
```

deploy a nonfunction pause container to demonstrate Datadog [AutoDiscovery](https://docs.datadoghq.com/agent/autodiscovery/) via a simple HTTP check against www.google.com
```
kubectl create -f pause.yaml
```

deploy kubernetes state files to demonstrate [kubernetes_state check](https://docs.datadoghq.com/integrations/kubernetes/#setup-kubernetes-state)

```
kubectl create -f kubernetes
```

And we are done!

## Points of Interest

WIP
