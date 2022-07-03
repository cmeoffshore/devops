# Install kubectl and RKE:

- Install dependencies
```
sudo apt install wget
```

First of all, on the rke-elk-master **Master node**, 

- Install Kubectl:

```
curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl

chmod +x ./kubectl

sudo mv ./kubectl /usr/local/bin/kubectl

kubectl version --client
```

- Install RKE:

```
curl -s https://api.github.com/repos/rancher/rke/releases/latest | grep download_url | grep amd64 | cut -d '"' -f 4 | wget -qi -

chmod +x rke_linux-amd64

sudo mv rke_linux-amd64 /usr/local/bin/rke

rke -v
```

- Install Helm:

```
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3

chmod 700 get_helm.sh

./get_helm.sh

helm version
```

- Include the binaries in the system path:

```
sudo ln -s /usr/local/bin/kubectl /usr/bin/kubectl
sudo ln -s /usr/local/bin/helm /usr/bin/helm
sudo ln -s /usr/local/bin/rke /usr/bin/rke
```

# RKE Cluster Pre-flight Setup

Before initializing the RKE cluster, perform the following on **all three nodes**, using the **root** user:

```
sudo yum update -y
```

- Synchronize Chronyd: `systemctl status chronyd`
- Load kernel modules for RKE

```
for module in br_netfilter ip6_udp_tunnel ip_set ip_set_hash_ip ip_set_hash_net iptable_filter iptable_nat iptable_mangle iptable_raw nf_conntrack_netlink nf_conntrack nf_conntrack_ipv4   nf_defrag_ipv4 nf_nat nf_nat_ipv4 nf_nat_masquerade_ipv4 nfnetlink udp_tunnel veth vxlan x_tables xt_addrtype xt_conntrack xt_comment xt_mark xt_multiport xt_nat xt_recent xt_set  xt_statistic xt_tcpudp;
     do
       if ! lsmod | grep -q $module; then
         echo "module $module is not present";
         modprobe $module
       fi;
    done
```

- Disable swap by commenting out the swap option in `/etc/fstab` and turn off swap `sudo swapoff -a`

Confirm swap is disabled (size should be 0).

```
free -h

# Output
              total        used        free      shared  buff/cache   available
Mem:           3928         406        1088          24        2433        3206
Swap:             0           0           0
```

- Customize sysctl parameters:
```
sudo tee -a /etc/sysctl.d/99-kubernetes.conf <<EOF
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sysctl --system
```

- Install Docker:
```
curl https://releases.rancher.com/install-docker/20.10.sh | sh
```

- Enable and start the docker service
```
systemctl enable docker
systemctl start docker
systemctl status docker
```

- Add the rke user
```
sudo adduser rke
```
The root user can't run RKE because there is a known bug of OpenSSH on Red Hat's Bugzilla about socket forwarding. Therefore, a user to run RKE containers will be created. We also added the user to the Docker group:
```
sudo usermod -aG docker rke
```

- Enable SSH Access

SSH access using SSH keys must be allowed, in order to allow RKE to SSH from the RKE (rke-elk-master in this case) machine, into all of the VMs. In this regard, on the **Master** machine, create an SSH key-pair:

```
ssh-keygen -o -a 100 -t rsa -b 4096 -N "" -f ~/.ssh/id_rsa -C "rke key"
```

Once created, copy the public key to the authorized-keys file **into all of the remaining VMs**. On every machine, perform the following commands, to create the required files and folders:

```
sudo su rke
mkdir -p ~/.ssh
touch ~/.ssh/authorized_keys 
chmod -R go= ~/.ssh
```

From the **Master** machine, copy the `~/.ssh/id_rsa.pub` key to the `~.ssh/authorized_keys` file of every remaining machine. Make sure you are using the `rke` user.

- Disable firewalld
Using the original admin account (or root), stop the firewalld service:

```
sudo systemctl disable firewalld
sudo systemctl stop firewalld
sudo systemctl status firewalld
```

# RKE Cluster Setup
Using the admin user, initiate the RKE cluster setup wizard from the RKE machine `rke config --name cluster.yml`:

```
rke config --name cluster.yml
[+] Cluster Level SSH Private Key Path [~/.ssh/id_rsa]: 
[+] Number of Hosts [1]: 3
[+] SSH Address of host (1) [none]: 10.0.3.4
[+] SSH Port of host (1) [22]: 
[+] SSH Private Key Path of host (10.0.3.4) [none]: 
[-] You have entered empty SSH key path, trying fetch from SSH key parameter
[+] SSH Private Key of host (10.0.3.43) [none]: 
[-] You have entered empty SSH key, defaulting to cluster level SSH key: ~/.ssh/id_rsa
[+] SSH User of host (10.0.3.4) [ubuntu]: rke
[+] Is host (10.0.3.4) a Control Plane host (y/n)? [y]: 
[+] Is host (10.0.3.4) a Worker host (y/n)? [n]: 
[+] Is host (10.0.3.4) an etcd host (y/n)? [n]: y
[+] Override Hostname of host (10.0.3.4) [none]: 
[+] Internal IP of host (10.0.3.4) [none]: 10.0.3.4
[+] Docker socket path on host (10.0.3.4) [/var/run/docker.sock]: 
[+] SSH Address of host (2) [none]: 10.0.3.5
[+] SSH Port of host (2) [22]: 
[+] SSH Private Key Path of host (10.0.3.5) [none]: 
[-] You have entered empty SSH key path, trying fetch from SSH key parameter
[+] SSH Private Key of host (10.0.3.5) [none]: 
[-] You have entered empty SSH key, defaulting to cluster level SSH key: ~/.ssh/id_rsa
[+] SSH User of host (10.0.3.5) [ubuntu]: rke
[+] Is host (10.0.3.5) a Control Plane host (y/n)? [y]: n
[+] Is host (10.0.3.5) a Worker host (y/n)? [n]: y
[+] Is host (10.0.3.5) an etcd host (y/n)? [n]: n
[+] Override Hostname of host (10.0.3.5) [none]: 
[+] Internal IP of host (10.0.3.5) [none]: 10.0.3.5
[+] Docker socket path on host (10.0.3.5) [/var/run/docker.sock]: 
[+] SSH Address of host (3) [none]: 10.0.3.6
[+] SSH Port of host (3) [22]: 
[+] SSH Private Key Path of host (10.0.3.6) [none]: 
[-] You have entered empty SSH key path, trying fetch from SSH key parameter
[+] SSH Private Key of host (10.0.3.6) [none]: 
[-] You have entered empty SSH key, defaulting to cluster level SSH key: ~/.ssh/id_rsa
[+] SSH User of host (10.0.3.6) [ubuntu]: rke
[+] Is host (10.0.3.6) a Control Plane host (y/n)? [y]: n
[+] Is host (10.0.3.6) a Worker host (y/n)? [n]: y
[+] Is host (10.0.3.6) an etcd host (y/n)? [n]: n
[+] Override Hostname of host (10.0.3.6) [none]: 
[+] Internal IP of host (10.0.3.6) [none]: 10.0.3.6
[+] Docker socket path on host (20.74.252.138) [/var/run/docker.sock]: 
[+] Network Plugin Type (flannel, calico, weave, canal, aci) [canal]: 
[+] Authentication Strategy [x509]: 
[+] Authorization Mode (rbac, none) [rbac]: 
[+] Kubernetes Docker image [rancher/hyperkube:v1.21.8-rancher2]: 
[+] Cluster domain [cluster.local]: 
[+] Service Cluster IP Range [10.43.0.0/16]: 
[+] Enable PodSecurityPolicy [n]: 
[+] Cluster Network CIDR [10.42.0.0/16]: 
[+] Cluster DNS Service IP [10.43.0.10]:  
```

- Spin the RKE cluster
Use the below command to spin the cluster with the above configuration. You should be in the same location as the previously created cluster.yml file: `rke up`

# Kubectl
Now that RKE is setup, it will create a file in the current directory named `kube_config_cluster.yml`. We need to move this file to a certain directory for kubectl to read from it, or point it to it.
- method 1: move the file
   ```
   mkdir ~/.kube
   mv ./kube_config_cluster.yml ~/.kube/config
   ```
- method 2: create an env file
   ```
   export KUBECONFIG=$(pwd)/kube_config_cluster.yml
   ```
Note that the second method will only work during the current session, the `export` command should be added in the `.bash_profile` file so it would be consistent.

# Storage
The ElasticSearch component will require storage for data persistence. Which raises the need for a dynamic method for data provisioning on a local environment (our-case) **which is not required in a cloud setup**.

To workaround this issue we will be using the local-path dynamic provisioner from rancher. More information [here](https://github.com/rancher/local-path-provisioner).

Follow the below steps to install and configure the local-path provisioner:
- On the **Master** node, Install the local provisioned:
```
wget https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml
```

- Configure it to the standard Storage class and change its name to **standard**:
 
```
vim local-path-storage.yml
```
- Change the StorageClass section to resemble the following:
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

- Check if the changes took effect with the below command you should see the name change and a **(default)** next to the name
```
kubectl get storageclass
```
Expected output:

```
NAME                 PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
standard (default)   rancher.io/local-path   Retain          WaitForFirstConsumer   false                  3m19s

```