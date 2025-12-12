# Multi-cloud demo
This is a step-by-step guide for connecting an external k3s cluster to Oss-mlops-platforms k8s cluster running on my CSC cPouta vm. The goal of this demo is to work as an proof of concept for offloading workloads from Oss-mlops-platform ml development pipeline to an external cluster witch is located on seperate VM on a different cloud provider. This is achieved with connecting the two clusters with Liqo. Liqo also handles.
* Vpn tunnel for intercluster connections.
* Setup and management of a virtual node.
* Namespace extending between nodes.
* Offloading strategy for pods under said namespace.

## Technologies added to Oss-mlops-platform Stack

* K3s
* liqo
* liqoctl (multicluster networking + scheduling)
* Helm (indirectly used by liqoctl)
* WireGuard (built-in) – created automatically by Liqo for cross-cluster VPN

## Development Environments
### Key requirement
* One of the VMs has to have public IP-address. If both of the machines are using NAT(Network Address Translation) the connection can't be established.
* **My CSC cPouta IS behind NAT**
### Environment 1. Consumer-Vm  k8s Oss-mlops-platform (My CSC cPouta)

VM specs:

| Component | |
| --------- | --------- |
| Cloud provider | my CSC cPouta |
| RAM       | 32 GB     |
| vCPUs     | 8         |
| Disk      | 80 GB     |
| Network   | Floating IP (NAT) |
| OS | Ubuntu 24.04 LTS |
|Kubernetes distribution| k8s kind |

### Environment 2. Provider-Vm k3s cluster-b (Verda)
VM specs:

| Component | |
| --------- | --------- |
| Cloud provider | Verda |
| RAM       | 64 GB     |
| vCPUs     | 16         |
| Disk      | 80 GB     |
| Network   | Public IP (ipv4) |
| OS | Ubuntu 24.04 LTS |
|Kubernetes distribution| k3s |

## Architecture Overview
![mlops_cluste_federation](/images/mlops_cluster_federation.png)



# Setup Guide





## 1. Kubernetes Setup for OSS-MLops-Platform and Verda
 
### **Consumer Cluster (kind-mlops-platform / Cpouta-vm)**
 
Only requirement is that the OSS-MLOps Platform repository is cloned and installed correctly.
 
Install reference: https://github.com/OSS-MLOPS-PLATFORM/oss-mlops-platform/blob/main/tools/CLI-tool/Installations%2C%20setups%20and%20usage.md
 
### **Provider Cluster (cluster-b / Verda-vm)**
 
Update system packages:
 
```bash
apt update
```
 
Install Python and pip (required by OSS-MLOps Platform)
 
```bash
apt install -y python3 python3-pip
```
 
Verify:
 
```bash
python3 --version
pip3 --version
```
 
Download and install kustomize:
 
```bash
curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash -s -- 5.2.1
v5.2.1
chmod +x ./kustomize
mv ./kustomize /usr/local/bin/kustomize
```
 
Verify:
 
```bash
kustomize version
```
 
Install k3s:
 
```bash
curl -sfL https://get.k3s.io | sh -
```
 
Export kubeconfig:
 
```bash
sudo cat /etc/rancher/k3s/k3s.yaml > ~/cluster-b.yaml
chmod 600 ~/cluster-b.yaml
```
 
Prepare kubectl directory:
 
```bash
mkdir -p /root/.kube
sudo cat /etc/rancher/k3s/k3s.yaml > /root/.kube/config
```

## 2. Install liqoctl

On BOTH Vm´s:

```bash
curl --fail -LS \
  "https://github.com/liqotech/liqo/releases/download/v1.0.1/liqoctl-linux-amd64.tar.gz" \
  | tar -xz

sudo install -o root -g root -m 0755 liqoctl /usr/local/bin/liqoctl
```

Verify:

```bash
liqoctl version
```

Expected:

```
Client version: v1.0.1
Server version: Unknown
```

(Server version appears after installing Liqo.)

## 3. Install Liqo on both clusters

### Oss-mlops-platform (consumer)

Get Oss-mlops-platfor specific CIDRs
```bash
kubectl get nodes -o jsonpath='{.items[0].spec.podCIDR}'
echo
```
```bash 
kubectl cluster-info dump | grep -m1 -E "service-cluster-ip-range|serviceSubnet"
```
![get-cidrs](/images/kubectl-get-CIDR.png)

```bash
liqoctl install kind --cluster-id kind-mlops-platform --pod-cidr 10.244.0.0/24 --service-cidr 10.96.0.0/16
```

![liqoctl-install-mlops](/images/liqoctl-install-mlops-platform.png)

Verify with liqoctl info:
```bash
liqoctl info
```

![liqoctl-info-mlops-platform](/images/liqoctl-info-mlops-platform.png)

### Cluster-b (provider)

```bash
liqoctl install k3s --cluster-id cluster-b
```



## 4. Transfer Cluster-b Kubeconfig to Oss-mlops-platform
In this case we serve it via http.server, but it can be transfered with SCP for example.

On **Cluster-b**:

```bash
python3 -m http.server 8080
```

Output:

```
Serving HTTP on 0.0.0.0 port 8080...
```

On **Oss-mlops-platform**:

Download the kubeconfig

```bash
wget http://<Cluster-b-public-ip>:8080/cluster-b.yaml -O cluster-b.yaml
```

For example in our case:

```
86.38.238.14
```

Verify:

```bash
ls -l cluster-b.yaml
```

## 5. Fix certificate validation (Rewrite server IP in cluster-b.yaml)

On **Oss-mlops-platform**, modify cluster-b.yaml

```bash
nano cluster-b.yaml
```




Change server address to:

```
server: https://<Cluster-b-public-ip>:6443
```
![cluster-b](/images/cluster-b-yaml.png)

Verify connection:

```bash
kubectl --kubeconfig cluster-b.yaml get nodes
```
![kubectl-verify](/images/kubectl-get-nodes-verify.png)

If nodes appear → **certificates OK**.

## 6. Establish Liqo Peering 

On Oss-mlops-platform:

```bash
liqoctl peer \
  --kubeconfig ~/.kube/config \
  --remote-kubeconfig cluster-b.yaml
```

![liqoctl-peer](/images/liqoctl-peer.png)



### Validate Peering on both Vm´s
```bash
liqoctl info
```



![liqoctl-peering](/images/liqoctl-info-peering-mlops.png)


![liqoctl-peering](/images/liqoctl-info-peering-verda.png)


## 7. Pod Scheduling to Cluster-b

Liqo requires namespaces to be offloaded before scheduling. Mlops platform schedules training workloads under kubeflow namespace. Liqo´s default offloading strategy is **LocalAndRemote** which schedules future pods in the node in said namespace that has most recources available.

### Step 1 — Offload namespace

On Oss-mlops-platform:

```bash
liqoctl offload namespace kubeflow
```

This:

* Enables cross-cluster scheduling
* Creates a twin namespace on Cluster B
* Removes the Liqo protective taint
* Enables automatic remote scheduling

### Step 2 — Run test workload

```bash
kubectl --kubeconfig cluster-a.yaml run test --image=nginx
```

### Step 3 — Verify scheduling

#### On Oss-mlops-platform:

```bash
kubectl get pods -n kubeflow -o wide
```

![kubeflow-mlops](/images/kubeflow-mlops-o-wide.png)

We now can see that a new pod under the namespace kubeflow was scheduled on cluster-b, but unfortunately the pod started but failed to complete due to errors. This is still a proof of concept of succesfull pod offloading in namespaces over virtual nodes cross clusters between different cloud providers.

![pipeline](/images/pipeline-error.png)

#### On Cluster-b:
```bash
kubectl get pods --all-namespaces -o wide
```

![kubeflow-verda](/images/kubeflow-verda.png)



  
### Author

**Brown Ikechukwu Aniebonam**

## SSH Connnection for DataCrunch

#### Projects > Keys > SSH KEYS > +Create

If you want to use a visual desktop interface, you can find guide here:
https://docs.verda.com/cpu-and-gpu-instances/remote-desktop-access

## Move Images between MyCSC Projects
Image sharing between projects is currently not supported using the Pouta web interface. However sharing can be done by using OpenStack CLI and OpenStack RC File that is found inside project API Access. Links below helps to install and use CLI:  
https://docs.csc.fi/cloud/pouta/install-client/

"Sharing images between Pouta projects" at bottom of the web page:  
https://docs.csc.fi/cloud/pouta/adding-images/#sharing-images-between-pouta-projects

#### !!NOTE: OpenStack CLI does not give you any responds from invalid logins or most commands!!

### Author

**Eemil Airaksinen**

---

