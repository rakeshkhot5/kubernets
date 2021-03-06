
 ### I have setup one HAproxy Loadbalancer , three master node in kubenetes master in CENT OS 7.6 
 
  
 
                     HAproxy Loadbalancer - 192.168.56.145
                     kubemaster1          - 192.168.56.142
                     kubemaster2          - 192.168.56.143
                     kubemaster3          - 192.168.56.144
  
  
  ### Disable firewalld on all above servers -> #systemctl stop firewalld && systemctl disable firewalld
 ``` 
-------------------------------------------------------------
vi /etc/hosts       - make below entry on all above servers

192.168.56.143 kubemas1.anbu.com kubemas1
192.168.56.144 kubemas2.anbu.com kubemas2
192.168.56.142 kubemas3.anbu.com kubemas3
192.168.56.145 loadbalancer.anbu.com loadbalancer
-------------------------------------------------------------
 ```  
   
### Install HAproxy package on HAproxy Loadbalancer server.
  ``` 
    #yum install haproxy
    
    vi /etc/haproxy/haproxy.cfg
    
        frontend kubernetes
            bind 192.168.56.145:6443
            option tcplog
            mode tcp
            default_backend app
        backend app
            balance     roundrobin
            server  app1 192.168.56.143:6443 check
            server  app2 192.168.56.145:6443 check
            server  app3 192.168.56.142:6443 check
            
     # systemctl daemon-reload      
     # systemctl restart haproxy
     # systemctl enable haproxy
     
---------------------------------------------- ----------------------------------------------------------------------------------------
```

### Installing cfssl on HAproxy loadbalancer  server for to generate certificate files.

1- Download the binaries.
```
# wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
# wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64

2- Add the execution permission to the binaries.

# chmod +x cfssl*

3- Move the binaries to /usr/local/bin.

# mv cfssl_linux-amd64 /usr/local/bin/cfssl
# mv cfssljson_linux-amd64 /usr/local/bin/cfssljson

4- Verify the installation.

# cfssl version
Version: 1.2.0
Revision: dev
Runtime: go1.6
```
-------------------------------------------
### Generating the TLS certificates
*******************************
Create the certificate authority configuration file
```
#vim ca-config.json
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "kubernetes": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}
```
-------------------------------------------
Create the certificate authority signing request configuration file.
```
#vim ca-csr.json
{
  "CN": "Kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
  {
    "C": "IE",
    "L": "Cork",
    "O": "Kubernetes",
    "OU": "CA",
    "ST": "Cork Co."
  }
 ]
}
```

### Generate the certificate authority certificate and private key.
```
#cfssl gencert -initca ca-csr.json | cfssljson -bare ca

Verify that the ca-key.pem and the ca.pem were generated.

#ls -la
```
### Creating the certificate for the cluster
```
#vim kubernetes-csr.json
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
  {
    "C": "IE",
    "L": "Cork",
    "O": "Kubernetes",
    "OU": "Kubernetes",
    "ST": "Cork Co."
  }
 ]
}
```
Generate the certificate and private key.
```
-------------------------------------------------
#cfssl gencert \
-ca=ca.pem \
-ca-key=ca-key.pem \
-config=ca-config.json \
-hostname=192.168.56.142,192.168.56.143,192.168.56.144,192.168.56.145,127.0.0.1,kubernetes.default \
-profile=kubernetes kubernetes-csr.json | cfssljson -bare kubernetes
--------------------------------------------------

Verify that the kubernetes-key.pem and the kubernetes.pem file were generated.

#ls -la
```
### Copying files to masters
```
scp ca.pem kubernetes.pem kubernetes-key.pem root@192.168.56.142:~
scp ca.pem kubernetes.pem kubernetes-key.pem root@192.168.56.143:~
scp ca.pem kubernetes.pem kubernetes-key.pem root@192.168.56.144:~
```


### Install Docker community edition on kubemas1 , kubemas2 , kubemas3 servers.

```
#yum install -y yum-utils \
  device-mapper-persistent-data \
  lvm2


#yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
    
    
#yum install docker-ce

#systemctl start docker && systemctl enable docker

```
### Install kubernetes on kubemas1 , kubemas2, kubemas3 servers.
```
---------------------------------------------
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
-------------------------------------------


# Set SELinux in permissive mode (effectively disabling it)
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
-------------------------------------------


yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

systemctl enable --now kubelet
-------------------------------------------


Some users on RHEL/CentOS 7 have reported issues with traffic being routed incorrectly due to iptables being bypassed.
You should ensure net.bridge.bridge-nf-call-iptables is set to 1 in your sysctl config

---------------------------------------

cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
---------------------------------------
sysctl --system
--------------------------------------
systemctl daemon-reload
systemctl restart kubelet
--------------------------------------
```
### Disable swap on kubemas1 , kubemas2, kubemas3 servers.
```    
   # swapoff -a
   # sed -i '/ swap / s/^/#/' /etc/fstab
```

 ### Install etcd cluster on kubemas1 , kubemas2, kubemas3 servers
  ```
    Create a configuration directory for Etcd
  
      #mkdir /etc/etcd /var/lib/etcd
      

    Move the certificates to the configuration directory.

      #mv ~/ca.pem ~/kubernetes.pem ~/kubernetes-key.pem /etc/etcd

      
  #wget https://github.com/etcd-io/etcd/releases/download/v3.3.18/etcd-v3.3.18-linux-amd64.tar.gz
  
  
      Extract the etcd archive.

  #tar xvzf etcd-v3.3.18-linux-amd64.tar.gz

  
  Move the etcd binaries to /usr/local/bin.

    #mv etcd-v3.3.18-linux-amd64/etcd* /usr/local/bin/

    
  # etcd --version

      etcd Version: 3.3.18
      Git SHA: 3cf2f69b5
      Go Version: go1.12.12
      Go OS/Arch: linux/amd64

```
### Create etcd systemd service file for each kubemas servers
  
   create a etcd.service on kubemas1 server
   ========================================
 ```  
#vi /etc/systemd/system/etcd.service
 
   
[Unit]
Description=etcd
Documentation=https://github.com/coreos


[Service]
ExecStart=/usr/local/bin/etcd \
  --name 192.168.56.143 \
  --cert-file=/etc/etcd/kubernetes.pem \
  --key-file=/etc/etcd/kubernetes-key.pem \
  --peer-cert-file=/etc/etcd/kubernetes.pem \
  --peer-key-file=/etc/etcd/kubernetes-key.pem \
  --trusted-ca-file=/etc/etcd/ca.pem \
  --peer-trusted-ca-file=/etc/etcd/ca.pem \
  --peer-client-cert-auth \
  --client-cert-auth \
  --initial-advertise-peer-urls https://192.168.56.143:2380 \
  --listen-peer-urls https://192.168.56.143:2380 \
  --listen-client-urls https://192.168.56.143:2379,http://127.0.0.1:2379 \
  --advertise-client-urls https://192.168.56.143:2379 \
  --initial-cluster-token etcd-cluster-0 \
  --initial-cluster 192.168.56.143=https://192.168.56.143:2380,192.168.56.144=https://192.168.56.144:2380,192.168.56.142=https://192.168.56.142:2380 \
  --initial-cluster-state new \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5


[Install]
WantedBy=multi-user.target
```
    
### create a etcd.service on kubemas2 server
 ```    
#vi /etc/systemd/system/etcd.service

[Unit]
Description=etcd
Documentation=https://github.com/coreos


[Service]
ExecStart=/usr/local/bin/etcd \
  --name 192.168.56.144 \
  --cert-file=/etc/etcd/kubernetes.pem \
  --key-file=/etc/etcd/kubernetes-key.pem \
  --peer-cert-file=/etc/etcd/kubernetes.pem \
  --peer-key-file=/etc/etcd/kubernetes-key.pem \
  --trusted-ca-file=/etc/etcd/ca.pem \
  --peer-trusted-ca-file=/etc/etcd/ca.pem \
  --peer-client-cert-auth \
  --client-cert-auth \
  --initial-advertise-peer-urls https://192.168.56.144:2380 \
  --listen-peer-urls https://192.168.56.144:2380 \
  --listen-client-urls https://192.168.56.144:2379,http://127.0.0.1:2379 \
  --advertise-client-urls https://192.168.56.144:2379 \
  --initial-cluster-token etcd-cluster-0 \
  --initial-cluster 192.168.56.143=https://192.168.56.143:2380,192.168.56.144=https://192.168.56.144:2380,192.168.56.142=https://192.168.56.142:2380 \
  --initial-cluster-state new \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5



[Install]
WantedBy=multi-user.target
```

 ###   create a etcd.service on kubemas3server
    =======================================
```  
# vi /etc/systemd/system/etcd.service

[Unit]
Description=etcd
Documentation=https://github.com/coreos


[Service]
ExecStart=/usr/local/bin/etcd \
  --name 192.168.56.142 \
  --cert-file=/etc/etcd/kubernetes.pem \
  --key-file=/etc/etcd/kubernetes-key.pem \
  --peer-cert-file=/etc/etcd/kubernetes.pem \
  --peer-key-file=/etc/etcd/kubernetes-key.pem \
  --trusted-ca-file=/etc/etcd/ca.pem \
  --peer-trusted-ca-file=/etc/etcd/ca.pem \
  --peer-client-cert-auth \
  --client-cert-auth \
  --initial-advertise-peer-urls https://192.168.56.142:2380 \
  --listen-peer-urls https://192.168.56.142:2380 \
  --listen-client-urls https://192.168.56.142:2379,http://127.0.0.1:2379 \
  --advertise-client-urls https://192.168.56.142:2379 \
  --initial-cluster-token etcd-cluster-0 \
  --initial-cluster 192.168.56.143=https://192.168.56.143:2380,192.168.56.144=https://192.168.56.144:2380,192.168.56.142=https://192.168.56.142:2380 \
  --initial-cluster-state new \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5



[Install]
WantedBy=multi-user.target

```

### Reload the daemon configuration & restart and enable etcd service 
```

  #systemctl daemon-reload
  
  #systemctl enable etcd && systemctl start etcd
  
  
Verify that the cluster is up and running.

# ETCDCTL_API=3 etcdctl member list

```

### Create kubeadm configuration file on kubemas1 server only.
```
# vi kubeadm-config.yaml


apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
kubernetesVersion: stable
controlPlaneEndpoint: 192.168.56.145:6443
etcd:
    external:
        endpoints:
        - https://192.168.56.143:2379
        - https://192.168.56.144:2379
        - https://192.168.56.142:2379
        caFile: /etc/etcd/ca.pem
        certFile: /etc/etcd/kubernetes.pem
        keyFile: /etc/etcd/kubernetes-key.pem
        
  ```      
 ### Initialize the machine as a master node.
```
    # kubeadm init --config=kubeadm-config.yml
 ```
### Copy Certificate file on Master1
 ```   
[root@kubemas1 pki]# scp -r /etc/kubernetes/pki root@192.168.56.142:/etc/kubernetes
root@192.168.56.142's password:
ca.key                                                                                                                           100% 1679     2.0MB/s   00:00
ca.crt                                                                                                                           100% 1025   958.9KB/s   00:00
apiserver.key                                                                                                                    100% 1675     1.6MB/s   00:00
apiserver.crt                                                                                                                    100% 1245     1.3MB/s   00:00
apiserver-kubelet-client.key                                                                                                     100% 1679     1.5MB/s   00:00
apiserver-kubelet-client.crt                                                                                                     100% 1099     1.4MB/s   00:00
front-proxy-ca.key                                                                                                               100% 1675     2.1MB/s   00:00
front-proxy-ca.crt                                                                                                               100% 1038     1.0MB/s   00:00
front-proxy-client.key                                                                                                           100% 1679     1.9MB/s   00:00
front-proxy-client.crt                                                                                                           100% 1058   999.7KB/s   00:00
sa.key                                                                                                                           100% 1675     1.8MB/s   00:00
sa.pub                                                                                                                           100%  451   471.0KB/s   00:00
```
### Copy Certificate file on Master2
```
[root@kubemas1 pki]# scp -r /etc/kubernetes/pki root@192.168.56.144:/etc/kubernetes
root@192.168.56.144's password:
ca.key                                                                                                                           100% 1679     1.3MB/s   00:00
ca.crt                                                                                                                           100% 1025   878.8KB/s   00:00
apiserver.key                                                                                                                    100% 1675     1.6MB/s   00:00
apiserver.crt                                                                                                                    100% 1245     1.1MB/s   00:00
apiserver-kubelet-client.key                                                                                                     100% 1679     1.9MB/s   00:00
apiserver-kubelet-client.crt                                                                                                     100% 1099     1.1MB/s   00:00
front-proxy-ca.key                                                                                                               100% 1675     1.7MB/s   00:00
front-proxy-ca.crt                                                                                                               100% 1038     1.0MB/s   00:00
front-proxy-client.key                                                                                                           100% 1679     1.5MB/s   00:00
front-proxy-client.crt                                                                                                           100% 1058     1.0MB/s   00:00
sa.key                                                                                                                           100% 1675     1.5MB/s   00:00
sa.pub                                    

```
You can login to 192.168.56.144 & 192.168.56.142 and remove the apiserver.key and apiserver.crt  file from  /etc/kubernetes/pki/

```
   #kubeadm init --config kubeadm-config.yaml
``` 

### Your Kubernetes control-plane has initialized successfully!
```
To start using your cluster, you need to run the following as a regular user on all kubemas servers.

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
### You should now deploy a pod network to the cluster.
```
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/
```
### You can now join any number of control-plane nodes by copying certificate authorities and service account keys on each node and then running the following as root:
~~~
  kubeadm join 192.168.56.145:6443 --token d5gtul.9su0rkdxsmj1zlux \
    --discovery-token-ca-cert-hash sha256:fe18f7a07c0f23734d14bdc160d9f2fd6bffcd52b5c1555a18fe2f4662590dce \
    --control-plane
~~~
### Then you can join any number of worker nodes by running the following on each as root:
~~~
kubeadm join 192.168.56.145:6443 --token d5gtul.9su0rkdxsmj1zlux \
    --discovery-token-ca-cert-hash sha256:fe18f7a07c0f23734d14bdc160d9f2fd6bffcd52b5c1555a18fe2f4662590dce
https://cloudformsblog.redhat.com/2018/03/22/cloudforms-on-aws-part-1-series/
~~~
### Test Cluster
```

 #kubectl --kubeconfig /etc/kubernetes/admin.conf get nodes
 
NAME                STATUS   ROLES    AGE   VERSION
kubemas1.anbu.com   Ready    master   23d   v1.17.0
kubemas2.anbu.com   Ready    master   23d   v1.17.0
kubemas3.anbu.com   Ready    master   23d   v1.17.0

```

### Apply the CNI plugin of your choice. The given example is for Weave Net , execute below command on all kubemas servers. use either one.
```
  Weave net
  
 #kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
 
  Calico Network
  
 #kubectl apply -f https://docs.projectcalico.org/v3.8/manifests/calico.yaml

```
