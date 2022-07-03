
# Introduction

Kubernetes is a portable, extensible, open-source platform for managing containerized workloads and services. 

Kubernetes provides you with:

- **Service discovery and load balancing**: Kubernetes handles traffic between services, and that across different replicas of the same service.
- **Storage orchestration*: Kubernetes integrates with different storage services for data persistence (Local Storage, EBS Volumes, etc).
- **Automated rollouts and rollbacks*.
- **Secret and configuration management*. 

This article serves as a complete guide to deploying applications on Google Cloud Platform, considering that a Kubernetes Cluster is already created. 

# Environment

- A Google Kubernetes Engine (GKE) cluster of two nodes, with a jump host machine where we will execute all our kubectl commands.
- The [NK-microservices](https://github.com/nicolaselkhoury/nk-microservices-deployment) project as an example project. 

# Kubernetes Main Components

## Pods

A pod is the smallest execution unit in Kubernetes. A pod encapsulates one or more applications. Pods are ephemeral by nature, if a pod (or the node it executes on) fails, Kubernetes can automatically create a new replica of that pod to continue operations. Pods include one or more containers (such as Docker containers).

## Deployments

Deployments represent a set of multiple, identical Pods with no unique identities. A Deployment runs multiple replicas of your application and automatically replaces any instances that fail or become unresponsive. In this way, Deployments help ensure that one or more instances of your application are available to serve user requests. Deployments are managed by the Kubernetes Deployment controller.

Deployments use a Pod template, which contains a specification for its Pods. The Pod specification determines how each Pod should look like: what applications should run inside its containers, which volumes the Pods should mount, its labels, and more.

When a Deployment's Pod template is changed, new Pods are automatically created one at a time.

## Services

An abstract way to expose an application running on a set of Pods as a network service.
With Kubernetes you don't need to modify your application to use an unfamiliar service discovery mechanism. Kubernetes gives Pods their own **permanent** IP addresses and a single DNS name for a set of Pods, and can load-balance across them.

There are many service types:

- ClusterIP. Exposes a service which is only accessible from within the cluster.
- NodePort. Exposes a service via a static port on each node’s IP.
- LoadBalancer. Exposes the service via the cloud provider’s load balancer.
- ExternalName. Maps a service to a predefined externalName field by returning a value for the CNAME record.

## Ingress

An API object that manages external access to the services in a cluster, typically HTTP.
Ingress may provide load balancing, SSL termination and name-based virtual hosting.

## Resource Limits

Requests and limits are the mechanisms Kubernetes uses to control resources such as CPU and memory. Requests are what the container is guaranteed to get. If a container requests a resource, Kubernetes will only schedule it on a node that can give it that resource. Limits, on the other hand, make sure a container never goes above a certain value. The container is only allowed to go up to the limit, and then it is restricted.

## StatefulSet

StatefulSet is the workload API object used to manage stateful applications.

Manages the deployment and scaling of a set of Pods, and provides guarantees about the ordering and uniqueness of these Pods.

Like a Deployment, a StatefulSet manages Pods that are based on an identical container spec. Unlike a Deployment, a StatefulSet maintains a sticky identity for each of their Pods. These pods are created from the same spec, but are not interchangeable: each has a persistent identifier that it maintains across any rescheduling.

If you want to use storage volumes to provide persistence for your workload, you can use a StatefulSet as part of the solution. Although individual Pods in a StatefulSet are susceptible to failure, the persistent Pod identifiers make it easier to match existing volumes to the new Pods that replace any that have failed.

StatefulSets are valuable for applications that require one or more of the following.

- Stable, unique network identifiers.
- Stable, persistent storage.
- Ordered, graceful deployment and scaling.
- Ordered, automated rolling updates.

In the above, stable is synonymous with persistence across Pod (re)scheduling. If an application doesn't require any stable identifiers or ordered deployment, deletion, or scaling, you should deploy your application using a workload object that provides a set of stateless replicas. Deployment or ReplicaSet may be better suited to your stateless needs.

## Secrets

A Secret is an object that contains a small amount of sensitive data such as a password, a token, or a key. Such information might otherwise be put in a Pod specification or in a container image. Using a Secret means that you don't need to include confidential data in your application code.

Because Secrets can be created independently of the Pods that use them, there is less risk of the Secret (and its data) being exposed during the workflow of creating, viewing, and editing Pods. Kubernetes, and applications that run in your cluster, can also take additional precautions with Secrets, such as avoiding writing confidential data to nonvolatile storage.

Secrets are similar to ConfigMaps but are specifically intended to hold confidential data.

## Volumes

On-disk files in a container are ephemeral, which presents some problems for non-trivial applications when running in containers. One problem is the loss of files when a container crashes. The kubelet restarts the container but with a clean state. A second problem occurs when sharing files between containers running together in a Pod. The Kubernetes volume abstraction solves both of these problems.

## Probes
Probes are kubelet’s answer to the health checks, there are three handlers:

- ExecAction: Command execution check, if the command’s exit status is 0, it is considered a success.
- TCPSocketAction: TCP check to determine if the port is open, if open, it is considered a success.
- HTTPGetAction: HTTP check to determine if the status code is equal to or above 200 and below 400.

We have 3 types of probes:
- Readiness Probe
- Liveness Probe
- Startup Probe

We will dig deeper into the Liveness and Readiness probes only.

Each type of probe has common configurable fields:

- initialDelaySeconds: Probes start running after initialDelaySeconds after container is started (default: 0)
- periodSeconds: How often probe should run (default: 10)
- timeoutSeconds: Probe timeout (default: 1)
- successThreshold: Required number of successful probes to mark container healthy/ready (default: 1)
- failureThreshold: When a probe fails, it will try failureThreshold times before deeming unhealthy/not ready (default: 3)

These parameters need to be configured per your application’s spec.

### Readiness-Probe

Sometimes, applications are temporarily unable to serve traffic. For example, an application might need to load large data or configuration files during startup, or depend on external services after startup. In such cases, you don't want to kill the application, but you don't want to send it requests either. Kubernetes provides readiness probes to detect and mitigate these situations. A pod with containers reporting that they are not ready does not receive traffic through Kubernetes Services.

#### Exec Probe

Exec action has only one field, and that is command. Exit status of the command is checked, and the status of zero (0) means it is healthy, and other value means it is unhealthy.

```
        readinessProbe:
          initialDelaySeconds: 1
          periodSeconds: 5
          timeoutSeconds: 1
          successThreshold: 1
          failureThreshold: 1
          exec:
            command:
            - cat
            - /etc/nginx/nginx.conf
```

#### TCP Probe

We need to define host and port parameters, host parameter defaults to the cluster-internal pod IP.

```
        readinessProbe:
          initialDelaySeconds: 1
          periodSeconds: 5
          timeoutSeconds: 1
          successThreshold: 1
          failureThreshold: 1
          tcpSocket:
            host:
            port: 80
```


#### HTTP Probe

HTTP Probe has additional options to configure:

- host: Host/IP to connect to (default: pod IP)
- scheme: Scheme to use when making the request (default: HTTP)
- path: Path
- httpHeaders: An array of headers defined as header/value map
- port: Port to connect to

```
        readinessProbe:
          initialDelaySeconds: 1
          periodSeconds: 2
          timeoutSeconds: 1
          successThreshold: 1
          failureThreshold: 1
          httpGet:
            host:
            scheme: HTTP
            path: /
            httpHeaders:
            - name: Host
              value: myapplication1.com
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
```

#### Which one to choose

While the 3 options provide exactly the same result, it is always recommended to use HTTP probes in case the pods have a readiness API implemented, as using the TCP probe can expose a port that you might use when going to production.

### Liveness-Probe

Many applications running for long periods of time eventually transition to broken states, and cannot recover except by being restarted. Kubernetes provides liveness probes to detect and remedy such situations.
So basically, Liveness probes let Kubernetes know if your app is alive or dead. If you app is alive, then Kubernetes leaves it alone. If your app is dead, Kubernetes removes the Pod and starts a new one to replace it.

#### HTTP Probe

```
livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
        httpHeaders:
        - name: Custom-Header
          value: Awesome
      initialDelaySeconds: 3
      periodSeconds: 3
```

#### TCP Probe

```
livenessProbe:
      tcpSocket:
        port: 8080
      initialDelaySeconds: 15
      periodSeconds: 20
```

#### Exec Probe

```
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: liveness-exec
spec:
  containers:
  - name: liveness
    image: k8s.gcr.io/busybox
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 600
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5
```
# Useful commands

Here are some useful kubectl commands that we will be using:

- Deploy resource from a manifest file:
```
kubectl apply -f [FILE NAME]
```
- Getting information about the available cluster components:
```
kubectl get [service | pod | namespaces | deployments | statefulset | …] -n production 
```
- Display information about a specific created object:
```
kubectl describe [service | pod | namespaces | deployments | statefulset |…] [OBJECT NAME] -n production
```
- Delete a specific object:
```
kubectl delete [service | pod | namespaces | deployments | statefulset |…] [OBJECT NAME] -n production
```
- Get logs from a pod:
```
 kubectl logs [POD NAME] -n [NAMESPACE]
```
- Scaling a deployment 
```
kubectl scale deployment [DEPLOYMENT NAME] -n production --replicas=[NUMBER OF REPLICAS]

```

# Useful components
## Resource Limits

Requests and limits are the mechanisms Kubernetes uses to control resources such as CPU and memory. Requests are what the container is guaranteed to get. If a container requests a resource, Kubernetes will only schedule it on a node that can give it that resource. Limits, on the other hand, make sure a container never goes above a certain value. The container is only allowed to go up to the limit, and then it is restricted.

## Update Strategy

Kubernetes offers Deployment strategies that allow you to update in a variety of ways depending on the needs of the system. The three most common are:

- **Rolling update strategy:** Minimizes downtime at the cost of update speed.
- **Recreation Strategy:** Causes downtime but updates quickly.
- **Canary Strategy:** Quickly updates for a select few users with a full rollout later.

### Rolling update strategy

In this strategy, the Deployment selects a Pod with the old programming, deactivates it, and creates an updated Pod to replace it. The Deployment repeats this process until no outdated Pods remain.

The advantage of the rolling update strategy is that the update is applied Pod-by-Pod so the greater system can remain active.

There is a minor performance reduction during this update process because the system is consistently one active Pod short of the desired number of Pods. This is often much preferred to a full system deactivation.

The rolling update strategy is used as the **default update strategy**.

```
strategy:
    type: RollingUpdate
```

### Recreate update strategy

In this strategy, the Deployment selects all outdated Pods and deactivates them at once.

Once all old Pods are deactivated, the Deployment creates updated Pods for the entire system. The system is inoperable starting at the old Pod’s deactivation and ending once the final updated Pod is created.

```
strategy:
    type: Recreate
```

### Canary update strategy

In this strategy, the Deployment creates a few new Pods while keeping most Pods on the previous version, usually at a 1:4 ratio.

Most users still use the previous version, but a small subset unknowingly use the new version to act as testers.

```
strategy:
    rollingUpdate:
    maxSurge: 1
    maxUnavailable: 1
  minReadySeconds: 5
```

# Deployment Example

In this example, we will deploy the [nk-microservices project](). We will be using configuration files to create the pods. Each configuration file will be used with the command `kubectl apply -f [filename]`.

## Namespace

First of all, we will be creating a new namespace called Production, as it will be our working environment:

```
vim namespace.yaml
```
Then add the following to the file:

```
kind: Namespace
apiVersion: v1
metadata:
  name: production
  labels:
    name: production
```
After saving, run the following command to create the namespace:
```
kubectl apply -f namespace.yaml
```

## ConfigMap

Then, we will create the configMap in the production namespace, as it will hold environment variables used by other pods that might change:

```
vim configmap.yaml
```
Then add the following to the file:

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: nk-configmap
data:
  backend-url: backend-service
  redis-url: redis-service
  backend-port: "1337"
  arangodb-host: "nk-db-service"
  deployment-mode: "Kubernetes"
```
After saving, run the following command to create the configMap:

```
kubectl apply -f configmap.yaml -n production
```

## Secret

Since the ArangoDB will need a password and username, we will create a secret that will hold this data, for security reasons:

```
vim secret.yaml
```

Then add the following to the file (Please note that here the secret data has to be in base64 format):

```
apiVersion: v1
kind: Secret
metadata:
  name: arango-secret
type: Opaque
data:
  ARANGO_ROOT_PASSWORD: b3BlblNlc2FtZQ== 
  ARANGO_STORAGE_ENGINE: cm9ja3NkYg==
```
After saving, run the following command to create the secret:

```
kubectl apply -f secret.yaml -n production
```

## Ingress

Before deploying Ingress, we have to deploy the ingress controller using the command:
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.1.1/deploy/static/provider/cloud/deploy.yaml
```
After deploying the controller, create the `ingress.yaml` as follows:
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: camunda-ingress
  namespace: backend
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/affinity: "cookie"
    nginx.ingress.kubernetes.io/session-cookie-name: "hello-cookie"
    nginx.ingress.kubernetes.io/session-cookie-expires: "172800"
    nginx.ingress.kubernetes.io/session-cookie-max-age: "172800"
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
    nginx.ingress.kubernetes.io/affinity-mode: persistent
    nginx.ingress.kubernetes.io/session-cookie-hash: sha1
spec:
  defaultBackend:
    service:
      name: camunda-bpm-platform
      port:
        number: 31222
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix      
        backend:
          service:
            name: camunda-bpm-platform
            port:
              number: 31222
```
Deploy ingress using the command: 
```
kubectl apply -f ingress.yaml
```

## Redis

Create the redis configuration file that will be used to create the redis pods:

```
vim redis-configuration.yaml
```
Then add the following to the file:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-nk
  namespace: production
  labels:
    app: nk
    tier: data
    env: production
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nk
      tier: data
      env: production
  template:
    metadata:
      labels:
        app: nk
        tier: data
        env: production
    spec:
      containers:
      - name: redis-nk
        image: "docker.io/redis:6.0.5"
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
          limits:
            memory: "500Mi"
            cpu: "200m"
        ports:
        - containerPort: 6379
---
apiVersion: v1
kind: Service
metadata:
  name: redis-service
  namespace: production
  labels:
    app: nk
    tier: data
    env: production
spec:
  ports:
  - port: 6379
    targetPort: 6379
  selector:
    app: nk
    tier: data
    env: production
```

After saving, run the following command to create the pods:

```
kubectl apply -f redis-configuration.yaml
```

## ArangoDB

Create the arangoDB configuration file that will be used to create the arangodb pods:

```
vim arangodb-configuration.yaml
```
Then add the following to the file:


```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: arango-db
  namespace: production
  labels:
    app: nk-db
    tier: data
    env: production
spec:
  selector:
    matchLabels:
      app: nk-db
      tier: data
      env: production
  serviceName: nk-db-service
  replicas: 1
  updateStrategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: nk-db
        role: db
        tier: data
        env: production
    spec:
      containers:
      - name: arango-db
        env:
          - name: ARANGO_ROOT_PASSWORD
            valueFrom:
              secretKeyRef:
                name: arango-secret
                key: ARANGO_ROOT_PASSWORD
          - name: ARANGO_STORAGE_ENGINE
            valueFrom:
              secretKeyRef:
                name: arango-secret
                key: ARANGO_STORAGE_ENGINE
        image: arangodb/arangodb:3.6.3
        resources:
          limits:
            memory: "500Mi"
          requests:
            memory: "250Mi"
        ports:
        - containerPort: 8529
          name: nk-db
        volumeMounts:
        - name: pvc-db-arangodb3
          mountPath: /var/lib/arangodb3
        - name: pvc-db-arangodb3-apps
          mountPath: /var/lib/arangodb3-apps
  volumeClaimTemplates:
  - metadata:
      name: pvc-db-arangodb3
      labels:
        env: production
        tier: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 3Gi
  - metadata:
      name: pvc-db-arangodb3-apps
      labels:
        env: production
        tier: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
---
apiVersion: v1
kind: Service
metadata:
  name: nk-db-service
  namespace: production
spec:
  selector:
    app: nk-db
  clusterIP: None
  ports:
  - port: 8529
    targetPort: 8529
```

After saving, run the following command to create the pods:

```
kubectl apply -f arangodb-configuration.yaml
```
## Gateway

Create the gateway configuration file that will be used to create the gateway pod:

```
vim gateway.yaml
```
Then add the following to the file:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nk-gateway
  namespace: production
  labels:
    app: nk
    tier: frontend
    env: production
spec:
  selector:
    matchLabels:
      app: nk
      tier: frontend
      env: production
  replicas: 2
  template:
    metadata:
        labels:
          app: nk
          tier: frontend
          env: production
    spec:
      containers:
      - name: nk-gateway-mc
        image: nicolaselkhoury44/nk-gateway-service
        env:
          - name: BACKEND_HOST
            valueFrom: 
              configMapKeyRef:
                name: nk-configmap
                key: backend-url
          - name: REDIS_HOST
            valueFrom: 
              configMapKeyRef:
                name: nk-configmap
                key: redis-url
          - name: BACKEND_PORT
            valueFrom: 
              configMapKeyRef:
                name: nk-configmap
                key: backend-port
        resources:
          limits:
            memory: "128Mi"
            cpu: "500m"
        ports:
        - containerPort: 1337
---
apiVersion: v1
kind: Service
metadata:
  name: gateway-service
  namespace: production
  labels:
    app: nk
    tier: frontend
    env: production
spec:
  type: LoadBalancer
  ports:
  - port: 1337
    targetPort: 1337
  selector:
    app: nk
    tier: frontend
    env: production
```
After saving, run the following command to create the pod:

```
kubectl apply -f gateway.yaml
```

## Backend

Create the backend configuration file that will be used to create the backend pod:

```
vim backend.yaml
```
Then add the following to the file:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nk-backend
  namespace: production
  labels:
    app: nk
    tier: backend
    env: production
spec:
  selector:
    matchLabels:
      app: nk
      tier: backend
      env: production
  replicas: 2
  template:
    metadata:
        labels:
          app: nk
          tier: backend
          env: production
    spec:
      containers:
      - name: nk-backend-mc
        image: nicolaselkhoury44/nk-backend-service
        env:
          - name: ARANGODB_HOST
            valueFrom: 
              configMapKeyRef:
                name: nk-configmap
                key: arangodb-host
          - name: DEPLOYMENT_MODE
            valueFrom: 
              configMapKeyRef:
                name: nk-configmap
                key: deployment-mode
        resources:
          limits:
            memory: "500Mi"
            cpu: "500m"
        ports:
        - containerPort: 1337
---
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  namespace: production
  labels:
    app: nk
    tier: backend
    env: production
spec:
  ports:
  - port: 1337
    targetPort: 1337
  selector:
    app: nk
    tier: backend
    env: production
```

After saving, run the following command to create the pod:

```
kubectl apply -f backend.yaml
```

## Liveness Probe

To deploy a liveness probe, first create the yaml file:
```
vim liveness.yaml
```
Then add the below to the file:

```
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: liveness-exec
spec:
  containers:
  - name: liveness
    image: k8s.gcr.io/busybox
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 600
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5
```
After saving, run the following command to create the probe:

```
kubectl apply -f liveness.yaml -n production
```

## Readiness Probe

To deploy a readiness probe, first create the yaml file:
```
vim readiness.yaml
```
Then add the below to the file:

```
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: readiness
  name: readiness-exec
spec:
  containers:
  - name: readiness
    image: k8s.gcr.io/busybox
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy-readiness; sleep 30; rm -rf /tmp/healthy-readiness; sleep 600
    readinessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy-readiness
      initialDelaySeconds: 5
      periodSeconds: 5
```
After saving, run the following command to create the probe:

```
kubectl apply -f readiness.yaml -n production
```
# Testing the service

After the above was deployed successfully it is time now to test the service. To do so we need to follow the below steps:

- Get the loadbalancer's public endpoint (IP address) and use the External-IP as the destination. Use the below command to do so:
```
kubectl get svc -n production gateway-service
```

- Use the curl commands in the NK-Project under the API Explanation section page to do the different scenarios that insure that the microservices are working well. The below scenarios are required to prove certain functions are working as should. (Replace localhost with the gateway's IP shown with the above command).
