# Conduit: An example application of The Kubernetes 

# Change directory

```
cd k8s/Conduit_app
```
## Create Persistent Volume For Mysql

```
kubectl apply -f conduit-mysql-pv.yaml
```

Note: It will create pv and pvc for mysql.


## Deployment of mysql

```
kubectl apply -f conduit-mysql-deployment.yaml

```

## Deployment of Backend

```
kubectl apply -f conduit-api-deployment.yaml

```

## Deployment of Frontend

```
kubectl apply -f conduit-frontend-deployment.yaml
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
