 # Developer Guide
 ## Environment
 ### Development Environment n.1
 - 2 Openstack Vm´s in this case CSC cpouta
 - Ubuntu 24.04 Lts
 - 32 GB ram 
 - 80 GB disk space
 - 8 VCPU
 - ssh commandline connection 
 ## API requirements
 ## Technology Stack
 - Kind
 - K8s
 - Submariner
 - Wireguard
 - Docker
 - kubectl
 - curl
 

 # Required Features
 ## Setting up cpouta vm's
 
 1. Create 2 new projects at CSC
 2. Create virtual machines on both projects using cPouta [Guide](https://docs.csc.fi/cloud/pouta/launch-vm-from-web-gui/). (windows powershell in our case)
 3. Connect to vms with ssh via local powershell. cPuota [Guide](https://docs.csc.fi/cloud/pouta/connecting-to-vm/)

Authors: Kosti Kangasmaa and Jouni Tuomela

 ## Setting up Wireguard
 We need wireguard tunnel for Multi-Cloud-Feature because we need to access vm-a from vm-b and vice versa for ease of use and also to host Submariner Broker API at an address that both vm´s see
 - You are connected on vm´s via ssh setting up is faster when done with a pair because we need to run commands on both vm´s.
 - Wireguard requires root user access to work. We want to change user´s on both vms
   
```bash
  sudo -i
  ```

 - Update apt and Install wireguard
  
```bash
  apt update
  apt install wireguard
  ```
- generate wireguard private keys and public keys from them on both vm´s
```bash
  cd /home/etc/wireguard
  umask 077
  wg genkey | tee private.key | wg pubkey > public.key
```
- Create wireguard configuration files on both vms, name of the file determines name of tunnel.
- Conf files are needed to use wg-quick utility [manual](https://man7.org/linux/man-pages/man8/wg-quick.8.html)
```bash
cd /home/etc/wireguard
nano wg0.conf
```
- Fill the conf file with the required information. keys can be accessed with commands and floating ips can be accessed by CSC instances

```bash
cat private
cat public
```
- Vm-a`s conf example 
```
[Interface]

PrivateKey = aLlmR+Kbeb3tsadfasdlgnaslkdgjlkasndlfansdflkjnDkw= #placeholder for your private key

Address = 10.0.0.1/24 #vm-a future wireguard tunnel CIDR

ListenPort = 56949 #UDP port chosen for tunnel pick a port that is not occupied higher numbers usually are free
 
[Peer]

PublicKey = K5dvexBv4p1asdfjashdfkjashfdkjasdhkfjak9fcVjo= #placeholder for VM-B´s public key

Endpoint = 86.52.21.44:56949 #vm-b public ip (floating ip from CSC) and UDP port

AllowedIPs = 10.0.0.2/32 #vm-b future CIDR that will connect to the tunnel
 
```
- VM-B´s conf example

```
[Interface]

PrivateKey = aLlmR+Kbeb3tsadfasdlgnaslkdgjlkasndlfansdflkjnDkw= #placeholder for your private key

Address = 10.0.0.2/24 #VM-B future wireguard tunnel CIDR

ListenPort = 56949 #UDP port chosen for tunnel pick a port that is not occupied higher numbers usually are free
 
[Peer]

PublicKey = K5dvexBv4p1asdfjashdfkjashfdkjasdhkfjak9fcVjo= #placeholder for VM-A´s public key

Endpoint = 86.62.81.124:56949 #VM-A´s public ip (floating ip from CSC) and UDP port

AllowedIPs = 10.0.0.1/32 #VM-A future CIDR that will connect to the tunnel
```
- Next we need to Create a new Openstack security groups on CSC for both VM´s. Name the group as wireguard
- VM-A
  - Ingress 	IPv4 	UDP PORT=56949 VM-B´s-public-ip as remote=86.52.21.44
  - Egress 	IPv4 	UDP PORT=56949 VM-A´s-public-ip as remote=86.62.81.124
- VM-B
  - Ingress 	IPv4 	UDP PORT=56949 VM-A´s-public-ip as remote=86.62.81.124
  - Egress 	IPv4 	UDP PORT=56949 VM-B´s-public-ip as remote=86.52.21.44

- Apply said groups to VM instances
- Then run wg-quick on both vms 
```bash
wg-quick up wg0
```
- Inspect wireguard info with wg

![wg](/images/wg.png)

- You should be able to ping VM-B from VM-A with its tunnel ip

```bash
ping 10.0.0.2
```

![wg-ping](/images/wg-ping.png)
Authors: Kosti Kangasmaa and Jouni Tuomela

## Allow vm-a to connect to vm-b
These instructions are meant only for CSC's cPouta virtual machines.

They configure vm-b to accept SCP connection from vm-a using password authentication insted of SSH keys.

#### Switch to root user on vm-b
```bash
sudo -i
  ```
- You’ll enter the root shell to perform system-level changes.

#### Create a new user for SSH/SCP access
```bash
sudo adduser <username>
  ```
- Ubuntu will prompt you to set and confirm a password.

- After that, it will ask for optional user information — just press Enter to skip.

- Finally, confirm with Y when asked if the information is correct.

- You should see:
```
passwd: password updated successfully
```
#### Enable password authentication for SSH
- Navigate to SSH configuration directory:
```bash
cd /etc/ssh/sshd_config.d
  ```
![wg-ping](/images/navigate.png)
- Edit the cloud image configuration file:
```bash
 nano 60-cloudimg-settings.conf
  ```
- Change its contents to:
```bash
PasswordAuthentication yes
  ```
![wg-ping](/images/password_authentiction.png)
- Confirm changes 
```
  CTRL + O
  ```
- Name the file 
```
  ENTER
  ```
- Exit the file
```
  CTRL + X
  ```
- This overrides the default cPouta settings (which diables password logins for security reasons)
#### Restart SSH service
```bash 
  systemctl restart ssh
  ```
#### Result
- Now vm-a can connect to vm-b using SCP or SSH with the credential of the newly created user:
```bash
scp <file> <username>@<vm-b-ip>:<destination>
  ```
#### Security note
- Password authentication should be enabled only for internal testing or controlled environments like cPouta VMs.

- In production systems, prefer SSH key-based authentication for security.


## Submariner sandbox with Wireguard that submariner handles with 2 different Vm´s on Openstack
Start by installing required technology stack, this demo has been done on versions stated below.
  
  - Docker version 28.5.1, build e180ab8
  - Kind kind version 0.30.0
  - Kubectl
    Client Version: v1.34.1
    Kustomize Version: v5.7.1
  - Wireguard & wireguard-tools v1.0.20210914
  - subctl version v0.21.0
  - Existing wireguard tunnel between vm´s
  
### Creating clusters with kind
- Start by creating config files that the clusters are built with.
#### VM-A
```Bash
nano cluster-a-kind.yaml
```
- The broker api of submariner is hosted inside our existing wireguard tunnel on vm-a at 10.0.0.1:6443 so it is discoverable on vm-b.
- The created subnets should not overlap with vm-b´s clusters subnets.
- UDP ports 4500 and 4490 are for routing the wireguard tunnel submariner creates.

```
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  apiServerAddress: "10.0.0.1"
  apiServerPort: 6443
  podSubnet: "10.244.0.0/16"
  serviceSubnet: "10.96.0.0/12"
nodes:
- role: control-plane
- role: worker
  extraPortMappings:
  - containerPort: 4500
    hostPort: 4500
    protocol: UDP
  - containerPort: 4490
    hostPort: 4490
    protocol: UDP
```
- Create the cluster from config file

```bash
kind create cluster --name cluster-a --config cluster-a-kind.yaml
```
- To use created cluster

```bash
kubectl cluster-info --context kind-cluster-a
```
- Next we need to label the workernode as a gateway node for submariner traffic

```bash
kubectl --context kind-cluster-a label node cluster-a-worker submariner.io/gateway=true --overwrite
```
- Lets deploy submariner broker and join our cluster-a to it
- Deploying broker creates a brokerfile broker-info.subm and we use that file for joining all clusters including clusters on different vm´s

```bash
subctl deploy-broker
```
-   Next we join cluster-a to the broker with brokerfile

```bash
 subctl join broker-info.subm --context kind-cluster-a --clusterid vm-a --cable-driver wireguard
```
![subctl join vm-a](/images/subctl-join-vm-a.png)
- We can look up info and diagnostics about our submariner with the commands
```bash
subctl show all
subctl diagnose all
```
- To join clusters on other vm´s we need to send the brokerfile to the vm´s we are joining clusters on with scp for example

```bash
scp broker-info.subm <user>@<VM-b´s ip>:~/
```

#### VM-B

```bash
nano cluster-b-kind.yaml
```
- Different subnets for cluster-b
```
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  podSubnet: "10.245.0.0/16"
  serviceSubnet: "10.80.0.0/12"
nodes:
- role: control-plane
- role: worker
  extraPortMappings:
  - containerPort: 4500
    hostPort: 4500
    protocol: UDP
  - containerPort: 4490
    hostPort: 4490
    protocol: UDP
```
- Create the cluster from config file

```bash
kind create cluster --name cluster-b --config cluster-b-kind.yaml
```
- To use created cluster

```bash
kubectl cluster-info --context kind-cluster-b
```
- Next we need to label the workernode as a gateway node for submariner traffic

```bash
kubectl --context kind-cluster-b label node cluster-b-worker submariner.io/gateway=true --overwrite
```
- To join vm-b´s cluster to vm-a´s cluster via submariner broker with submariner we need to have the brokerfile that vm a has created when deploying the broker
- Join cluster-b with brokerfile

```bash
subctl join broker-info.subm --context kind-cluster-b --clusterid vm-b --cable-driver wireguard
```
- Connection takes few seconds, but to confirm and diagnose submariner connection on either vm.

```bash
subctl show all
```
![subctl show all](/images/subctl-show-all-vm-a.png)

```bash
subctl diagnose all
```
![subctl diagnose all](/images/subctl-diagnose-all-vm-a.png)

Now the connections should be up and running.

Author: Kosti Kangasmaa