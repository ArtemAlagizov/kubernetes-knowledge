## maintainance
* [node draining/conrdoning](https://github.com/ArtemAlagizov/kubernetes-knowledge/blob/master/topics/maintainance.md#node-drainingconrdoning)
* [upgrade kubernetes](https://github.com/ArtemAlagizov/kubernetes-knowledge/blob/master/topics/maintainance.md#upgrade-kubernetes)
* [backup/restore](https://github.com/ArtemAlagizov/kubernetes-knowledge/blob/master/topics/maintainance.md#backuprestore)

### node draining/conrdoning
* empty the node node01 of all applications and mark it unschedulable (must be forced if node contains pods that are not part of replicaset)
  ```
  kubectl drain node01 --ignore-daemonsets
  ```
* set node to be unschedulable without killing the running apps
  ```
  kubectl cordon node01
  ```
* configure node to be schedulable again
  ```
  kubectl uncordon node01
  ```
  
[back to the top](https://github.com/ArtemAlagizov/kubernetes-knowledge/blob/master/topics/maintainance.md#maintainance)
### upgrade kubernetes 
* what is the latest stable version available for upgrade?
  ```
  kubeadm upgrade plan
  ```
* recommended way to upgrade
  * upgrade one minor version at a time until the latest stable version
* upgrade procedure
  * uprade kubeadm tool
    ```
    apt install kubeadm=1.17.0-00
    ```
  * upgrade components
    ```
    kubeadm upgrade apply v1.17.0
    ```
  * upgrade the kubelet
    ```
    apt install kubelet=1.17.0-00
    ```
    
[back to the top](https://github.com/ArtemAlagizov/kubernetes-knowledge/blob/master/topics/maintainance.md#maintainance)
### backup/restore
* three things to consider for backup (velero can help to backup a cluster deployment)
  * resources
    * store on github    
  * etcd db
  * persistance
* etcd
  * for all the etcd commands
    * specify 
      * --endpoints
      * --cacert
      * --cert
      * --key
  * possible to create a snapshot of etcd db
    ```bash
    etcdctl snapshot save snapshot.db
    etcdctl snapshot status snapshot.db #get the status
    ```
    **example**
      ```bash
      ETCDCTL_API=3 etcdctl --endpoints=https://[127.0.0.1]:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt \
     --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key \
     snapshot save /tmp/snapshot-pre-boot.db
      ```
  * to restore from db file
    ```bash
    service kube-apiserver stop #depends on etcd
    etcdctl snapshot restore snapshot.db
    system daemon-reload
    service etcd start
    service kube-apiserver start
    ```
    **example**
      ```
      ETCDCTL_API=3 etcdctl --endpoints=https://[127.0.0.1]:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt \
     --name=master \
     --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key \
     --data-dir /var/lib/etcd-from-backup \
     --initial-cluster=master=https://127.0.0.1:2380 \
     --initial-cluster-token etcd-cluster-1 \
     --initial-advertise-peer-urls=https://127.0.0.1:2380 \
     snapshot restore /tmp/snapshot-pre-boot.db
      ```

[back to the top](https://github.com/ArtemAlagizov/kubernetes-knowledge/blob/master/topics/maintainance.md#maintainance)
