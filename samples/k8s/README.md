# suzieq on k8s

Why?

I don't know k8s well, it's just the compute hammer I have to work with in my current environment. That said, I do happen to like k8s as a hammer, and luckily suzieq seems to do a pretty good job of being a k8s nail.  

This document started as a weekend project to walk thru getting suzieq on k8s in my private lab for no other reason than it seemed fun.  The more I worked with it on k8s the more I liked the idea, and the more I liked the idea of deploying in my real physical lab.

The enviroment I have for my network management software is effectively a managed k8s enviroment.  It's what I have access to for "managed compute", and it also has access to the managment plane of my network gear.  So if I want to go to the real lab or prod, I have a vested interest in being able to deploy suzieq on k8s.

I also like the idea of suzieq as a api endpoint for network data.  Putting aside for a moment that I like software, as a network engineer sometimes I would rather just have the data without the tinkering with the software.   SuzieQ is quick and simple to get up and running, I don't want to complicate it, part of the appeal is the simplicity.

I like the idea of connecting to a suzieQ deployment versus running the pollers directly on my workstation.  Playing with that as a puzzle problem appeals to me as much as suzieQ as a product appeals to me as a network engineer.  I like the idea of both the journey and the destination both as it were.

## who is this for, why share it

mostly me, so I can do it again if I need to, but I don't imagine I'm gonna be the first person that thinks of running SuzieQ on k8s. 

## the basic target

What's needed

- For persisting data I will use a k8s [persistent volume](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- For managing config I will store configs as a k8s [config map](https://kubernetes.io/docs/concepts/configuration/configmap/)
- For storing secrets like certificates, keys or passwords I will use k8s [secrets](https://kubernetes.io/docs/concepts/configuration/secret/)

I will run 3 containers in a single k8s pod, one container for each of the following processes that make up suzieq.  

1. `sq-poller`
2. `sq-rest-server.py`
3. `suzieq-gui`

For initial discovery it keeps things simple having them share a pod.

I will use a persistent volume to store the data, I have no real idea what I'd need in temrs of size so will default to 5Gi of storage as a placeholder.  

/Stretch goal/

Use ReadWriteMany volumes to have the `sq-rest-server.py` and `sq-poller` containers run on different pods accessing the same parquet files.

## Generating the certificates

I need a cert for the demo deployment, a self signed cert will do.  In production the certificate would be signed by my internal CA.  I use tls secrets in k8s today where I need to expose certificates to containers as opposed to building the certificates into the continers.

We can use openssl to generate the certificate with the following command 

```
[ec2-user@ip-10-255-0-79 suzieq]$ openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 -nodes
Generating a 4096 bit RSA private key
....................++
........................................++
writing new private key to 'key.pem'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:US
State or Province Name (full name) []:NC
Locality Name (eg, city) [Default City]:Charlotte    
Organization Name (eg, company) [Default Company Ltd]:Hyperbolic Hyposcaler Industries
Organizational Unit Name (eg, section) []:network
Common Name (eg, your name or your server's hostname) []:suzieq.network.hyposcaler.net
Email Address []:devnull@hyposcaler.io
```

Once I have the certs I'll store them as secrets on the k8s server.  For the lab I'll just use kubectl for that.  The following command will create the secret from the cert.pem and key.pem created by openssl earlier.  


```
kubectl create secret tls suzieq-tls-secret \
  --cert=cert.pem \
  --key=key.pem
```

Regardless of how the cert is created, a k8s TLS secret is typically the interface for the deployment to access it.  There are a variety of methods to manage secrets securely on k8, they tend to have use of k8s secrets or similar mounts, in common as the delivery method for exposing the secrets to containers.  

In practice it creates a good approximation of what suzieq would see in prod.

Once I have created the k8s TLS secret, the key and cert files from openssl can be deleted.  

## Generating the configMaps

There are two config related files that we want to use with suzieQ

the first is `suzieq-cfg.yml`. It is the main suzieq config file, it holds pointers to the assorted paths used by suzieq. It also holds references to the the API_KEY used by the Rest server for auth, and pointers to the locations of the TLS certificates. 

The following is a typical `suzieq-cfg.yml` file

```yaml
data-directory: /suzieq/parquet
service-directory: /suzieq/config
schema-directory: /suzieq/config/schema
temp-directory: /tmp/
# kafka-servers: localhost:9093
logging-level: WARNING
period: 15
API_KEY: 496157e6e869ef7f3d6ecb24a6f6d847b224ee4f
rest_certfile: /suzieq/tls/cert.pem
rest_keyfile: /suzieq/tls/key.pem
```

In fact it's the default config, with one small change:  I have added a subdirectory under /suzieq/ to hold the certificates.  I like having an empty directory to mount the TLS secrets into.

The other config we need to keep track of is the inventory.  The inventory is also a yaml based file for this demo we have two, but we are going to store them in the same config map



```yaml
- namespace: eos
  hosts:
    - url: https://neteng:arista123@10.255.0.10 devtype=eos
```

```yaml
- namespace: junos
  hosts:
    - url: ssh://neteng:juniper123@10.255.0.11 devtype=junos-mx
```

Note I'm storing the paswords here, for a small private virtual lab that gets spun up and down as needed, I'm happy to do that.  Not something I would want to do in production, and in but in my specific production instance, I need a way to support password based auth.

If I were using SSH keys, I would use a k8s ssh key secret and mount them into the suzieq directory under a keys subfolder.  You specify a path to the key in the suzieq poller config, so it would work well for k8s SSH secret.

Either way I mash these up into the same yaml file named configmap.yml.  

The resulting file looks like the following.

```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: suzieq-inventory
  namespace: suzieq
data:
  eos.yml: |
    - namespace: eos
      hosts:
        - url: https://neteng:arista123@10.255.0.10 devtype=eos
  junos.yml: |
    - namespace: junos
      hosts:
        - url: ssh://neteng:juniper123@10.255.0.11 devtype=junos-mx
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: suzieq-config
  namespace: suzieq
data:
  suzieq-cfg.yml: |
    data-directory: /suzieq/parquet
    service-directory: /suzieq/config
    schema-directory: /suzieq/config/schema
    temp-directory: /tmp/
    # kafka-servers: localhost:9093
    logging-level: WARNING
    period: 15
    API_KEY: 496157e6e869ef7f3d6ecb24a6f6d847b224ee4f
    rest_certfile: /suzieq/tls/tls.crt
    rest_keyfile: /suzieq/tls/tls.key
```

use kubectl to create the two configmaps on the k8s cluster by running `kubectl create -f samples/k8s/configmap.yml` from the root of the repo

once created use kubectl again to verify they exist

```cli
[dev-suzieq]$ kubectl -n suzieq describe configmap
NAME               DATA   AGE
suzieq-config      1      25h
suzieq-inventory   2      20h
[dev-suzieq]$
```

After they are created, they can later be edited inplace, or overwritten with new 

Another Idea that appeals to me is using an init container that dynamically creates the poller configmap.

## Persistent Volumes

For the persistent volume, create a file named pvc.yml with the following contents

```yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  namespace: suzieq
  name: parquet-pv-claim
spec:
  accessModes:
    - ReadWriteOnce     # will only have 1 pod so read 
  resources:            # write once for now is fine
    requests:
      storage: 5Gi      # 5Gi completely arbitrary value. 
```

deploy it to the cluster via `kbuectl create -f samples/k8s/pvc.yml`, this will cause the cluster to set aside 5Gi of storage, it's lifecyle is not tied to the pod.  The pods are ephemeral, the storage can persist across the life of many pods. 

## the deployment

with the volumes, secrets, and storage lined up on the k8s cluster, all that remains is the deployment for the pod.

use a k8s [deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) to deploy suzieq.

The docuementation does a pretty good job of explaining what a deployment is. Refer to the k8s deployment documentation for more details. 

This deployment will ensure that there are always 3 containers running.  The containers only really have 3 differences between them

- the command executed at start
- the volumes they have access to
- the resource memory and cpu requests/limits

### command executed

There 3 containers, one for each of the the 3 processes/apps that make up suzie-q.  

1. sq-poller
2. sq-rest-server.py
3. suzieq-gui


sq-rest-server.py and suzieq-gui are pretty straight forward, in this deployment they don't need any addtional args so thier "commands" are going to be `command: ['sq-rest-server.py']` and `command: ['suzieq-gui']` respectively.

For poller we need to pass in some addtional args 

-k to disable host key checking and then arguments for two seperate inventory files

The final command looks like this 

`command: ['sq-poller', '-k', '-D', '/suzieq/inventory/eos.yml', '-D', '/suzieq/inventory/junos.yml']`


Note: I don't know that I need to use two different files here, but I seem to recall having had issues with doing two namespaces in the same inventory file.  I don't mind the need for two files, but would prefer being able to point the poller at a directory if multiple files are required for multiple namespaces.

### volumes they have access to

I gave all containers access to all volumes except for the and poller config.  I don't really know what makes sense here, my instinct was to just keep the poller config off of the two containers that are extneral facing.

### the resource memory and cpu requests/limits

I don't have a good idea what makes sense for thse, I picked the values mostly just to try an keep the resources for the pod under what I know my nodes in my private k8s clusters have (t3.mediums).

Until I get into the real lab I won't really have a feel for what practical values here might be.

```yaml
  resources:
    requests:
      memory: "256Mi"
      cpu: "200m"
    limits:
      memory: "500Mi"
      cpu: "500m"   
```



## The whole deployment file 

```yml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: suzieq
  name: suzieq
  labels:
    app: suzieq
spec:
  replicas: 1            
  selector:
    matchLabels:
      app: suzieq          
  template:
    metadata:
      labels:
        app: suzieq
    spec:
      volumes:                
          - name: suzieq-tls
            secret:
              secretName: suzieq-tls-secret
          - name: parquet-volume
            persistentVolumeClaim:
              claimName: parquet-pv-claim
          - name: config-volume
            configMap:
              name: suzieq-config
          - name: inventory-volume
            configMap:
              name: suzieq-inventory
      containers:
      - name: sq-rest-server
        image: hyposcaler/suzieq:wip
        resources:
          requests:
            memory: "256Mi"
            cpu: "200m"
          limits:
            memory: "500Mi"
            cpu: "500m"        
        imagePullPolicy: IfNotPresent
        volumeMounts:
          - name: suzieq-tls
            mountPath: "/suzieq/tls/"
            readOnly: true
          - name: parquet-volume
            mountPath: "/suzieq/parquet"
          - name: config-volume
            mountPath: /root/.suzieq/suzieq-cfg.yml 
            subPath: suzieq-cfg.yml 
        command: ['sq-rest-server.py']
        
      - name: sq-poller
        image: hyposcaler/suzieq:wip
        resources:
          requests:
            memory: "256Mi"
            cpu: "200m"
          limits:
            memory: "500Mi"
            cpu: "500m"        
        imagePullPolicy: IfNotPresent
        volumeMounts:
          - name: suzieq-tls
            mountPath: "/suzieq/tls/"
            readOnly: true
          - name: parquet-volume
            mountPath: "/suzieq/parquet"
          - name: config-volume
            mountPath: /root/.suzieq/suzieq-cfg.yml
            subPath: suzieq-cfg.yml
          - name: inventory-volume
            mountPath: /suzieq/inventory/eos.yml
            subPath: eos.yml
          - name: inventory-volume
            mountPath: /suzieq/inventory/junos.yml
            subPath: junos.yml
        command: ['sq-poller', '-k', '-D', '/suzieq/inventory/eos.yml', '-D', '/suzieq/inventory/junos.yml']
 
      - name: suzieq-gui
        image: hyposcaler/suzieq:wip
        resources:
          requests:
            memory: "128Mi"
            cpu: "125m"
          limits:
            memory: "250Mi"
            cpu: "250m"        
        imagePullPolicy: IfNotPresent
        volumeMounts:
          - name: suzieq-tls
            mountPath: "/suzieq/tls/"
            readOnly: true
          - name: parquet-volume
            mountPath: "/suzieq/parquet"
          - name: config-volume
            mountPath: /root/.suzieq/suzieq-cfg.yml
            subPath: suzieq-cfg.yml          
        command: ['suzieq-gui']
      imagePullSecrets:
      - name: regcred
```

deploy with `kubectl apply -f samples/k8s/deployment.yml`