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
 - 
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
- You should be able to ping VM-B from VM-A with its CIDR
```bash
ping 10.0.0.2
```
Authors: Kosti Kangasmaa and Jouni Tuomela

## Submariner sandbox with Wireguard that submariner handles with 2 different Vm´s on Openstack 
