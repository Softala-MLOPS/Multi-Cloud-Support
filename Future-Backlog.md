# Future backlog
* Investigate how Auto Scaling can be added to the system.
* Investigate how to add automatic Spin Up which starts the DataCruch VM automaticly if the resources of the original cluster are running low.
* Investigate how to add automatic shut down after certain time for the DataCrunch VM
* Clusters are set up with k3s at the moment. Invetigate if there is need for k8s. If the k3s doesn't have enough resources then modify the insturctions to be compatible with k8s. Liqo can be deployed both k3s and k8s clusters.