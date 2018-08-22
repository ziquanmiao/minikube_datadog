
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

# Use the Flask App

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

# Some points of interest

A side-care Datadog agent container is deployed and is now acting as a collector and middleman between the spun up services and Datadog backend. Through actions -- curling the endpoints -- and the lack-thereof, metrics will be generated and directed to the corresponding Datadog Account based off your supplied API key

Below is a quick discussion on some points of interest

## Infrastructure Product
This part pertains to the ingestion of timeseries data, status checks, and events. 

By deploying the agent referencing the [Datadog Container Image](https://github.com/ziquanmiao/minikube_datadog/blob/ba94f6072fbfccbaaf8595020690df9b2f6ebdfb/agent_daemon.yaml#L13) in the agent_daemon.yaml file, the check automatically comes prepackaged with system level (CPU, Mem, IO, Disk), [Kubernetes](https://docs.datadoghq.com/agent/kubernetes/), and [Docker](https://docs.datadoghq.com/integrations/docker_daemon/) level checks.

The gist of the setup portion is:
1. Deploy the agent daemon the proper environment variables, volume and volumeMount arguments
2. Deploy relevant applications with annotations
3. Validate metrics go to agent and ends up in our application

### System Metric Requirements

[volumeMount](https://github.com/ziquanmiao/minikube_datadog/blob/ba94f6072fbfccbaaf8595020690df9b2f6ebdfb/agent_daemon.yaml#L65-L67) and [volume](https://github.com/ziquanmiao/minikube_datadog/blob/ba94f6072fbfccbaaf8595020690df9b2f6ebdfb/agent_daemon.yaml#L93-L95) for the proc directory is required from the host level

In the Datadog web application you can reference the [host map](datadoghq.com/infrastructure/map), and filter on the particular hostname to see what is going on.

### Kubernetes/Docker
docker.sock and cgroup [volumeMounts](https://github.com/ziquanmiao/minikube_datadog/blob/ba94f6072fbfccbaaf8595020690df9b2f6ebdfb/agent_daemon.yaml#L63-L70) and [volumes](https://github.com/ziquanmiao/minikube_datadog/blob/ba94f6072fbfccbaaf8595020690df9b2f6ebdfb/agent_daemon.yaml#L90-L98) are required to be attached in the daemonset

### Autodiscovery
Application/Service Pods and Containers go up and down. The Datadog agent traditionally requires a modification of the hardcoded host/port values of corresponding configuration files (example with [postgres](https://github.com/DataDog/integrations-core/blob/fd4414ed3d85a6ad835f6440f4bd091a4cf1a0f2/postgres/datadog_checks/postgres/data/conf.yaml.example#L4-L5)) and an agent restart to collect Data for installed software.

Rather than having a Mechanical Turk sit on standy ready to make the changes, the containerized accommodates makes this process automagic using [Autodiscovery](https://docs.datadoghq.com/agent/autodiscovery/) where the agent has the capability to monitor the annotations of deployments and automatically establish checks as pods come and go.

#### Postgres Example
Autodiscovery of the postgres pod in this container is straightforward, [simply annotation](https://github.com/ziquanmiao/minikube_datadog/blob/ba94f6072fbfccbaaf8595020690df9b2f6ebdfb/postgres_deployment.yaml#L14-L17) in the configuration file by adding in the typical check sections required.

Note the annotation arguments [here](https://cl.ly/d77e73e9786d) must be identical for the agent to properly connect to the container.

## Live Processes/Containers
[Live Process/Container Monitoring](https://www.datadoghq.com/blog/live-process-monitoring/) is the capability to get container and process level granularity for all monitored systems.
This feature provides not only standard system level metrics at the process/container level, but also on the initial run commands used to set up the process/container.

Simply add [DD_PROCESS_AGENT_ENABLED](https://github.com/ziquanmiao/minikube_datadog/blob/ba94f6072fbfccbaaf8595020690df9b2f6ebdfb/agent_daemon.yaml#L38-L39) env variable in the daemonset to turn on this feature

### Requirements
Sometimes passwords are revealed in the initial run commands, the agent comes equipped with [passwd](https://github.com/ziquanmiao/minikube_datadog/blob/ba94f6072fbfccbaaf8595020690df9b2f6ebdfb/agent_daemon.yaml#L72-L74) to remove a [standard set of arguments](https://docs.datadoghq.com/graphing/infrastructure/process/#process-arguments-scrubbing)

We again need [docker.sock](https://docs.datadoghq.com/graphing/infrastructure/process/#kubernetes-daemonset) to get container information.

## APM Tracing
The same agent that handles infrastructure metrics can also accommodate receiving [Trace Data](https://www.datadoghq.com/blog/tag/apm/) from a designated [APM module](datadoghq.com/apm/docs) -- these modules sit on top of your applications and forward [payloads](https://docs.datadoghq.com/api/?lang=bash#send-traces) to a local Datadog agent to middle man to our backend.

Applications are spinning up in pods and we need payloads being fired to the sidecar agent pod. In this example, we set up a route between the pods with a port going through the host level.

### Requirements

#### From the agent daemon side
Enable agent to receive traces from the Agent deployment side via [environment variables](https://github.com/ziquanmiao/minikube_datadog/blob/ba94f6072fbfccbaaf8595020690df9b2f6ebdfb/agent_daemon.yaml#L26-L27)

Create a [port connection to host](https://github.com/ziquanmiao/minikube_datadog/blob/ba94f6072fbfccbaaf8595020690df9b2f6ebdfb/agent_daemon.yaml#L21-L24) via the 8126 port

#### From the application Side
Provide the deploy file with a [link to the host level](https://github.com/ziquanmiao/minikube_datadog/blob/ba94f6072fbfccbaaf8595020690df9b2f6ebdfb/app_deployment.yaml#L27-L32) for port 8126 via environmental variables, so that applications can reference the host/port values to fire traces to

##### Flask specific
Have the [ddtrace module](https://github.com/ziquanmiao/minikube_datadog/blob/ba94f6072fbfccbaaf8595020690df9b2f6ebdfb/flask_app/requirements.txt#L2)

In the app.py code, [import ddtrace module](https://github.com/ziquanmiao/minikube_datadog/blob/ba94f6072fbfccbaaf8595020690df9b2f6ebdfb/flask_app/app.py#L18-L25) and patch both [sqlalchemy](https://github.com/ziquanmiao/minikube_datadog/blob/ba94f6072fbfccbaaf8595020690df9b2f6ebdfb/flask_app/app.py#L27) and the [Flask app object](https://github.com/ziquanmiao/minikube_datadog/blob/ba94f6072fbfccbaaf8595020690df9b2f6ebdfb/flask_app/app.py#L37).

**Note**: the trace module is an implementation as all modules are. If certain spans are not being captured, you can always [hardcode](https://github.com/ziquanmiao/minikube_datadog/blob/ba94f6072fbfccbaaf8595020690df9b2f6ebdfb/flask_app/app.py#L102) them in.

### Validation

#### Agent Side
run ``` kubectl get pods``` to get the pod name of the agent container.

run ```kubectl exec -it POD_NAME bash ``` to hop into the container

run ```agent status``` to see the status summary and look for the tracing section to see agent has tracing turned on

run ``` cat /var/log/datadog/trace-agent.log``` to see logs pertaining to the trace agent

#### Datadog Side

head over to [Trace Services Page](datadoghq.com/apm/services) and look for your service level metrics and traces!

## Logs

