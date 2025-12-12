# Multi-Cloud-Support
Repository for OSS MLOps Multi-Cloud Support sub group. 

The goal of the group has been to find a solution for sharing Kubernetes resources between clusters. The main goal is to offload additional resources to clusters which have available resources.

For connecting the clusters and resource sharing, we have selected open-source software Liqo, as it handles both.

The testing environment has been a cPouta VM in CSC and a VM in Verda. Both VMs have Kubernetes cluster which are sert up with k3s and k8s and orchestrated by Liqo.

## Proof of concept

See the [Demo Guide](Multi-cloud-poc-demo.md) for detailed setup and usage instructions.

## Future backlog

See the [Future backlog](Future-Backlog.md) for projects next steps.

## Investigations and tests for alternative solutions

See the [Failed investigations](Failed-Investigations.md) for other solutions we have researched during the project.


    
    

