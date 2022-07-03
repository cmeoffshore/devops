
## Redis
Redis is deployed as a single instance deployment on kubernetes. Moreover, it has a clusterIP service for internal communication, and another Service of type NodePort for troubleshooting.

Apply the following YML file from the clients machine:

```
kind: ConfigMap
apiVersion: v1
metadata:
  name: redis-config
  labels:
    app: redis
data:
  redis.conf: |-
    dir /data
    port 6379
    bind 0.0.0.0
    appendonly yes
    protected-mode no
    #    requirepass
    pidfile /data/redis-6379.pid
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  labels:
    app: redis
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      #Initialize the system, modify the system configuration, and solve the warning information when redis starts
      initContainers:
        - name: system-init
          image: busybox:1.32
          imagePullPolicy: IfNotPresent
          command:
            - "sh"
            - "-c"
            - "echo 2048 > /proc/sys/net/core/somaxconn && echo never > /sys/kernel/mm/transparent_hugepage/enabled"
          securityContext:
            privileged: true
            runAsUser: 0
          volumeMounts:
          - name: sys
            mountPath: /sys
      containers:
        - name: redis
          image: redis:6.2
          ports:
            - containerPort: 6379
          resources:
            limits:
              cpu: 1000m
              memory: 1024Mi
            requests:
              cpu: 1000m
              memory: 1024Mi
          livenessProbe:
            tcpSocket:
              port: 6379
            initialDelaySeconds: 300
            timeoutSeconds: 1
            periodSeconds: 10
            successThreshold: 1
            failureThreshold: 3
          readinessProbe:
            tcpSocket:
              port: 6379
            initialDelaySeconds: 5
            timeoutSeconds: 1
            periodSeconds: 10
            successThreshold: 1
            failureThreshold: 3

      volumes:
        - name: config
          configMap:
            name: redis-config
        - name: sys
          hostPath:
            path: /sys
---
apiVersion: v1
kind: Service
metadata:
  name: redis
  labels:
    app: redis
spec:
  type: ClusterIP
  ports:
    - name: redis
      port: 6379
  selector:
    app: redis
---
apiVersion: v1
kind: Service
metadata:
  name: redis-np
  labels:
    app: redis
spec:
  type: NodePort
  selector:
    app: redis
  ports:
    - port: 6379
      targetPort: 6379
      nodePort: 30379
```
Several tests can be conducted to ensure the external and internal connectivity for Redis:

- **External connectivity**: Download any Redis client on your local machine, enter one of the RKE master or worker hostnames as the host, and the nodeport as the port. The connection must be successful:

![Screen Shot 2022-05-09 at 4.49.14 PM.png](../../.attachments/Screen%20Shot%202022-05-09%20at%204.49.14%20PM-e1c2ed58-5428-4823-b708-a328f86086f2.png)

- **Internal connectivity**: 
Create a redis-client pod with the correct hostname `kubectl run redis-client --rm --tty -i --restart='Never' --image redis --command -- redis-cli -h redis` and another pod with the wrong hostname `kubectl run redis-client2 --rm --tty -i --restart='Never' --image redis --command -- redis-cli -h redis22`

Once running, get the logs of both pods. `redis-client` with the correct hostname will not generate any logs, while the latter will complain:

![Screen Shot 2022-05-09 at 4.53.20 PM.png](../../.attachments/Screen%20Shot%202022-05-09%20at%204.53.20%20PM-6a2843f0-9af0-477e-b6a8-e7173940f15a.png)

Remember to delete the pods:
```
kubectl delete pod/redis-client
kubectl delete pod/redis-client2
