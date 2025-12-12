# Multi-Cloud-Support
Repository for OSS MLOps Multi-Cloud Support sub group. 

The goal of the group has been to find a solution for sharing Kubernetes resources between clusters. The main goal is to offload additional resources to clusters which have available resources.

For connecting the clusters and resource sharing, we have selected open-source software Liqo, as it handles both.

The testing environment has been a cPouta VM in CSC and a VM in Verda. Both VMs have Kubernetes cluster which are set up with k3s and k8s and orchestrated by Liqo.

## Proof of concept

See the [Demo Guide](Multi-cloud-poc-demo.md) for detailed setup and usage instructions.

## Future backlog

See the [Future backlog](Future-Backlog.md) for projects next steps.

## Investigations and tests for alternative solutions

See the [Failed investigations](Failed-Investigations.md) for other solutions we have researched during the project.

## Verda API and possible usages
Before you can use REST API:
 1. Get Credentials from the Verda Cloud Dashboard
 2. Generate Access Token
More in-depth guidance and about every usage of API:
https://api.verda.com/v1/docs#description/verda-cloud

Potential API Endpoints for Integration:
 - Instances
   - Get instances
   - Deploy instances
   - Perform action on instance or multiple instances
   - Get instance by ID
 - Instance Availability (if current option is not available)

Setups for SSH and startup scripts can be done via dashboard.

Scaling will be handled on cPouta (cluster A) by deploying and deleting instances (instance needs to be totally removed, or else it will still use credits from project's balance!)

    
    

