# Future backlog
* MVP example of GPU recource sharing.
  - Get and install correct drivers and device plugins so k3s or k8s detects available gpu recources
  - Assert available gpu recources with liqo so the consumer can detect providers resources via the created virtual node. 
* Investigate how Auto Scaling can be added to the system. This can include:
  - How does the system detect and decide when to scale?
  - Provisioning and creating new VM´s upon gpu / cpu load requirements.
  - Renting a Vm instance with automaticly scaling recourses by scaling and limiting the computing capacity of nodes. This can maby be done with Liqo ResourceSlice virtual nodes on consumer. You can assert the virtual nodes computing capacity even though truely capacity is greater.
  - A naming system for multiple VM´s, clusters, nodes and CIDR provisioning for non-overlapping addresses with different machines and clusters.
* Investigate how to add automatic shut down and deletion for joined VM after determined time of provider node idle.
  - Do all created Vm´s have the same time of idle or does the potential idle time scale with somehow with the amount of instances active and idling.
  - Should the shutdown time be easily customizable by mlops user / admin
* Investigate how to create a safe de-coupling and uninstalling script of the joined clusters to ensure succesful rejoining of clusters. (liqo specific problem).
* Clusters are set up with k3s at the moment. Invetigate if there is need for k8s. If the k3s doesn't have enough resources then modify the insturctions to be compatible with k8s. Liqo can be deployed both k3s and k8s clusters.
