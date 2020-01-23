# Persistent Volume with NFS in Kubernetes 

# Install and configure NFS server on new ubuntu machine.

ssh to NFS server machine and use following commands to install nfs.

```
sudo apt-get update
sudo apt-get install nfs-kernel-server
```
# Create folder to share via nfs.

```
sudo mkdir /shome
```
# Create index file inside folder.

```
sudo vim /shome/index.html 
Add html code
save and exit
```

#Add folder configrutaion main config file for NFS.

```
sudo vim /etc/exports 

Add below line end of file.

/shome        *(rw,sync,no_subtree_check,insecure)

```
save and exit

# Restart the NFS server

```
sudo /etc/init.d/nfs-kernel-server restart

```

# Export the latest config

```
sudo exportfs -rav

```
# Test Mount 

```
sudo showmount -e
```

#IMP: Need to install nfs client on k8s node
```
For Ubuntu :
 sudo apt install nfs-common

For Centos :
  yum -y install nfs-utils
 
```

#Depolyment 

  Colen the repo

  ```
   git clone git@github.com:kajal414/Kubernetes.git

  ```

# Change directory

```
cd /Kubernetes/k8s/NFS_Persistent_Volume
```
## Create Persistent Volume with nfs

```
kubectl apply -f nfs-pv.yml
```

Note: It will create pv from nfs.


## Create Persistent Volume claim.

```
kubectl apply -f nfs-pvc.yml
```


## Deployment of Nginx

```
kubectl apply -f nfs-nginx.yaml

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

Note: Services are using node port so access application via node port.Connect to NFS server and change the index file content and check new content are able to see in browser.

     
