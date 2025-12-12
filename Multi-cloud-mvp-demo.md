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




# Setup Guide

Complete Developer Setup Using:

* **mlops-vm (CSC)** → Cluster A
* **weak-mind-unfolds-fin-01 (Datacrunch)** → Cluster B (GPU V100)

Final result:
Cluster A **can schedule pods** onto Cluster B **seamlessly**, through the Liqo virtual node.



## 2. Install k3s on Each Cluster

### **Cluster A (CSC / mlops-vm)**

```bash
curl -sfL https://get.k3s.io | sh -
```

Export kubeconfig to your user:

```bash
sudo cat /etc/rancher/k3s/k3s.yaml > ~/.kube/config
```

### **Cluster B (Datacrunch / weak-mind-unfolds-fin-01)**

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

## 3. Install liqoctl (manual method)

On BOTH clusters:

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

## 4. Install Liqo

### Cluster A (consumer)

```bash
liqoctl install k3s --cluster-id cluster-a
```

### Cluster B (provider)

```bash
liqoctl install k3s --cluster-id cluster-b
```

## 5. Transfer Cluster B Kubeconfig to Cluster A

On **Cluster B**:

```bash
python3 -m http.server 8080
```

Output:

```
Serving HTTP on 0.0.0.0 port 8080...
```

On **Cluster A**:

```bash
wget http://<dc-ip>:8080/cluster-b.yaml -O cluster-b.yaml
```

For example:

```
135.181.8.194
```

Verify:

```bash
ls -l cluster-b.yaml
```

## 6. Fix certificate validation (Rewrite server IP in cluster-b.yaml)

On **Cluster A**, open:

```bash
nano cluster-b.yaml
```

Find:

```
server: https://10.xxx.xxx.xxx:6443
```

Change to:

```
server: https://135.181.8.194:6443
```

**Do NOT modify certificate-authority-data**
TLS will still validate correctly.

Verify connection:

```bash
kubectl --kubeconfig cluster-b.yaml get nodes
```

If nodes appear → **certificates OK**.

## 7. Establish Liqo Peering (MOST IMPORTANT STEP)

On Cluster A:

```bash
liqoctl peer \
  --kubeconfig cluster-a.yaml \
  --remote-kubeconfig cluster-b.yaml
```

Expected output includes:

* Connection is established
* Tenant applied
* Identity generated
* ResourceSlice resources: Accepted

## 8. Validate Peering

```bash
kubectl --kubeconfig cluster-a.yaml get nodes
```

Expected:

```
NAME        STATUS   ROLES
cluster-b   Ready    agent
mlops-vm    Ready    control-plane
```

## 9. Test Pod Scheduling to Cluster B

Liqo requires namespaces to be offloaded before scheduling.

### Step 1 — Offload namespace

On Cluster A:

```bash
liqoctl offload namespace default --kubeconfig cluster-a.yaml
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

```bash
kubectl --kubeconfig cluster-a.yaml get pods -o wide
```

Expected:

```
test   1/1   Running   10.40.0.18   cluster-b
```

**cluster-b** = Liqo virtual node on Datacrunch.

This confirms:

* Peering works
* WireGuard tunnel established
* Scheduling works
* Pod is running on V100 GPU cluster
  
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

