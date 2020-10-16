
# Secret/ConfigMap Mount Dashboard for Jenkins
## Prerequisites
1. [Pipeline Utility Steps Plugins](https://plugins.jenkins.io/pipeline-utility-steps/)
2. [Blueocean](https://plugins.jenkins.io/blueocean/) (For GUI)
3. Kubernetes/Openshift **serviceaccount Token** or **Openshift User-Pass** to add it to Jenkins Credentials.
## How to use Script
- First add credentials to Jenkins Credentials.
	- **serviceaccount Token:** use **Secret Text Kind** and add **Token** to **Secret box**
	-  **Openshift User-Pass:** use **Username with Password Kind** and then add **Username and Password** to the related boxes.
<p align="center">
  <img width="780" height="251" src="https://user-images.githubusercontent.com/59168275/96262466-7c690580-0fca-11eb-98b3-d8986ec40191.png">
</p>

- Create new pipeline and use Jenkinsfile inside this repository. (In "InCluster" folder there is another Jenkinsfile that can be used to mount Secret/ConfigMap that reside in In Cluster)
- After starting Pipeline you will see Input Table.
<p align="center">
  <img width="466" height="1198" src="https://user-images.githubusercontent.com/59168275/96261685-6f97e200-0fc9-11eb-9aff-b0e232bd645a.png">
</p>

- The Dashboard itself is self explanatory.
- You can choose different mount types and different resources.
