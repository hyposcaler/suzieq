# suzieq on k8s

My current environment is largely k8s, this is also the enviroment where I would ultimately run suzieq for production.  I don't know k8s well, it's just the compute hammer I have to work with in my current environment. That said, I do happen to like k8s as a hammer, and luckily suzieq seems to do a pretty good job of being a k8 nail.  

This document started as a weekend project to walk thru getting suzieq on k8 in my private lab for no other reason than it seemed fun.  

## who is this for

mostly me, but.... 

If you....

- already know k8s; this will speed things up and give you a good start for getting suzieq up and running on k8, this is a pretty straight forward starting point for thinking about how to scale it out.

- already know docker and docker compose but not k8s; this is more less just trying to do a "k8" version of the docker-compose in /samples/docker. I tried to stick to the same ideas aside from how I handle secrets in k8s, I tried to make this a direct translation from docker compose to k8.

- feel lost; don't worry. You don't need k8 to do cool things with suzeiq. Docker works just as well and Dinesh and Justin have provided a good starting point with the pre built containers. learn the docker stuph first.

## the basic target

- For persisting data I will use a k8 [persistent volume](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- For managing config I will store configs as k8 [config maps](https://kubernetes.io/docs/concepts/configuration/configmap/)
- For storing secrets like certificates, keys or passwords I will use k8 [secrets](https://kubernetes.io/docs/concepts/configuration/secret/)

I will run 3 containers in a single k8 pod, one container for each of the following processes that make up suzieq

1. `sq-poller`
2. `sq-rest-server.py`
3. `suzieq-gui`

I will use a persistent volume to store the data, I have no real idea what I'd need but we're gonna use 5Gi as a placeholder.  

## Generating the certificates

I need a cert for the demo deployment, a self signed cert will do.  In production the certificate would be signed by my internal CA.  I use tls secrets in k8s today where I need to expose certificate to containers as opposed to building the certificates into the continers.

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
Once I have the certs I'll store them as secrets on the k8 server.  For the lab I'll just use kubectl for that.  The following command will create the secret from the cert.pem and key.pem created by openssl earlier.  

```
kubectl create secret tls suzieq-tls-secret \
  --cert=cert.pem \
  --key=key.pem
```

In production things might work out differently but ultimately will have the same results; the public and private bits of the TLS cert are stored in a k8 tls secret.  

Once I hvae the TLS secret The key and cert files from openssl can be deleted.  I can get the cert/key back from the server if we need to.

## Generating the configMaps

There are two config related files that we want to use with suzieW

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

In fact it's the default config, with one small change:  I have added a subdirectory under /suzieq/ to hold the certificates.  I like having an empty directory to mount the TLS secrets into, also helps avoid accidentally wiping out the other contents of the directory when you mount the secrets.  

The other config we need to keep track of is the inventory.  The inventory is also a yaml based file for this demo we have two

```yaml
- namespace: eos
  hosts:
    - url: https://neteng:arista123@10.255.0.10 devtype=eos
```

```yaml
- namespace: junos
  hosts:
    - url: ssh://neteng:juniper123@192.168.123.252 devtype=junos-mx
```

Note I'm storing the paswords here, for a small private virtual lab that gets spun up and down as needed, not an issue.  Not something I would want to do in production, my situation is unique I can't use keys for all my hardware.  If I were using SSH keys, I would use a k8 ssh key secret and mount them into the suzieq directory.

Either way I mash these up into two seperate documents in the same yaml file named configmap.yml.  

Each document defines one configmap with one config map holding both inventory files, the other holding just the suzieq config

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

I use kubectl to create the two configmaps on the k8 cluster using the configmap.yml in /samples/k8s directory using the following command `kubectl create -f ./configmap.yml`

once created we can use kubectl again to verify they exist

```cli
[dev-suzieq]$ kubectl -n suzieq describe configmap
NAME               DATA   AGE
suzieq-config      1      25h
suzieq-inventory   2      20h
[dev-suzieq]$
```

## Persistent Volumes

For the persistent volume I can create a file named pvc.yml with the following contents

```yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  namespace: suzieq
  name: parquet-pv-claim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

and deploy it to the cluster via `kbuectl create -f samples/k8s/pvc.yml`, this will cause the cluster to set aside 5Gi of storage, it's lifecyle is not tied to the pod.  The pods are ephemeral, much like the secrets and hte configMaps the storage can persist across the life of many pods.

## the deployment

We have all our volumes, secrets and storage lined up on the k8 cluster, all that remains isthe deployment for the pod.


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
  strategy: 
    type: RollingUpdate
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
        # command: ['sh', '-c', 'echo "Hello, I am sq-rest-server.py" && sleep 3600']
        
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
        # command: ['sq-poller', '-D', '/suzieq/inventory/eos.yml']
        # command: ['sh', '-c', 'echo "Hello, I am sq-poller!" && sleep 3600']

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
        # command: ['sh', '-c', 'echo "Hello, I am suqzieq-gui" && sleep 3600']
      imagePullSecrets:
      - name: regcred
```


