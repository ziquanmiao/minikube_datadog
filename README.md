This Workshop will assume [minikube](https://github.com/kubernetes/minikube/blob/v0.28.2/README.md) is installed

This has only been tested in a Mac environment but should generally work for linux as well

The goal of this workshop is to demonstrate via a simple Flask app the pathing involved to install a dataagent, accommodate [APM tracing](https://docs.datadoghq.com/tracing/setup/kubernetes/) and dogstatsd, as well as autodiscovery of Datadog integration related pods

minikube start --bootstrapper=localkube

#use minikube docker
eval $(minikube docker-env)
#build image
docker build -t sample_flask:007 ./flask_app/


#### k8s deploy way

#make deployment + create the pods
kubectl create -f app_deployment.yaml

#make the service
kubectl create -f app_service.yaml

#double check it works
minikube ssh
# curl EXTERNAL_IP:5000 (run kubectl get services)

#run daemonset
kubectl create -f agent_daemon.yaml


###Make Postgres container
docker build -t sample_postgres:007 ./postgres/

#make deployment + create pod
kubectl create -f postgres_deployment.yaml
kubectl exec -it DOCKER_POD bash
psql -h localhost -p 5432 -U docker -d docker

#pause
kubectl create -f pause.yaml

#kube_state
kubectl apply -f kubernetes/
#####Clearing
minikube stop
minikube delete
rm -rf 

#delete daemonset
kubectl delete -f agent_daemon.yaml
kubectl delete service flask-app
kubectl delete deployment flask-app
kubectl delete -f postgres_deployment.yaml
