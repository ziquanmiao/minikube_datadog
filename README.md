
# Intro

This Workshop assumes [minikube](https://github.com/kubernetes/minikube/blob/v0.28.2/README.md) is installed (along with kubectl + kubernetes) and has only been tested in a Mac environment.

In addition, you will need a Datadog Account and have access to an API key -- Start a Free Trial [Here](https://www.datadoghq.com/lpg6/)!

This repo showcases a minikube-based path to deploying a simple flask app container that returns some sample text contained in a separate postgres container. 

The goal of this repo is to demonstrate the steps involved in installing a [Datadog]
(datadoghq.com/) agent to demonstrate the product's [Infrastructure Monitoring](https://www.datadoghq.com/server-monitoring/), [Application Performance Monitoring](https://www.datadoghq.com/blog/announcing-apm/), [Live Process/Container Monitoring](https://www.datadoghq.com/blog/live-process-monitoring/), and [Log Monitoring Capabilities](https://www.datadoghq.com/blog/announcing-logs/) in a Kubernetes x Docker based environment.

This repo makes no accommodations for proxy scenarios or situations where machines are unable to pull from the internet to download packages

# Steps to Success

The gist of the setup portion is:
1. Spin up Minikube Instance
2. Load Dockerfiles into images
3. Deploy!

## Set up 
Start Minikube instance 
```
minikube start
```
In Mac's case, there may be a need to use localkube as a bootstrapper

```
minikube start --bootstrapper=localkube
```

You will need to use minikube's docker engine by running:
```
eval $(minikube docker-env)
```

## build images

Then, build the images based off the provided Dockerfiles
```
docker build -t sample_flask:007 ./flask_app/
docker build -t sample_postgres:007 ./postgres/
```

## deploy things

Deploy the application container and turn it into a service
```
kubectl create -f app_deployment.yaml
kubectl create -f app_service.yaml
```

Deploy the postgres container
```
kubectl create -f postgres_deployment.yaml
```

Deploy the Datadog agent container

**super important** edit the agent_daemon.yaml file and include the API KEY for `DD_API_KEY` [environment variable](https://cl.ly/2q3U3l1b240v)
```
kubectl create -f agent_daemon.yaml
```

Deploy a nonfunction pause container to demonstrate Datadog [AutoDiscovery](https://docs.datadoghq.com/agent/autodiscovery/) via a simple HTTP check against www.google.com
```
kubectl create -f pause.yaml
```

Deploy kubernetes state files to demonstrate [kubernetes_state check](https://docs.datadoghq.com/integrations/kubernetes/#setup-kubernetes-state)

```
kubectl create -f kubernetes
```

Deploy a logs ConfigMap to demonstrate Datadog Daemon Log parsing

```
kubectl create -f logsConfigMap.yaml
```

And we are done!

# Points of Interest

The Flask App offers 3 endpoints that returns some text `IP:5000/`, `IP:5000/query`, `IP:5000/bad`

Run ```kubectl get services``` to find the IP address of the flask application service

You can then access the endpoints within the minikube vm:
```
minikube ssh
```

then hit one of the following:
```
curl FLASK_SERVICE_IP:5000/
curl FLASK_SERVICE_IP:5000/query
curl FLASK_SERVICE_IP:5000/bad
```
to see the Flask application at work

