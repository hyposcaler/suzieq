# suzieq on k8

## create TLS cert

openssl to generate a self signed cert, for production we wouldn't use self signed, but for demo purposes self signed is sufficent.

```cli
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 -nodes
```

## create name space 

I use a namespace to hold all resources that are releated to the deployment.

can create it via 

```cli
kubectl create namespace suzieq
```

## creating TLS secret

I stash the cert in a tls secret named suzieq-tls-secret on the cluster, this avoids having to have the certificate in the container itself.  Should make rotating out the certificate simpler by not tying it directly to the container, or process to build the container.

```
kubectl -n suzieq create secret tls suzieq-tls-secret \
  --cert=./cert.pem \
  --key=./key.pem
```

## create a pv and pvc

Need to create a persistent volume, the value I used for storage is here arbitrary, I don't have feel for what an adequate ammount of space would be, so just going with 5gi. 5gi is overkill for the sample data that is built into the docker container 

The persistent volume will give a means to persist our parquet tables between deployments or testing

to create the pv, create a file named `pv.yaml` with the following contents

```yaml
---
apiVersion: v1
kind: PersistentVolume
metadata:
  namespace: suzieq
  name: parquet-volume
  labels:
    type: local
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/suzieq/parquet"
```

can be deployed with  `kubectl apply -f ./pv.yaml`


### create pvc 

We will also need a pvc

create a file named `pvc.yaml` with the following contents.

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

deploy the pvc with `kubectl apply -f ./pvc.yaml`

For testing we'll need to populate this with data, either via the poller (as we would in production) or we can use kubectl later to copy some parquet files up.  will come back to this later.

## Modifications to sq-rest-server.py

sq-rest.server.py has hardcoded values for the tls certificate locations.  It specifically points at `/.suzieq/key.pem`.   In my example I want to expose the certificates to the container, not have them live in the container.  To do this I planned on using a TLS secret as a volumemount.   The biggest blocker here is that we lose access to any existing files in the mount point.  /root/.suzieq is the default location for the config.  To avoid losing access to other files that are or will be in `/root/.suzieq`, we need another location to mount the key secret into.  I used `./suzieq/tls` any directory works

Configurable cert location would help address this, either as a command line option to use in the entrypoint, or expressed as a ENV varaible.

ENV seems quicker/easier for me, and doesn't require much change, and is't really lost effort if it ends up being a horrible choice.  So I modified `check_for_cert_files()` to just look look for the paths in a env variable rather than building the path from the HOME env + a string.  Error string still needs updating.

```python
def check_for_cert_files():
    if not os.path.isfile(os.getenv("SQUZIEQ_REST_TLS_KEY")) or not \
            os.path.isfile(os.getenv("SQUZIEQ_REST_TLS_CERT")):
        print("ERROR: Missing cert files in ~/.suzieq/tls") 
        sys.exit(1)
```

need to rememober to set `SQUZIEQ_TLS_KEY=/.suzieq/tls/tls.key` and `SQUZIEQ_TLS_CERT=/.suzieq/tls/tls.crt` will do that later in modifications to the the deployment.

Also have to update the call to `uvicorn.run()` in sq-rest-server.py to the following

```python
uvicorn.run(app, host="0.0.0.0", port=8000,
                log_level=get_configured_log_level(),
                log_config=get_log_config(),
                ssl_keyfile=os.getenv("SQUZIEQ_REST_TLS_KEY"),
                ssl_certfile=os.getenv("SQUZIEQ_REST_TLS_CERT"))
```

Rest server also currently logs to file, some means to log to STDOUT instead.  This makes integrating with existing systems for collecting logs from STDOUT possible, as well as easier k8 tshooting with "kubectl log"

## Dockerfile Changes

Basically replace this

```dockerfile
COPY ./build/key.pem /root/.suzieq/
COPY ./build/cert.pem /root/.suzieq/
```

with

```dockerfile
# create a mount point for the certificates
RUN mkdir /root/.suzieq/tls
```

just creating a sub dir named `tls` under `/root/.suzieq/` specifically to hold the tls certs.

## The deployment

One pod with two containers. One container is for the REST server which we just have run `sq-rest-server.py`. and the other will become the poller, but for now we just have a placeholder sleep command running in the second container.

For my production enviroment the container registry would be internal, but I'm doing this in my private lab so just using a private repo and pullsecrets in this deployment.  

Other wise, for a demo, this deployment spec meets the needs.  No loadbalancers nothign else fancy, goal right now is replicating the default docker container behavior in k8s before the next step of tweaking and configuring the poller.

the deployment spec is a file named deployment.yaml with the following contents:

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: suzieq
  name: suzieq-deployment
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
      containers:
      - name: suzieq-rest
        image: hyposcaler/suzieq:wip
        env:
          - name: SQUZIEQ_REST_TLS_CERT
            value: "/.suzieq/tls/tls.crt"
          - name: SQUZIEQ_REST_TLS_KEY
            value: "/.suzieq/tls/tls.key"
        imagePullPolicy: Always
        volumeMounts:
          - name: suzieq-tls
            mountPath: "/root/.suzieq/tls/"
            readOnly: true
          - name: parquet-volume
            mountPath: "/suzieq/parquet"
        command: ['sq-rest-server.py']
      - name: suzieq-poller
        image: hyposcaler/suzieq:wip
        imagePullPolicy: Always
        volumeMounts:
          - name: suzieq-tls
            mountPath: "/root/.suzieq/tls/"
            readOnly: true
          - name: parquet-volume
            mountPath: "/suzieq/parquet"        
        command: ['sh', '-c', 'echo Container 1 is Running ; sleep 3600']
      imagePullSecrets:
      - name: regcred
```

deploy it via `kubectl apply -f ./deployment.yaml`

## first run

```cli
[dev-suzieq]$ kubectl apply -f ./samples/k8/deployment.yaml 
deployment.apps/suzieq-deployment created
[dev-suzieq]$ kubectl -n suzieq get all 
NAME                                    READY   STATUS    RESTARTS   AGE
pod/suzieq-deployment-c6ffbb487-jkgvs   2/2     Running   0          14s

NAME                                READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/suzieq-deployment   1/1     1            1           15s

NAME                                          DESIRED   CURRENT   READY   AGE
replicaset.apps/suzieq-deployment-c6ffbb487   1         1         1       14s
[dev-suzieq]$ kubectl -n suzieq describe deployment.apps/suzieq-deployment
Name:                   suzieq-deployment
Namespace:              suzieq
CreationTimestamp:      Sat, 09 Jan 2021 20:07:20 +0000
Labels:                 app=suzieq
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               app=suzieq
Replicas:               1 desired | 1 updated | 1 total | 1 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=suzieq
  Containers:
   suzieq-rest:
    Image:      hyposcaler/suzieq:wip
    Port:       <none>
    Host Port:  <none>
    Command:
      sq-rest-server.py
    Environment:
      SQUZIEQ_REST_TLS_CERT:  /.suzieq/tls/tls.crt
      SQUZIEQ_REST_TLS_KEY:   /.suzieq/tls/tls.key
    Mounts:
      /root/.suzieq/tls/ from suzieq-tls (ro)
      /suzieq/parquet from parquet-volume (rw)
   suzieq-poller:
    Image:      hyposcaler/suzieq:wip
    Port:       <none>
    Host Port:  <none>
    Command:
      sh
      -c
      echo Container 1 is Running ; sleep 3600
    Environment:  <none>
    Mounts:
      /root/.suzieq/tls/ from suzieq-tls (ro)
      /suzieq/parquet from parquet-volume (rw)
  Volumes:
   suzieq-tls:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  suzieq-tls-secret
    Optional:    false
   parquet-volume:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  parquet-pv-claim
    ReadOnly:   false
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   suzieq-deployment-c6ffbb487 (1/1 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  23s   deployment-controller  Scaled up replica set suzieq-deployment-c6ffbb487 to 1
[dev-suzieq]$ 
```
still need to put some data on our persistent volume, can use `kubectl cp` 

I combined all of the `test/data` parquets into a directory called combined.

```cli
[dev-suzieq]$ cd ../combined/
[dev-combined]$ ls -la
total 0
drwxrwxr-x 20 ec2-user ec2-user 252 Jan  9 16:40 .
drwx------ 14 ec2-user ec2-user 299 Jan  9 19:32 ..
drwxrwxr-x  3 ec2-user ec2-user  24 Jan  9 16:07 arpnd
drwxrwxr-x  3 ec2-user ec2-user  24 Jan  9 16:07 bgp
drwxrwxr-x  3 ec2-user ec2-user  24 Jan  9 16:07 device
drwxrwxr-x  3 ec2-user ec2-user  24 Jan  9 16:40 evpnVni
drwxrwxr-x  3 ec2-user ec2-user  24 Jan  9 16:07 fs
drwxrwxr-x  3 ec2-user ec2-user  24 Jan  9 16:07 ifCounters
drwxrwxr-x  3 ec2-user ec2-user  24 Jan  9 16:07 interfaces
drwxrwxr-x  3 ec2-user ec2-user  24 Jan  9 16:07 lldp
drwxrwxr-x  3 ec2-user ec2-user  24 Jan  9 16:07 macs
drwxrwxr-x  3 ec2-user ec2-user  24 Jan  9 16:07 mlag
drwxrwxr-x  3 ec2-user ec2-user  24 Jan  9 16:08 ospfIf
drwxrwxr-x  3 ec2-user ec2-user  24 Jan  9 16:08 ospfNbr
drwxrwxr-x  3 ec2-user ec2-user  24 Jan  9 16:07 routes
drwxrwxr-x  4 ec2-user ec2-user  42 Jan  9 16:07 sqPoller
drwxrwxr-x  3 ec2-user ec2-user  24 Jan  9 16:07 time
drwxrwxr-x  3 ec2-user ec2-user  24 Jan  9 16:07 topcpu
drwxrwxr-x  3 ec2-user ec2-user  24 Jan  9 16:07 topmem
drwxrwxr-x  3 ec2-user ec2-user  24 Jan  9 16:07 vlan
[dev-combined]$
```

copy them up

```cli
[dev-combined]$ kubectl cp ./ suzieq-deployment-c6ffbb487-jkgvs:/suzieq/parquet
Defaulting container name to suzieq-rest.
[dev-combined]$ 
```

and remote in

```cli
[dev-combined]$ kubectl -n suzieq exec -it pod/suzieq-deployment-c6ffbb487-jkgvs -- /bin/bash
Defaulting container name to suzieq-rest.
Use 'kubectl describe pod/suzieq-deployment-c6ffbb487-jkgvs -n suzieq' to see all of the containers in this pod.
root@suzieq-deployment-c6ffbb487-jkgvs:/suzieq# cd parquet/
root@suzieq-deployment-c6ffbb487-jkgvs:/suzieq/parquet# ls -la
total 8
drwxr-xr-x  3 root root 4096 Jan  9 20:11 .
drwxr-xr-x  1 root root   21 Jan  9 20:07 ..
drwxr-xr-x 20 root root 4096 Jan  9 20:11 parquet
root@suzieq-deployment-c6ffbb487-jkgvs:/suzieq/parquet# cd parquet/
root@suzieq-deployment-c6ffbb487-jkgvs:/suzieq/parquet/parquet# ls
arpnd  bgp  device  evpnVni  fs  ifCounters  interfaces  lldp  macs  mlag  ospfIf  ospfNbr  routes  sqPoller  time  topcpu  topmem  vlan
root@suzieq-deployment-c6ffbb487-jkgvs:/suzieq/parquet/parquet#
```

I have no idea why that happens, I'm asssuming it's a side effect of the volume mount but the fix is

```cli
root@suzieq-deployment-c6ffbb487-jkgvs:/suzieq/parquet/parquet# mv ./* ../
root@suzieq-deployment-c6ffbb487-jkgvs:/suzieq/parquet/parquet# cd ..
root@suzieq-deployment-c6ffbb487-jkgvs:/suzieq/parquet# rm -rf parquet/
root@suzieq-deployment-c6ffbb487-jkgvs:/suzieq/parquet# ls -la 
total 76
drwxr-xr-x 20 root root 4096 Jan  9 20:14 .
drwxr-xr-x  1 root root   21 Jan  9 20:07 ..
drwxr-xr-x  3 root root 4096 Jan  9 20:11 arpnd
drwxr-xr-x  3 root root 4096 Jan  9 20:11 bgp
drwxr-xr-x  3 root root 4096 Jan  9 20:11 device
drwxr-xr-x  3 root root 4096 Jan  9 20:11 evpnVni
drwxr-xr-x  3 root root 4096 Jan  9 20:11 fs
drwxr-xr-x  3 root root 4096 Jan  9 20:11 ifCounters
drwxr-xr-x  3 root root 4096 Jan  9 20:11 interfaces
drwxr-xr-x  3 root root 4096 Jan  9 20:11 lldp
drwxr-xr-x  3 root root 4096 Jan  9 20:11 macs
drwxr-xr-x  3 root root 4096 Jan  9 20:11 mlag
drwxr-xr-x  3 root root 4096 Jan  9 20:11 ospfIf
drwxr-xr-x  3 root root 4096 Jan  9 20:11 ospfNbr
drwxr-xr-x  3 root root 4096 Jan  9 20:11 routes
drwxr-xr-x  4 root root 4096 Jan  9 20:11 sqPoller
drwxr-xr-x  3 root root 4096 Jan  9 20:11 time
drwxr-xr-x  3 root root 4096 Jan  9 20:11 topcpu
drwxr-xr-x  3 root root 4096 Jan  9 20:11 topmem
drwxr-xr-x  3 root root 4096 Jan  9 20:11 vlan
root@suzieq-deployment-c6ffbb487-jkgvs:/suzieq/parquet# 
```

we only need to do it once, aftwards

```cli
root@suzieq-deployment-c6ffbb487-jkgvs:/suzieq/parquet# suzieq-cli
root> device unique columns=namespace
     namespace  count
0     dual-bgp     14
1    dual-evpn     14
2          eos     14
3        junos      6
4         nxos     14
5    ospf-ibgp     14
6  ospf-single     14
root> 






 
Suzieq  Verbose   OFF  Pager   OFF  Namespace     Hostname     StartTime     EndTime     Engine   pandas  Query Time   0.0871s                                                      
```

## the REST API

```cli
root@suzieq-deployment-c6ffbb487-jkgvs:/suzieq/parquet# cat ~/.suzieq/suzieq-cfg.yml 
data-directory: /suzieq/parquet
service-directory: /suzieq/config
schema-directory: /suzieq/config/schema
temp-directory: /tmp/
# kafka-servers: localhost:9093
logging-level: WARNING
period: 15
API_KEY: 496157e6e869ef7f3d6ecb24a6f6d847b224ee4f
root@suzieq-deployment-c6ffbb487-jkgvs:/suzieq/parquet# 
```

```cli
[dev-combined]$ kubectl port-forward deployment.apps/suzieq-deployment 8000:8000
Forwarding from 127.0.0.1:8000 -> 8000
Forwarding from [::1]:8000 -> 8000
```

```cli
[dev-suzieq]$ curl --insecure 'https://localhost:8000/api/v1/device/summarize?&access_token=496157e6e869ef7f3d6ecb24a6f6d847b224ee4f' | jq
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  1652  100  1652    0     0   8137      0 --:--:-- --:--:-- --:--:--  8137
{
  "ospf-single": {
    "deviceCnt": 14,
    "downDeviceCnt": 0,
    "vendorCnt": {
      "Cumulus": 9,
      "Ubuntu": 5
    },
    "modelCnt": {
      "vm": 14
    },
    "archCnt": {
      "x86-64": 14
    },
    "versionCnt": {
      "3.7.11": 9,
      "16.04.6 LTS": 5
    },
    "upTimeStat": [
      2508213,
      2673214,
      2511051
    ]
  },
  "dual-bgp": {
    "deviceCnt": 14,
    "downDeviceCnt": 0,
    "vendorCnt": {
      "Cumulus": 9,
      "Ubuntu": 5
    },
    "modelCnt": {
      "vm": 14
    },
    "archCnt": {
      "x86-64": 14
    },
    "versionCnt": {
      "3.7.11": 9,
      "16.04.6 LTS": 5
    },
    "upTimeStat": [
      93486,
      751774,
      749458
    ]
  },
  "eos": {
    "deviceCnt": 14,
    "downDeviceCnt": 0,
    "vendorCnt": {
      "Arista": 8,
      "Ubuntu": 5,
      "Juniper": 1
    },
    "modelCnt": {
      "vEOS": 8,
      "vm": 5,
      "vqfx-10000": 1
    },
    "archCnt": {
      "x86_64": 8,
      "x86-64": 5,
      "": 1
    },
    "versionCnt": {
      "4.23.5M": 8,
      "18.04.3 LTS": 5,
      "19.4R1.10": 1
    },
    "upTimeStat": [
      571368453,
      600089479,
      600082471
    ]
  },
  "junos": {
    "deviceCnt": 6,
    "downDeviceCnt": 0,
    "vendorCnt": {
      "Ubuntu": 4,
      "Juniper": 2
    },
    "modelCnt": {
      "vm": 4,
      "vqfx-10000": 2
    },
    "archCnt": {
      "x86-64": 4,
      "": 2
    },
    "versionCnt": {
      "16.04 LTS": 4,
      "19.4R1.10": 2
    },
    "upTimeStat": [
      455183,
      1482000,
      652090
    ]
  },
  "ospf-ibgp": {
    "deviceCnt": 14,
    "downDeviceCnt": 0,
    "vendorCnt": {
      "Cumulus": 9,
      "Ubuntu": 5
    },
    "modelCnt": {
      "vm": 14
    },
    "archCnt": {
      "x86-64": 14
    },
    "versionCnt": {
      "4.2.1": 9,
      "16.04.7 LTS": 5
    },
    "upTimeStat": [
      421461,
      6211682,
      423371
    ]
  },
  "dual-evpn": {
    "deviceCnt": 14,
    "downDeviceCnt": 0,
    "vendorCnt": {
      "Cumulus": 9,
      "Ubuntu": 5
    },
    "modelCnt": {
      "vm": 14
    },
    "archCnt": {
      "x86-64": 14
    },
    "versionCnt": {
      "4.2.1": 9,
      "16.04.7 LTS": 5
    },
    "upTimeStat": [
      426001,
      1532900,
      428007
    ]
  },
  "nxos": {
    "deviceCnt": 14,
    "downDeviceCnt": 0,
    "vendorCnt": {
      "Cisco": 8,
      "Ubuntu": 5,
      "Juniper": 1
    },
    "modelCnt": {
      "Nexus9000 C9300v Chassis": 8,
      "vm": 5,
      "vqfx-10000": 1
    },
    "archCnt": {
      "Intel Core Processor (Skylake, IBRS)": 8,
      "x86-64": 5,
      "": 1
    },
    "versionCnt": {
      "9.3(4)": 8,
      "18.04.3 LTS": 5,
      "19.4R1.10": 1
    },
    "upTimeStat": [
      92024175,
      2859166000,
      2859082898
    ]
  }
}
```

## next step

poller
