# Kubernetes Cluster (RKE2)
Rancher kuberenetes Engine 2 (RKE2) is Rancher's next-generation Kubernetes distribution.

It is a fully conformant Kubernetes distribution that focuses on security and compliance within the U.S. Federal Government sector. More information [here](https://docs.rke2.io/)

On the technical level the major differences between RKE and RKE2 to my understanding are:
* RKE2 runs as a single binary and can be installed as a systemd service while RKE is in simple terms the Kubernetes engine components broken into separate docker containers each serving the needed purpose.

* RKE2 uses containerd as a container runtime while RKE uses and will continue to use docker though it will be discontinued.

* RKE2 is more recent in terms of the Kubernetes engine it supports compared to RKE.

* RKE2 has more security feature compliance compared to RKE (SELinux...).

* RKE2 uses a configuration file on the level of /etc/ by default to configure the cluster contrary to the **cluster.yaml** used by RKE.


This tutorial will be installing a RKE2 cluster in HA mode. There are two types of nodes in the RKE2 cluster a **server** nodes which has the role of the control-plane / master node on the cluster, and an **agent** node which is used as the worker to run the workloads.

It is worth noting that the **server** nodes can be configured to play the dual role of a master and worker which it does by default if you didn't add taints that prohibited this in to the configuration file at creation time

### prerequisites

#### Fixed Registration Address
In a HA architecture and given the fact that the abstraction that was provided previously by RKE to the cluster in terms of networking is not present with the absence of docker we will need some extra supporting components to help aid a HA control-plane. It is referred to as the Fixed Registration Address in the official documentation. This address will act in a similar way to a Virtual IP that resides in front of a loadbalanced cluster were it unifies the communication with the cluster. Examples to provide the Fixed Registration Address service range from loadbalancers (layer-4 or 7) to Round-robin DNS and Virtual or elastic IP addresses. more information around these points [here](https://docs.rke2.io/install/ha/#high-availability)

#### Layer-4 loadbalancer
In our case we will be using a layer-4 software loadbalancer set on a virtual machine which is HA proxy.

#### Root Access
Root access is required to properly run and configure the RKE2 Server and agent systemd services.

### Important Notes

* In an HA setup the server nodes need to maintain quorum to run the cluster, by which the cluster won't boot unless it has enough nodes to establish a quorum (n/2 + 1). This is the case as well if a restart needs to be done on the level of the rke2-server systemd service.
* Please note, in the commands below the master nodes are added one after the other. If 2 master nodes where booted at the same time the second one is likely to fail. The reason for that is that there would too many learner etcd members in the cluster.
  * This is only the case upon registering the nodes to the cluster.
In the case of adding agent nodes, we can add all of them at the same time without a problem.


### Known issues and limitations 

* Firewalld conflicts with RKE2's default Canal (Calico + Flannel) networking stack. To avoid unexpected behavior, firewalld should be disabled on systems running RKE2.
* If NetworkManager is enabled you will need to go through the additional steps mentioned in the following resource [here](https://docs.rke2.io/known_issues/#networkmanager).
* RKE2 v1.19.5+ ships with containerd v1.4.x or later, hence should run on cgroups v2 capable systems. for more information on this check [here](https://docs.rke2.io/known_issues/#control-groups-v2 )
* Calico with vxlan encapsulation issue which mainly happens In ubuntu but please check any ways in the [official resource](https://docs.rke2.io/known_issues/#calico-with-vxlan-encapsulation) 
* It doesn’t recognize Oracle Linux as supported OS if we were to use rancher on top.
* Certificates expire every 12 months
    * By default, certificates in RKE2 expire in 12 months. If the certificates are expired or have fewer than 90 days remaining before they expire, the certificates are rotated when RKE2 is restarted.

### Installation 

For the installation, we will be following the official documentation to install a proper highly available RKE2 cluster. The official documentation provides a script to install the cluster components. The script makes a determination of what type of system it is. If it's an OS that uses RPMs (such as CentOS or RHEL), it will perform an RPM-based installation, otherwise, the script defaults to the tarball. RPM based installation is covered below.

**Perform all the below activities as root user unless advised otherwise**
#### Server nodes installation

##### Installation steps [First Server node in the Cluster]
The installation of the first server requires some attention. The exact sequence of actions need to be followed to get accurate results.

Each machine must have a unique hostname. If your machines do not have unique hostnames, set the node-name parameter in the `/etc/rancher/rke2/config.yaml` file and provide a value with a valid and unique hostname for each node.

* Check the installation guide to validate 
* Setup the load-balancer [**below section**]
* Open the needed ports if you wish your nodes' communication to go through firewalls (not in the same network segment) refer to the following resource for more information https://docs.rke2.io/install/requirements/#networking.
* Stop firewalld
```
systemctl disable firewalld
```
* Check if NetworkManager is installed and active and take action according to the above

```
systemctl status NetworkManager
```
  * If it is proceed with the below action points
    * Create a configuration file called `rke2-canal.conf` in `/etc/NetworkManager/conf.d` with the below contents:
```
[keyfile]
unmanaged-devices=interface-name:cali*;interface-name:flannel*
```
 * If the system has RKE2 already installed you will need to reboot the server if not proceed with the below:
    * Reload the NetworkManager service after the above changes. 
```
systemctl reload NetworkManager
```
   * In some operating systems like RHEL 8.4, NetworkManager includes two extra services called nm-cloud-setup.service and nm-cloud-setup.timer. These services add a routing table that interfere with the CNI plugin's configuration. Unfortunately, there is no config that can avoid that as explained in the issue. Therefore, if those services exist, they should be disabled and the node must be rebooted.Follow this [guide](https://docs.rke2.io/known_issues/#networkmanager).


* Install the Repo according to the OS by using the below command.
```
cat << EOF > /etc/yum.repos.d/rancher-rke2-1-18-latest.repo
[rancher-rke2-common-latest]
name=Rancher RKE2 Common Latest
baseurl=https://rpm.rancher.io/rke2/latest/common/centos/8/noarch
enabled=1
gpgcheck=1
gpgkey=https://rpm.rancher.io/public.key

[rancher-rke2-1-18-latest]
name=Rancher RKE2 1.18 Latest
baseurl=https://rpm.rancher.io/rke2/latest/1.18/centos/8/x86_64
enabled=1
gpgcheck=1
gpgkey=https://rpm.rancher.io/public.key
EOF
```
* Run the installer as sudo 
```
curl -sfL https://get.rke2.io | sh -
```
* Configure the Server / master node and prepare it to launch in `/etc/rancher/rke2/config.yaml`
  * To avoid certificate errors with the fixed registration address, you should launch the server with the tls-san parameter set
    * In /etc/rancher/rke2/config.yaml add the below you can add multiple tis-san in my case I added the FQDN of the hosts and the hosts' names including the load balancer you can add IPs as well. These will be included in the certificate of the cluster.
  * For the other master nodes add the below (loadballancer-name is the load-balancer machine hostname)
  * We will be adding the needed node-taints to make the master node non-schedulable.
```
tls-san:
  - domain-used-by-the-nodes.com
  - rke2-master-1
  - rke2-master-2
  - rke2-master-3
  - loadbalancer
node-taint:
  - "CriticalAddonsOnly=true:NoExecute"
selinux: true
```
* Start and enable the service then get the token On the first Server / master node **ONLY**.
```
systemctl status rke2-server
systemctl enable rke2-server
systemctl start rke2-server
```
* You can follow the logs with the below command (Starting requires some patience)
```
journalctl -u rke2-server -f
```

* Copy the token from the first master server from
    * /var/lib/rancher/rke2/server/node-token

##### Install and configure the Fixed Registration Address [Layer-4 Loadbalancer]

This section will explain the steps needed to install and configure our Fixed Registration Address which is in our case a software layer-4 loadbalancer on a separate virtual machine. We will be using HAproxy as our software Layer-4 load balancer.

Our use-case requires the setup of two haproxy pass through layer-4 load-balancer services. More information on the below setup is [here](https://serversforhackers.com/c/using-ssl-certificates-with-haproxy) under the 
HAProxy with SSL Pass-Through section.

We will be launching two instances of the service one for each listening port. This will require a workaround to launch the second server using the binary rather than the systemd service.

it is preferable to use a layer-4 loadbalancer appliance in a production ready environment.

* Install the HAproxy service using the below command.
```
dnf install haproxy
```


The firsts service (registration service) the service will listen on the port TCP/9345 this service is required for node registration. put the below information in `/etc/haproxy/haproxy.cfg`
```
frontend localhost
    bind *:9345
    option tcplog
    mode tcp
    default_backend k8

backend k8
    mode tcp
    balance roundrobin
    option ssl-hello-chk
    server k81 rke2-master:9345 check
    server k82 rke2-worker-1:9345 check
    server k83 rke2-worker-2:9345 check
```
* Start the service using the below command
```
systemctl enable haproxy
systemctl start haproxy
```

The second service will listen on the port TCP/6443 which is used by the Kube-API server `/etc/haproxy/haproxy_api.cfg`
```
frontend localhost
    bind *:6443
    option tcplog
    mode tcp
    default_backend k8

backend k8
    mode tcp
    balance roundrobin
    option ssl-hello-chk
    server k81 rke2-master:6443 check
    server k82 rke2-worker-1:6443 check
    server k83 rke2-worker-2:6443 check
```
* Start the service using the below command 
```
/usr/sbin/haproxy -D -f /etc/haproxy/haproxy_api.cfg -p /var/run/haproxy_s.pid
```



##### Installation steps [Other Server node in the Cluster]
This step will explain the installation process of the other server nodes. The exact sequence of actions need to be followed to get accurate results. A very important note is that the server numbers needs to be an odd number for proper quorum (3 in our case).

Each machine must have a unique hostname. If your machines do not have unique hostnames, set the node-name parameter in the `/etc/rancher/rke2/config.yaml` file and provide a value with a valid and unique hostname for each node.

* Check the installation guide to validate 
* Setup the load-balancer [**Above section**] to act as the registration address
* Open the needed ports if you wish your nodes communication goes through firewalls (not in the same network segment) refer to the following resource for more information https://docs.rke2.io/install/requirements/#networking.
* Stop firewalld
```
systemctl disable firewalld
```
* Check if NetworkManager is installed and active and take action according to the above

```
systemctl status NetworkManager
```
  * If it is proceed with the below action points
    * Create a configuration file called `rke2-canal.conf` in `/etc/NetworkManager/conf.d` with the below contents:
```
[keyfile]
unmanaged-devices=interface-name:cali*;interface-name:flannel*
```
 * If the system has RKE2 already installed you will need to reboot the server if not proceed with the below:
    * Reload the NetworkManager service after the above changes. 
```
systemctl reload NetworkManager
```
   * In some operating systems like RHEL 8.4, NetworkManager includes two extra services called nm-cloud-setup.service and nm-cloud-setup.timer. These services add a routing table that interfere with the CNI plugin's configuration. Unfortunately, there is no config that can avoid that as explained in the issue. Therefore, if those services exist, they should be disabled and the node must be rebooted.Follow this [guide](https://docs.rke2.io/known_issues/#networkmanager).


* Install the Repo according to the OS by using the below command.
```
cat << EOF > /etc/yum.repos.d/rancher-rke2-1-18-latest.repo
[rancher-rke2-common-latest]
name=Rancher RKE2 Common Latest
baseurl=https://rpm.rancher.io/rke2/latest/common/centos/8/noarch
enabled=1
gpgcheck=1
gpgkey=https://rpm.rancher.io/public.key

[rancher-rke2-1-18-latest]
name=Rancher RKE2 1.18 Latest
baseurl=https://rpm.rancher.io/rke2/latest/1.18/centos/8/x86_64
enabled=1
gpgcheck=1
gpgkey=https://rpm.rancher.io/public.key
EOF
```
* Run the installer as sudo 
```
curl -sfL https://get.rke2.io | sh -
```
* Configure the Server / master node and prepare it to launch in `/etc/rancher/rke2/config.yaml` where you will be using the token generated on the master to establish the proper connection and boot-up the cluster.
  * To avoid certificate errors with the fixed registration address, you should launch the server with the tls-san parameter set
  * We will use the loadbalancer address as our server address since it will relay the request to the available master (The first master in our case)
    * In /etc/rancher/rke2/config.yaml add the below you can add multiple tis-san in my case it was just k8s.cmeoffshore.com
  * To avoid certificate errors with the fixed registration address, you should launch the server with the tls-san parameter set
    * In /etc/rancher/rke2/config.yaml add the below you can add multiple tis-san in my case I added the FQDN of the hosts and the hosts' names including the load balancer you can add IPs as well. These will be included in the certificate of the cluster.
  * We will be adding the needed node-taints to make the master node non-schedulable.
  * We will also be using the token generated on the first master.

```
server: https://<domain>:9345
token: <token>
  - domain-used-by-the-nodes.com
  - rke2-master-1
  - rke2-master-2
  - rke2-master-3
  - loadbalancer
node-taint:
  - "CriticalAddonsOnly=true:NoExecute"
selinux: true
```

* Start and enable the service then get the token On the other Server / master nodes each at a time.
  * Please note, in the commands below the master nodes are added one after the other. If 2 master nodes where booted at the same time the second one is likely to fail. The reason for that is that there would too many learner etcd members in the cluster.
```
systemctl status rke2-server
systemctl enable rke2-server
systemctl start rke2-server
```
* You can follow the logs with the below command (Starting requires some patience)
```
journalctl -u rke2-server -f
```

* Make the kubectl command reachable
```
sudo ln -s /var/lib/rancher/rke2/bin/kubectl /usr/bin/kubectl
```
* Export the kubeconfig file to the default directory.
```
mkdir -p ~/.kube
cp -p /etc/rancher/rke2/rke2.yaml ~/.kube/config
```

* Check the node's status

```
kubectl get nodes
NAME            STATUS   ROLES                       AGE     VERSION
rke2-master     Ready    control-plane,etcd,master   123m    v1.22.7+rke2r2
rke2-worker-1   Ready    control-plane,etcd,master   6m30s   v1.22.7+rke2r2
rke2-worker-2   Ready    control-plane,etcd,master   20m     v1.22.7+rke2r2
```


#### Agent nodes installation

Each machine must have a unique hostname. If your machines do not have unique hostnames, set the node-name parameter in the `/etc/rancher/rke2/config.yaml` file and provide a value with a valid and unique hostname for each node.

* Check the installation guide to validate 
* Open the needed ports if your nodes' communication goes through firewalls (not in the same network segment) refer to the following resource for more information https://docs.rke2.io/install/requirements/#networking.
* Stop firewalld
```
systemctl disable firewalld
```
* Check if NetworkManager is installed and active and take action according to the above

```
systemctl status NetworkManager
```
  * If it is proceed with the below action points
    * Create a configuration file called `rke2-canal.conf` in `/etc/NetworkManager/conf.d` with the below contents:
```
[keyfile]
unmanaged-devices=interface-name:cali*;interface-name:flannel*
```
 * If the system has RKE2 already installed you will need to reboot the server if not proceed with the below:
    * Reload the NetworkManager service after the above changes. 
```
systemctl reload NetworkManager
```
   * In some operating systems like RHEL 8.4, NetworkManager includes two extra services called nm-cloud-setup.service and nm-cloud-setup.timer. These services add a routing table that interfere with the CNI plugin's configuration. Unfortunately, there is no config that can avoid that as explained in the issue. Therefore, if those services exist, they should be disabled and the node must be rebooted.Follow this [guide](https://docs.rke2.io/known_issues/#networkmanager).


* Install the Repo according to the OS by using the below command.
```
cat << EOF > /etc/yum.repos.d/rancher-rke2-1-18-latest.repo
[rancher-rke2-common-latest]
name=Rancher RKE2 Common Latest
baseurl=https://rpm.rancher.io/rke2/latest/common/centos/8/noarch
enabled=1
gpgcheck=1
gpgkey=https://rpm.rancher.io/public.key

[rancher-rke2-1-18-latest]
name=Rancher RKE2 1.18 Latest
baseurl=https://rpm.rancher.io/rke2/latest/1.18/centos/8/x86_64
enabled=1
gpgcheck=1
gpgkey=https://rpm.rancher.io/public.key
EOF
```
* Run the installer as sudo 
    * Run the installer as root
```
curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE="agent" sh -
```

* Copy the token from the first master server from
    * /var/lib/rancher/rke2/server/node-token
* In /etc/rancher/rke2/config.yaml add the below configuration 
* Add the server address for registration 

```
server: https://<host>:9345
token: -- #<------ add your token in here
selinux: true
```
* Start and enable the service then get the token On the other Server / master nodes each at a time.
```
systemctl status rke2-agent
systemctl enable rke2-agent
systemctl start rke2-agent
```
* You can follow the logs with the below command (Starting requires some patience)
```
journalctl -u rke2-agent -f
```

Check if the nodes are ready
```
NAME            STATUS   ROLES                       AGE     VERSION
rke2-master-1   Ready    control-plane,etcd,master   152m    v1.22.7+rke2r2
rke2-master-2   Ready    control-plane,etcd,master   35m     v1.22.7+rke2r2
rke2-master-3   Ready    control-plane,etcd,master   49m     v1.22.7+rke2r2
rke2-agent-1    Ready    <none>                      4m31s   v1.22.7+rke2r2
rke2-agent-2    Ready    <none>                      4m31s   v1.22.7+rke2r2
rke2-agent-3    Ready    <none>                      4m31s   v1.22.7+rke2r2
```



## Workload Prerequisites
### Storage
Some Deployments will require storage for data persistence. Which raises the need for a dynamic method for data provisioning on a local environment (our-case) **which is not required in a cloud setup**.

To workaround this issue we will be using the local-path dynamic provisioner from rancher. More information [here](https://github.com/rancher/local-path-provisioner).

Follow the below steps to install and configure the local-path provisioner:
- Install the local provisioned 
```
wget https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml
```

- Configure it to the standard Storage class and change its name to **standard**
 
```
vim local-path-storage.yaml
```
- Change the StorageClass section to resemble the following
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
  annotations:
          storageclass.kubernetes.io/is-default-class: "true"
provisioner: rancher.io/local-path
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Retain

```

- Apply the manifest
```
kubectl apply -f local-path-storage.yaml
```

- Check if the changes take effect, you should see the name change and a **(default)** next to the name

```
NAME                 PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
standard (default)   rancher.io/local-path   Delete          WaitForFirstConsumer   false                  7d17h
```
- Apply the needed SELinux context for the storage directory

```
semanage fcontext -a -t container_file_t  "/opt/local-path-provisioner(/.*)?" 
restorecon -Rv /opt/local-path-provisioner
```
### Install Helm

```
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3

chmod 700 get_helm.sh

./get_helm.sh

```
* Include the binaries in the system PATH
```
sudo ln -s /usr/local/bin/helm /usr/bin/helm
```

# Conclusion

Suited for what type of work
when to use it as HA and when not to
what capabilities does it have (security)
...
