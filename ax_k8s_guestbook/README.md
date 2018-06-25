# Deploying Appswitch enabled PHP Guestbook application with redis
This example demonstrates you how to build and deploy a multi-tier kubernetes web application with Appwitch.
This simple guestbook application consists of following components:

### Redis master
This application uses an open-source distributes, in-memory key-value database for storing its data. Application’s writes would be processed by redis master and reads served by single (multiple) redis slave instance.

This following manifest file (`redis-master-deployment.yaml`) specifies a Deployment controller that runs a single replica Redis master Pod along with the redis-master service which would be used by the guestbook application to communicates to the redis master.

``` yaml
apiVersion: v1
kind: Service
metadata:
  name: redis-master
  labels:
    app: redis
    tier: backend
    role: master
spec:
  ports:
  - port: 6379
    targetPort: 6379
  selector:
    app: redis
    tier: backend
    role: master
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: redis-master
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: redis
        role: master
        tier: backend
    spec:
      containers:
      - name: master
        image: k8s.gcr.io/redis:e2e 
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
        ports:
        - containerPort: 6379
---
```

### Redis slave
With the help of redis worker you can make redis database highly available. It also helps in distributing the I/O load on the redis DB. All the read operations are served by slave instance.

Below mentioned k8s resource file (`redis-slave-deployment.yaml`) describing a Deployment for the Redis worker pods along with redis-slave service which helps application frontend in discovering slave.

``` yaml
apiVersion: v1
kind: Service
metadata:
  name: redis-slave
  labels:
    app: redis
    tier: backend
    role: slave
spec:
  ports:
  - port: 6379
  selector:
    app: redis
    tier: backend
    role: slave
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: redis-slave
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: redis
        role: slave
        tier: backend
    spec:
      containers:
      - name: slave
        image: gcr.io/google_samples/gb-redisslave:v1
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
        env:
        - name: GET_HOSTS_FROM
          value: dns
          # If your cluster config does not include a dns service, then to
          # instead access an environment variable to find the master
          # service's host, comment out the 'value: dns' line above, and
          # uncomment the line below:
          # value: env
        ports:
        - containerPort: 6379
---
```

### Web Frontend
It is a simple PHP application which served the HTTP requests. It is configured to communicate with redis master and slave services deepening upon the type of request (read or write).  

Take a look at the `frontend-deployment.yaml` manifest file describing the Deployment for the guestbook web server. It also exposes a NodePort service which would helps in accessing it from outside the kubernetes cluster.

```YAML
apiVersion: v1
kind: Service
metadata:
  name: frontend
  labels:
    app: guestbook
    tier: frontend
spec:
  type: NodePort
  ports:
  - port: 80
    nodePort: 43000
  selector:
    app: guestbook
    tier: frontend
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: guestbook
        tier: frontend
    spec:
      containers:
      - name: php-redis
        image: gcr.io/google-samples/gb-frontend:v4
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
        env:
        - name: GET_HOSTS_FROM
          value: dns
          # If your cluster config does not include a dns service, then to
          # instead access environment variables to find service host
          # info, comment out the 'value: dns' line above, and uncomment the
          # line below:
          # value: env
        ports:
        - containerPort: 80
---
```

## Pre-requisites
This post considered that user have already setup atleast one (or two) node kubernates cluster setup with Appswitch daemon set in running on it.

## Deploy Guestbook application
The preliminary step for each and every deployment with 'appswtch' is to pass the kubernates resoucrece file to 'axinjector'. This tool inject 'ax' dependencies into it and make it configurable under 'ax'.

### Step 1: Deploy the redis master
*	Run the following command to inject ax into the Redis master resource file
```
axinjector inject -i redis-master-deployment.yaml -o ax-redis-master-deployment.yaml
```
* Create the Redis Master Deployment from the ax_redis-master-deployment.yaml file:
```
kubectl create -f ax-redis-master-deployment.yaml
```
* Alernatively both the above mentined stpes can be clubbed into the following one command:
```
Kubectl create –f < (axinjector inject -i redis-master-deployment.yaml)
```

Verify that the Redis master Pod is running:

```
kubectl get pods
NAME                            READY     STATUS    RESTARTS   AGE
redis-master-5d54cb7954-r6kp9   1/1       Running   0          5s
```

`axinjector` tool removes k8s services during parsing and make sure that corresponding `ax` virtual services would get create during deployment of pod. We can verify the same with the help of following command:
```
ax get vservices

     VSNAME     VSTYPE      VSIP         VSPORTS     VSBACKENDIPS                 VSAPPIDS
---------------------------------------------------------------------------------------------------------
  redis-master          172.28.17.231  []            []            [cd2755b2-db61-4093-a67a-65c1af8f3b33]
```

### Step-2: Deploy the redis slave
Create the Redis worker Deployment and service:
```
axinjector inject -i redis-slave-deployment.yaml -o ax-redis-slave-deployment.yaml

Kubectl create –f ax-redis-slave-deployment.yaml

kubectl get pods
NAME                            READY     STATUS    RESTARTS   AGE
redis-master-5d54cb7954-r6kp9   1/1       Running   0          50s
redis-slave-9dfc87d98-fc956     1/1       Running   0          5s
```

Verify `ax` virtual service for redis-slave:
```
ax get vservices

     VSNAME     VSTYPE      VSIP         VSPORTS     VSBACKENDIPS                 VSAPPIDS
---------------------------------------------------------------------------------------------------------
  redis-slave           172.29.210.71  []            []            [247f9ab9-47e9-4ae6-ba16-0fc009f76d7c]
```

### Step-3: Create the frontend pod
Configuring  the frontend Service as a Nodeport so that it should be externally visible. Client can request the Service from outside the container cluster.

To create the guestbook web frontend Deployment, run.
```
Kubectl create –f < (axinjector inject -i frontend-deployment.yaml)
```

Verify that all the deployments are up and running fine:
```
kubectl get pods
NAME                            READY     STATUS    RESTARTS   AGE
frontend-6475fc8688-ndpzw       1/1       Running   0          5s
redis-master-5d54cb7954-r6kp9   1/1       Running   0          60s
redis-slave-9dfc87d98-fc956     1/1       Running   0          45s
```
 
Frontend service is type of Nodeport and it exposes port `43000` to external world to acces the application. We can verify the same 
in `ax` command output that port `43000` is present in `VSPORTS` column.
```
ax get vservices

     VSNAME     VSTYPE      VSIP         VSPORTS     VSBACKENDIPS                 VSAPPIDS
---------------------------------------------------------------------------------------------------------
    frontend              172.16.19.249  [{80 43000}]  []            [769600fa-b684-44ec-be41-50c50fce4c61]
```

Alternatively we can specify all the deployment in one manifest file and deploy them in one shot:-
```
kubectl create -f <(axinjector inject -i ax-guestbook-all-in-one.yaml)
```

## Accessing the guestbook website
For acccesing the Guestbook website on the web, find out the host IP address where the frontend pod got deployed. Once we have the IP address website can be accessed on the following url: (Consindering host IP is 209.205.221.41)

```
http:\\209.205.221.41:43000
```
