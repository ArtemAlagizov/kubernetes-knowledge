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

### high availability
* kube-api server
  * can be ran in active-active mode
  * in case of multiple master nodes it is required to have a load balancer in front of master nodes, so that requests are split between the master nodes
* controller manager and scheduler
  * should be ran in active-standby mode
    * must not run in parallel to avoid request duplication
    * achieved through leader-election process
      ```bash
      # kube-controller tries to become a leader every 2 seconds, in case other master node crashes
      # current leader renews the lease (leadership) evrey 15s
      kube-controller-manager --leader-elect true --leader-elect-retry-period 2s --leader-elect-lease-duration 15s
      ```
      * scheduler follows the same approach
* etcd
  * can be part of master node (stacked control plane topology) (eaasier to manage/setup)
  * can be ran on a seprate servers (external etcd topology) (more reliable)
    * kube-api server accepts list of etcd servers
      ```
      kube-apiserver --etcd-servers=https://10.240.0.10:2379,https://10.40.0.11:2379
      ```
    * kube-api server can read/write to any of the etcd servers, but write will be delegated to the node that is responsible for writing
      * responsible node is elected through leader-election process
    
### etcd
* etcd => distributed reliable key-value store that is simple, secure and fast
* "write" is done by etcd cluster leader
  * leader election done with [RAFT](https://raft.github.io/) algorithm
    * after a random time a node sends request to other nodes for permission to become the leader (starts election)
    * after getting approvals he becomes the leader
    * the leader sends notifications to other nodes periodically that he is still the leader
    * in case there is no notification in time the re-election starts amongst the rest of the nodes
    * questions
       * what happens if the old leader comes back online
         * leaders can step down if there is a better leader present (majority votes)
         * when coming online node is in the "follower" state
       * what happens with network partition
         * each network will elect leader
         * in case of network merge one leader will step down based on majority votes
  * "write" is considered successful only when majority nodes are notified of the "write"
  * quorum = floor(n/2 + 1)
    * minimum number of nodes that should be online for a cluster to function properly
    * for 1 node quorum is 1
      * if it goes down everything is lost
    * for 2 nodes quorum is 2
      * if 1 fails "writes" will not be processed as there is no quorum
      * that is why recommended minimum is 3 nodes
        * gives fault tolerance of at least 1 node (number of nodes that you can afford to loose while keeping the cluster alive)
  * it is recommended to have odd number of master nodes in a cluster
* etcd installation
  ```
  TODO
  ```
* etcdctl
  * set version
    ```
    export ETCDCTL_API=3
    ```
  * write data
    ```
    etcdctl put name jimmy
    ```
  * read data
    ```
    etcdctl get name
    ```
  * get all the keys
    ```
    etcdctl get / --prefix --keys-only
    ```
