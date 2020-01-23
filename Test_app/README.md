# Project Title

Kubernets POC


### Prerequisites

```
Minikube or Kubernets Cluster on baremetal or GKE or EKS or AKS service.
Kubectl command line tool
```
## Cluster Test

Connect to master plane and check connected nodes.

```
kubectl get nodes
```

## Deployment

Change directory to k8s

```
cd k8s
```
## Create Persistent Volume For Mysql

```
kubectl apply -f mysql-pv.yml
```

Note: It will create pv and pvc for mysql.


## Deployment of mysql

```
kubectl apply -f mysql-deployment.yml

```

## Deployment of Backend

```
kubectl apply -f appJar-deployment.yml

```

## Testing and Verifying Deployment 

To check created pods: 

```
kubectl get pods

```

Detail info of pods: 

```
kubectl describe pods pod-id

```

To check created services: 

```
kubectl get svc

```


Note: Services are using node port so access application via node port.
