## installation

### cases
* high performance => ssd backed storage
* multiple concurrent connections => network based storage
* shared access across multiple pods => persistent shared volumes
* label nodes with specific disk types
  * use node selectors to assign applications to nodes with specific disk types

### nodes
* virtual or physical machines
* minimum of 4 nodes cluster (m3.medium on aws)
* master nodes preferably shouldn't host workloads (taint)

### installation options
* minikube
  * deploys VMs
  * single node cluster
* kubeadm
  * requires VMs to be ready
  * single/multi node cluster
  
### hosted vs. tunable solutions
* hosted
  * managed kubernetes cluster
    * EKS
    * GKE
    * openshift online
* tunable
  * scripted deployment/management of kubernetes cluster, makes it easy to deploy kubernetes cluster privately within an organization
    * openshift
    * cloud foundry container runtime
    * vmware cloud pks
    * vargant
