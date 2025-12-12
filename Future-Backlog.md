# Future backlog
* Succesfully run Oss-mlops-platform pipeline with multicloud pod offloading
  - Check failed pod error messages from pod logs. This could possibly be a notebook problem? the new pod was trying to connect to a pod with an address that had no pod in it. Maby its hardcoded somewhere?
  - Investigate and solve possible dependency problems with the external vm.
* MVP example of GPU recource sharing.
  - Get and install correct drivers and device plugins so k3s or k8s detects available gpu recources
  - Assert available gpu recources with liqo so the consumer can detect providers resources via the created virtual node.
* Tailor startup and setup scripts for vm that hosts external cluster.
* Investigate how Auto Scaling can be added to the system. This can include:
  - How does the system detect and decide when to scale?
  - Provisioning and creating new VM´s upon gpu / cpu load requirements.
  - Renting a Vm instance with automaticly scaling recourses by scaling and limiting the computing capacity of nodes. This can maby be done with Liqo ResourceSlice virtual nodes on consumer. You can assert the virtual nodes computing capacity even though truely capacity is greater.
  - A naming system for multiple VM´s, clusters, nodes and CIDR provisioning for non-overlapping addresses with different machines and clusters.
* Investigate how to add automatic shut down and deletion for joined VM after determined time of provider node idle.
  - Do all created Vm´s have the same time of idle or does the potential idle time scale with somehow with the amount of instances active and idling.
  - Should the shutdown time be easily customizable by mlops user / admin
* Investigate how to create a safe de-coupling and uninstalling script of the joined clusters to ensure succesful rejoining of clusters. (liqo specific problem).
* Remote cluster is set up with k3s at the moment. Investigate if there is need for k8s. If the k3s doesn't have enough resources then modify the instructions to be compatible with k8s. Liqo can be deployed both k3s and k8s clusters.
