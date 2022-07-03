# Introduction
RabbitMQ is open-source and lightweight message-broker software that originally implemented the Advanced Message Queuing Protocol and has since been extended with a plug-in architecture to support Streaming Text Oriented Messaging Protocol, MQ Telemetry Transport, and other protocols. RabbitMQ can be deployed in distributed and federated configurations to meet high-scale, high-availability requirements.

RabbitMq acts as a middleman for various services (e.g. a web application, as in this example). They can be used to reduce loads and delivery times of web application servers by delegating tasks that would normally take up a lot of time or resources to a third party that has no other job.

# Concepts
RabbitMQ is software where queues are defined, to which applications connect in order to transfer a message or messages.

A message can take any form of data or information, as an example it can be a text message, a file, a set of info, etc... The queue manager catches and stores the messages until a client or a consumer connects and takes a message off the queue. The receiving application then processes the message.

Message queueing allows web servers to process and respond to any requests quickly instead of being forced to perform resource-heavy procedures on the spot that may delay response time. 

It is also used when you want to distribute a message to multiple consumers or to balance loads between workers.

![message-queue-small.png](../../.attachments/message-queue-small-08255642-4615-4960-9d20-47d31055aec8.png)

## Exchanges

Exchanges are a component of a message-broker software where messages instead of being published directly to a queue, producer send it to the exchange that is responsible for the routing of a message to different queue with the help of bindings and routing keys. A binding is a link between a queue and an exchange.

![exchanges-bidings-routing-keys.png](../../.attachments/exchanges-bidings-routing-keys-1bf1f9fd-2fef-4a05-a6e8-114b9d62614f.png)

**Type of exchanges**:

- Direct: Messages are directly bonded to the exact match of the routing key of the massage, as an example if the queue is bound to the exchange with the binding key "key1", a message published to the exchange with a routing key key1 is routed to that queue.
- Fanout: A fanout exchange routes messages to all of the queues bound to it.
- Topic: The topic exchange does a wildcard match between the routing key and the routing pattern specified in the binding.
- Headers: Headers exchanges use the message header attributes for routing.

## Terminology
- Producer: Application that sends the messages.
- Consumer: Application that receives the messages.
- Queue: Buffer that stores messages.
- Message: Information that is sent from the producer to a consumer through RabbitMQ.
- Connection: A TCP connection between your application and the RabbitMQ broker.
- Channel: A virtual connection inside a connection. When publishing or consuming messages from a queue - it's all done over a channel.
- Exchange: Receives messages from producers and pushes them to queues depending on rules defined by the exchange type. To receive messages, a queue needs to be bound to at least one exchange.
- Binding: A binding is a link between a queue and an exchange.
- Routing key: A key that the exchange looks at to decide how to route the message to queues. Think of the routing key as an address for the message.
- AMQP: Advanced Message Queuing Protocol is the protocol used by RabbitMQ for messaging.
- Users: It is possible to connect to RabbitMQ with a given username and password. Every user can be assigned permissions such as rights to read, write and configure privileges within the instance. Users can also be assigned permissions for specific virtual hosts.
- Vhost, virtual host: Provides a way to segregate applications using the same RabbitMQ instance. Different users can have different permissions to different vhost and queues and exchanges can be created, so they only exist in one vhost.


# Downloading and Installing RabbitMQ
- Debian and Ubuntu: https://www.rabbitmq.com/install-debian.html
- RedHat Enterprise Linux, CentOS, Fedora, openSUSE: https://www.rabbitmq.com/install-rpm.html
- Docker: https://registry.hub.docker.com/_/rabbitmq/

# Configuration
https://www.rabbitmq.com/configure.html
While some settings in RabbitMQ can be tuned using environment variables, most are configured using a main configuration file, usually named rabbitmq.conf. This includes configuration for the core server as well as plugins. An additional configuration file can be used to configure settings that cannot be expressed in the main file's configuration format. This is covered in more details below.

The active configuration file can be verified by inspecting the RabbitMQ log file. It will show up in the log file at the top, along with the other broker boot log entries. For example:
```
node           : rabbit@example
home dir       : /var/lib/rabbitmq
config file(s) : /etc/rabbitmq/advanced.config
               : /etc/rabbitmq/rabbitmq.conf
```
Alternatively, the location of configuration files used by a local node, use the rabbitmq-diagnostics status command:
```
# displays key
rabbitmq-diagnostics status
```
and look for the Config files section that would look like this:
```
Config files

 * /etc/rabbitmq/advanced.config
 * /etc/rabbitmq/rabbitmq.conf
```
To inspect the locations of a specific node, including nodes running remotely, use the -n (short for --node) switch:
```
rabbitmq-diagnostics status -n [node name]
```


Example of configuration files can be found here:
- https://github.com/rabbitmq/rabbitmq-server/blob/master/deps/rabbit/docs/rabbitmq.conf.example
- https://github.com/rabbitmq/rabbitmq-server/blob/master/deps/rabbit/docs/advanced.config.example

# Ports
The final consideration for the Stateful Set is the ports to open on the RabbitMQ Pods. Protocols supported by RabbitMQ are all TCP-based and require the protocol ports to be opened on the RabbitMQ nodes. Depending on the plugins that are enabled on a node, the list of required ports can vary.

The example enabled_plugins file mentioned above enables a few plugins: rabbitmq_peer_discovery_k8s (mandatory), rabbitmq_management and rabbitmq_prometheus. Therefore, the service must open several ports relevant for the core server and the enabled plugins:
```
5672: used by AMQP 0-9-1 and AMQP 1.0 clients
15672: management UI and HTTP API)
15692: Prometheus scraping endpoint)
```
# Parameters and Policies
https://www.rabbitmq.com/parameters.html
**Parameters**
While much of the configuration for RabbitMQ lives in the configuration file, some things do not mesh well with the use of a configuration file:

- If they need to be the same across all nodes in a cluster
- If they are likely to change at run time

RabbitMQ calls these items parameters. 

Parameters can be set by invoking rabbitmqctl or through the management plugin's HTTP API. There are 2 kinds of parameters: **vhost-scoped parameters** and **global parameters**. Vhost-scoped parameters are tied to a virtual host and consist of a component name, a name and a value. Global parameters are not tied to a particular virtual host and they consist of a name and value.

**Policies**
Client-controlled properties in some of the protocols RabbitMQ supports generally work well but they can be inflexible: updating TTL values or mirroring parameters that way required application changes, redeployment and queue re-declaration (which involves deletion). In addition, there is no way to control the extra arguments for groups of queues and exchanges. Policies were introduced to address the above pain points.

A policy matches one or more queues by name (using a regular expression pattern) and appends its definition (a map of optional arguments) to the x-arguments of the matching queues. In other words, it is possible to configure x-arguments for multiple queues at once with a policy, and update them all at once by updating policy definition.

In modern versions of RabbitMQ the set of features which can be controlled by policy is not the same as the set of features which can be controlled by client-provided arguments.

Key policy attributes are:

- name: it can be anything but ASCII-based names without spaces are recommended
- pattern: a regular expression that matches one or more queue (exchange) names. Any regular expression can be used.
- definition: a set of key/value pairs (think a JSON document) that will be injected into the map of optional arguments of the matching queues and exchanges
- policy priority: used to determine which policy should be applied to a queue or exchange if multiple policies match its name
Policies automatically match against exchanges and queues, and help determine how they behave. Each exchange or queue will have at most one policy matching (see Combining Policy Definitions below), and each policy then injects a set of key-value pairs (policy definition) on to the matching queues (exchanges).

Policies can match only queues, only exchanges, or both. This is controlled using the apply-to flag when a policy is created.

Policies can change at any time. When a policy definition is updated, its effect on matching exchanges and queues will be reapplied. Usually it happens instantaneously but for very busy queues can take a bit of time (say, a few seconds).

Policies are matched and applied every time an exchange or queue is created, not just when the policy is created.

Example:
```
rabbitmqctl set_policy federate-me \
    "^federated\." '{"federation-upstream-set":"all"}' \
    --priority 1 \
    --apply-to exchanges
```
This matches the value "all" with the key "federation-upstream-set" for all exchanges with names beginning with "federated.", in the virtual host "/".

The "pattern" argument is a regular expression used to match exchange or queue names.

In the event that more than one policy can match a given exchange or queue, the policy with the greatest priority applies.

The "apply-to" argument can be "exchanges", "queues" or "all". The "apply-to" and "priority" settings are optional, in which case the defaults are "all" and "0" respectively.

# Virtual Hosts
https://www.rabbitmq.com/vhosts.html

RabbitMQ is multi-tenant system: connections, exchanges, queues, bindings, user permissions, policies and some other things belong to virtual hosts, logical groups of entities. If you are familiar with virtual hosts in Apache or server blocks in Nginx, the idea is similar. There is, however, one important difference: virtual hosts in Apache are defined in the configuration file; that's not the case with RabbitMQ: virtual hosts are created and deleted using rabbitmqctl or HTTP API instead.


# Authentication, Authorisation, Access Control

https://www.rabbitmq.com/access-control.html#authentication

Clients use RabbitMQ features to connect to it. Every connection has an associated user which is authenticated. It also targets a virtual host for which the user must have a certain set of permissions.

User credentials, target virtual host and (optionally) client certificate are specified at connection initiation time.

There is a default pair of credentials called the default user. This user can only be used for host-local connections by default. Remote connections that use it will be refused.

Production environments should not use the default user and create new user accounts with generated credentials instead.

hen the server first starts running, and detects that its database is uninitialised or has been deleted, it initialises a fresh database with the following resources:

- a virtual host named / (a slash)
- a user named guest with a default password of guest, granted full access to the / virtual host
It is advisable to pre-configure a new user with a generated username and password or delete the guest user or at least change its password to reasonably secure generated value that won't be known to the public.

After an application connects to RabbitMQ and before it can perform operations, it must authenticate, that is, present and prove its identity. With that identity, RabbitMQ nodes can look up its permissions and authorize access to resources such as virtual hosts, queues, exchanges, and so on.

Two primary ways of authenticating a client are username/password pairs and X.509 certificates. Username/password pairs can be used with a variety of authentication backends that verify the credentials.

Connections that fail to authenticate will be closed with an error message in the server log.

# Monitoring
https://www.rabbitmq.com/monitoring.html

Collect the following metrics on all hosts that run RabbitMQ nodes or applications:
- CPU stats (user, system, iowait & idle percentages)
- Memory usage (used, buffered, cached & free percentages)
- Virtual Memory statistics (dirty page flushes, writeback volume)
- Disk I/O (operations & amount of data transferred per unit time, time to service operations)
- Free disk space on the mount used for the node data directory
- File descriptors used by beam.smp vs. max system limit
- TCP connections by state (ESTABLISHED, CLOSE_WAIT, TIME_WAIT)
- Network throughput (bytes received, bytes sent) & maximum network throughput
- Network latency (between all RabbitMQ nodes in a cluster as well as to/from clients)


The recommended metric collection interval is 15 second. To collect at an interval which is closer to real-time, use 5 second - but not lower. For rate metrics, use a time range that spans 4 metric collection intervals so that it can tolerate race-conditions and is resilient to scrape failures.

For production systems a collection interval of 30 or even 60 seconds is recommended. Prometheus exporter API is designed to be scraped every 15 seconds, including production systems.

# Logging
There are two ways to configure log file location. One is the configuration file. This option is recommended. The other is the RABBITMQ_LOGS environment variable. It can be useful in development environments.

Default RabbitMQ logging configuration will direct log messages to a log file. Standard output is another option available out of the box.

Several outputs can be used at the same time. Log entries will be copied to all of them.

Different outputs can have different log levels. For example, the console output can log all messages including debug information while the file output can only log error and higher severity messages.

Logging to a File:

- log.file: log file path or false to disable the file output. The default value is taken from the RABBITMQ_LOGS environment variable or configuration file
- log.file.level: log level for the file output. The default level is info
- log.file.formatter: controls log entry format, text lines or JSON
- log.file.rotation.date, log.file.rotation.size, log.file.rotation.count for log file rotation settings
- log.file.formatter.time_format: controls timestamp formatting

# File and Directory Locations
https://www.rabbitmq.com/relocate.html
Every RabbitMQ node uses a number of files and directories to load configuration; store data, metadata, log files, and so on. Their location can be changed.

![Screen Shot 2022-05-09 at 4.41.42 PM.png](../../.attachments/Screen%20Shot%202022-05-09%20at%204.41.42%20PM-5f2ed818-efef-44f5-bf53-10fde7af0993.png)

![Screen Shot 2022-05-09 at 4.42.06 PM.png](../../.attachments/Screen%20Shot%202022-05-09%20at%204.42.06%20PM-a96551ba-3aaf-496c-bed1-b3a037b5c4e7.png)


# clustering and HA
rabbitmq.com/clustering.html#restarting
## kubernetes settings

```
cd
mkdir rabbitmq 
cd rabbitmq
nano rabbit-statefulset.yaml
```
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: rabbitmq
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: rabbitmq
rules:
- apiGroups: 
    - ""
  resources: 
    - endpoints
  verbs: 
    - get
    - list
    - watch
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: rabbitmq
subjects:
- kind: ServiceAccount
  name: rabbitmq
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: rabbitmq

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: rabbitmq-config
data:
  enabled_plugins: |
    [rabbitmq_federation,rabbitmq_management,rabbitmq_peer_discovery_k8s].
  rabbitmq.conf: |
    loopback_users.guest = false
    listeners.tcp.default = 5672

    cluster_formation.peer_discovery_backend  = rabbit_peer_discovery_k8s
    cluster_formation.k8s.host = kubernetes.default.svc.cluster.local
    cluster_formation.k8s.address_type = hostname
    cluster_formation.node_cleanup.only_log_warning = true
    ##cluster_formation.peer_discovery_backend = rabbit_peer_discovery_classic_config
    ##cluster_formation.classic_config.nodes.1 = rabbit@rabbitmq-0.rabbitmq.rabbits.svc.cluster.local
    ##cluster_formation.classic_config.nodes.2 = rabbit@rabbitmq-1.rabbitmq.rabbits.svc.cluster.local
    ##cluster_formation.classic_config.nodes.3 = rabbit@rabbitmq-2.rabbitmq.rabbits.svc.cluster.local
---
apiVersion: v1
kind: Secret
metadata:
  name: rabbit-secret
type: Opaque
data:
  # echo -n "cookie-value" | base64
  RABBITMQ_ERLANG_COOKIE: V0lXVkhDRFRDSVVBV0FOTE1RQVc=
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: rabbitmq
spec:
  serviceName: rabbitmq
  replicas: 3
  selector:
    matchLabels:
      app: rabbitmq
  template:
    metadata:
      labels:
        app: rabbitmq
    spec:
      serviceAccountName: rabbitmq
      initContainers:
      - name: config
        image: busybox
        command: ['/bin/sh', '-c', 'cp /tmp/config/rabbitmq.conf /config/rabbitmq.conf && ls -l /config/ && cp /tmp/config/enabled_plugins /etc/rabbitmq/enabled_plugins']
        volumeMounts:
        - name: config
          mountPath: /tmp/config/
          readOnly: false
        - name: config-file
          mountPath: /config/
        - name: plugins-file
          mountPath: /etc/rabbitmq/
      containers:
      - name: rabbitmq
        image: rabbitmq:3.8-management
        ports:
        - containerPort: 4369
          name: discovery
        - containerPort: 5672
          name: amqp
        env:
        - name: RABBIT_POD_NAME
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: metadata.name
        - name: RABBIT_POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: RABBITMQ_NODENAME
          value: rabbit@$(RABBIT_POD_NAME).rabbitmq.$(RABBIT_POD_NAMESPACE).svc.cluster.local
        - name: RABBITMQ_USE_LONGNAME 
          value: "true"
        - name: RABBITMQ_CONFIG_FILE
          value: "/config/rabbitmq"
        - name: RABBITMQ_ERLANG_COOKIE
          valueFrom:
            secretKeyRef:
              name: rabbit-secret
              key: RABBITMQ_ERLANG_COOKIE
        - name: K8S_HOSTNAME_SUFFIX
          value: .rabbitmq.$(RABBIT_POD_NAMESPACE).svc.cluster.local
        volumeMounts:
        - name: data
          mountPath: /var/lib/rabbitmq
          readOnly: false
        - name: config-file
          mountPath: /config/
        - name: plugins-file
          mountPath: /etc/rabbitmq/
      volumes:
      - name: config-file
        emptyDir: {}
      - name: plugins-file
        emptyDir: {}
      - name: config
        configMap:
          name: rabbitmq-config
          defaultMode: 0755

  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "standard"
      resources:
        requests:
          storage: 100Mi
---
apiVersion: v1
kind: Service
metadata:
  name: rabbitmq
spec:
  clusterIP: None
  ports:
  - port: 4369
    targetPort: 4369
    name: discovery
    #  - port: 25672
    #    targetPort: 25672
    #    name: clustering
  - port: 5672
    targetPort: 5672
    name: amqp
  selector:
    app: rabbitmq
---
apiVersion: v1
kind: Service
metadata:
  name: rabbit-np
spec:
  type: NodePort
  selector:
    app: rabbitmq
  ports:
    - port: 15672
      targetPort: 15672
      nodePort: 30080
```
now apply the configuration:
```
kubectl apply -f rabbit-statefulset.yaml
```
### Activating Mirroring and failover
1- login to the node
```
kubectl exec -it rabbitmq-0 -- /bin/bash
```
2- Execute the policy command to add a policy on all cluster nodes
```
rabbitmqctl set_policy ha-fed \
    ".*" '{"federation-upstream-set":"all", "ha-sync-mode":"automatic", "ha-mode":"nodes", "ha-params":["rabbit@rabbitmq-0.rabbitmq.default.svc.cluster.local","rabbit@rabbitmq-1.rabbitmq.default.svc.cluster.local","rabbit@rabbitmq-2.rabbitmq.default.svc.cluster.local"]}' \
    --priority 1 \
    --apply-to queues
```
## Testing deployment
### The easy way
Rabbit mq by default offer a managment user interface that can be accessed on port  `30080`:
On any host that contain a rabbitmq replication, get the host/machine ip address and in a browser:
go to the rabbit mq management dashboard ex:
```
172.17.0.153:30080
```
```
username: guest
password: guest
```
![Screen Shot 2022-05-11 at 11.22.47 PM.png](../../.attachments/Screen%20Shot%202022-05-11%20at%2011.22.47%20PM-cbcb3199-02d2-49f6-8963-83e05122d753.png)

you should be able to see in the above screenshot the cluster name and the nodes being part of it.

**policy check**
check if the policy was created successfully click the cluster on the UI name on the top right: rabbit@rabbitmq-0.rabbitmq.default.svc.cluster.local > Policies and check if the ha-fed policy was created.


![Screen Shot 2022-05-12 at 12.23.20 AM.png](../../.attachments/Screen%20Shot%202022-05-12%20at%2012.23.20%20AM-b93e7152-4e2a-4ba6-b94f-af8e6da78df6.png)

**Queue**
On the UI you should be able to see a new queue and the cluster status (primary and secondary node distribution) you can use any of the worker node IPs

A queue should be created after the application starts using the rabbitmq-clusters

At this point navigate to the Queues tab and check the created queues and check the cluster status after clicking on the queue name

![Screen Shot 2022-05-12 at 12.26.25 AM.png](../../.attachments/Screen%20Shot%202022-05-12%20at%2012.26.25%20AM-aaf6b787-fcb0-412a-bba6-13b2b6fb34e1.png)

**Node health**
Healthy and available nodes will have the green color in its matrics and showing the metrics values.

If you take a node down: ex in kubernetes:
```
kubectl delete pod rabbitmq-0 
```
![Screen Shot 2022-05-11 at 11.41.47 PM.png](../../.attachments/Screen%20Shot%202022-05-11%20at%2011.41.47%20PM-964c8301-2708-41e2-a6b1-7eeedc91a9ae.png)

Node 0 will be disconnected and not running and will glow in red.

Kubernetes will automatically bring a new pod and it will connect to the cluster directly, once it is up and running you will see in the management ui that the node is back to the green state again.

### The other way :P
In whatever way you are running rabbitmq (docker kubernetes, or on the host directly) the following can be applied.

Enter the host on which rabbitmq is running or the container: ex in kubernetes

```
kubectl exec -it rabbitmq-0 sh
```
now check the available node using:
```
rabbitmqadmin list nodes name type running
```
```
+------------------------------------------------------+------+---------+
|                         name                         | type | running |
+------------------------------------------------------+------+---------+
| rabbit@rabbitmq-0.rabbitmq.default.svc.cluster.local | disc | True    |
| rabbit@rabbitmq-1.rabbitmq.default.svc.cluster.local | disc | True    |
+------------------------------------------------------+------+---------+
```
If you stop a node or delete it maybe using `kubectl delete pod rabbitmq-1 ` and rerun the above command you she see that the deleted node running status will become false and when you turn it back the state will be back to true.

###Using rabbitmq-perf-test to Run a Functional and Load Test of the Cluster
RabbitMQ comes with a load simulation tool, PerfTest, which can be executed from outside of a cluster or deployed to Kubernetes using the perf-test public docker image. Here’s an example of how the image can be deployed to a Kubernetes cluster
```
kubectl run perf-test --image=pivotalrabbitmq/perf-test -- --uri amqp://guest:guest@rabbitmq
```
then run:
```
  kubectl logs -f perf-test
```
the output will be similar to:
```
id: test-215225-861, time: 147.596s, sent: 17156 msg/s, received: 14842 msg/s, min/median/75th/95th/99th consumer latency: 1921256/2102453/2159165/2206304/2211897 µs
id: test-215225-861, time: 148.598s, sent: 13344 msg/s, received: 14939 msg/s, min/median/75th/95th/99th consumer latency: 1839433/2034027/2124838/2187971/2208323 µs
id: test-215225-861, time: 149.598s, sent: 15610 msg/s, received: 15358 msg/s, min/median/75th/95th/99th consumer latency: 1776078/1995031/2066342/2118497/2138031 µs
id: test-215225-861, time: 150.601s, sent: 15542 msg/s, received: 13391 msg/s, min/median/75th/95th/99th consumer latency: 1817354/1956919/2034069/2126057/2176519 µs
id: test-215225-861, time: 151.601s, sent: 13371 msg/s, received: 14805 msg/s, min/median/75th/95th/99th consumer latency: 1866447/2034211/2127581/2195115/2222075 µs
id: test-215225-861, time: 152.612s, sent: 12596 msg/s, received: 13812 msg/s, min/median/75th/95th/99th consumer latency: 1842907/2043363/2115173/2169357/2212608 µs
id: test-215225-861, time: 153.612s, sent: 14956 msg/s, received: 13548 msg/s, min/median/75th/95th/99th consumer latency: 1849685/2062960/2125730/2181929/2202419 µs
id: test-215225-861, time: 154.614s, sent: 11761 msg/s, received: 14023 msg/s, min/median/75th/95th/99th consumer latency: 1962667/2172681/2221911/2301097/2330609 µs
id: test-215225-861, time: 155.623s, sent: 17668 msg/s, received: 13875 msg/s, min/median/75th/95th/99th consumer latency: 1912806/2110658/2186760/2260303/2266491 µs
id: test-215225-861, time: 156.637s, sent: 13814 msg/s, received: 14003 msg/s, min/median/75th/95th/99th consumer latency: 1896091/2090793/2164574/2217589/2273892 µs
id: test-215225-861, time: 157.652s, sent: 12545 msg/s, received: 13004 msg/s, min/median/75th/95th/99th consumer latency: 1965777/2119555/2199819/2276636/2307012 µs
id: test-215225-861, time: 158.652s, sent: 13371 msg/s, received: 15126 msg/s, min/median/75th/95th/99th consumer latency: 1956958/2161963/2239261/2322925/2356522 µs
id: test-215225-861, time: 159.672s, sent: 14981 msg/s, received: 14386 msg/s, min/median/75th/95th/99th consumer latency: 1791587/2018573/2103859/2181435/2215754 µs
id: test-215225-861, time: 160.674s, sent: 13344 msg/s, received: 12550 msg/s, min/median/75th/95th/99th consumer latency: 1862300/2081213/2151327/2209413/2233162 µs

```

And in the management dashboard you will see the matrix started to be shown as in this:
![Screen Shot 2022-05-12 at 12.58.06 AM.png](../../.attachments/Screen%20Shot%202022-05-12%20at%2012.58.06%20AM-11c4daee-1dc6-496e-a0a2-658205247563.png)

When you finish monitoring make sure to clean up using:
```
kubectl delete pod perf-test
```

### Clean up
To clean up the above setup you'll have to delete all the set components using the configuration yaml.
```
kubectl delete -f rabbit-statefulset.yaml 
```
delete the created volumes by deletein the corresponding volume-claims which should have the names data-rabbitmq-0 data-rabbitmq-1 data-rabbitmq-2
```
kubectl delete pvc data-rabbitmq-0 data-rabbitmq-1 data-rabbitmq-2
```

# Troubleshooting
https://www.rabbitmq.com/troubleshooting.html

# Grafana and prometheus
https://www.rabbitmq.com/prometheus.html

# RabbitMQ Checklist For Production Environments
- https://www.rabbitmq.com/production-checklist.html
- https://www.cloudamqp.com/blog/rabbitmq-checklist-for-production-environments-a-complete-guide.html
