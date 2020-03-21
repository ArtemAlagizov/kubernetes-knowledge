### create static pod on worker node
* check kubelet status
  ```
  sudo service kubelet status
  ```
* get config.yaml location
* check config.yaml for staticPodPath
* set it to /etc/kubernetes/manifests
### create a user  with permissions to update pods in a namespace
* create csr object
  * get base64 representaton of csr
    ```
    cat user.csr | base 64 | tr -d '\n' 
    ```
* create role
* create role-binding
### check dns resolution of pod and service
* create pod
  ```
  kubectl run --generator=run-pod/v1 nginx --image=nginx
  ```
* expose pod
  ```bash
  kubectl expose pod nginx --port=80 --name nginx-service --dry-run -o yaml > nginx.service.yaml
  
  # specify nodePort afterwards and create the service
  kubectl apply -f nginx.service.yaml
  ```
* run the following to check dns
  ```
  kubectl run --generator=run-pod/v1 nslookup --image:busybox:1.28 --rm -it -- nslookup <service-name> > nginx.svc
  kubectl run --generator=run-pod/v1 nslookup --image:busybox:1.28 --rm -it -- nslookup <dashed-pod-ip>.<namespace>.pod > nginx.pod
  ```
### create deployment, apply rolling update, record it
* create deployment
  ```
  kubectl create deployment --image=nginx
  ```
* apply roling update
  ```
  kubectl set image deployment/nginx-deployment nginx=nginx:1.9.1 --record
  ```
### create serviceaccount that has access to list all persistentvolumes
* create service account
  ```
  kubectl create serviceaacount pvviewer
  ```
* create cluster role to list persistent volumes
  ```
  kubectl create clusterrole pvviewer-role --resource=persistentvolumes --verb=list
  ```
* create cluster role binding
  ```
  kubectl create clusterrolebinding pvviewr-role-binding --clusterrole=pvviewer-role --serviceaccount=default:pvviwer
  ```
* specify service account in a pod
  ```
  apiVersion: v1
  kind: Pod
  metadata:
    name: nginx
  spec:
    containers:
    - image: nginx
      name: nginx
      volumeMounts:
      - mountPath: /var/run/secrets/tokens
        name: vault-token
    serviceAccountName: build-robot
  ```
* verify
  ```bash
  kubectl describe pod nginx
  
  ...
  Volumes:
    pvviewer-token-6m8n9:
      Type:        Secret (a volume populated by a Secret)
      SecretName:  build-robot-token-6m8n9 # <= this indicates correct use of the service account
      Optional:    false
  ```
### list all internal ip addresses of nodes with node names
  ```
  kubectl get nodes -o jsonpath='{range.items[*]} {.status.addresses[?(@.type=="InternalIP")].address} of {.metadata.name}'
  ```
### allow incoming connections from all pods to a pod without modifying existing deployments
* create an ingress network policy
  ```
  apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata:
    name: ingress-to-nptest
  spec:
    podSelector:
      matchLabels:
        run: np-test-1
    ingress:
    - from:
      - podSelector: {}
      ports:
      - protocol: TCP
        port: 80
  ```
* verify 
  ```bash
  kubectl run --generator=run-pod/v1 busybox --image=busybox --rm -it -- sh
  
  # inside the container
  nc -zvw 2 np-test-service 80
  
  # expected output
  np-test-service (10.110.165.96:80) open
  ```
### taint node with key:env_type, value:production and effect:NoSchedule, create pod woth tolerations to run on this node
* taint node
  ```
  kubectl taint node node01 env_type=production:NoSchedule
  ```
* create matching toleration for pod
  ```
  apiVersion: v1
  kind: Pod
  metadata:
    creationTimestamp: null
    labels:
      run: prod-redis
    name: prod-redis
  spec:
    containers:
    - image: redis:alpine
      imagePullPolicy: IfNotPresent
      name: prod-redis
      resources: {}
    tolerations:
    - key: "env_type"
      operator: "Equal"
      value: "production"
      effect: "NoSchedule"
    dnsPolicy: ClusterFirst
    restartPolicy: Always
  status: {}
  ```
* verify
  ```
  kubectl get pod -o wide
  ```
### create pod with labels and in hr namespace
  ```bash
  # create namespace
  kubectl create namespace hr
  
  # create pod 
  kubectl run --generator=run-pod/v1 hr-pod --image=redis:alpine --labels=environment=production,tier=frontend --dry-run -o yaml > 7-hr-pod.yaml
  
  # add namespace to the pod metadata
  apiVersion: v1
  kind: Pod
  metadata:
    creationTimestamp: null
    labels:
      environment: production
      tier: frontend
    name: hr-pod
    namespace: hr
  spec:
    containers:
    - image: redis:alpine
      imagePullPolicy: IfNotPresent
      name: hr-pod
      resources: {}
    dnsPolicy: ClusterFirst
    restartPolicy: Always
  status: {}
  
  # deploy pod
  kubectl apply -f 7-hr-pod.yaml
  ```
### fix kubeconfig file
  ```bash
  # verify that host and port for kube-apiserver are correct
  kubectl cluster-info --kubeconfig=/root/super.kubeconfig
  
  # check correct port for kube-api
  kubectl get pod -n kube-system kube-apiserver-master -o yaml
  
  # update the config file
  ...
  clusters:
  - cluster:
      server: https://172.17.0.14:6443
  ...
  
  # verify
  kubectl cluster-info --kubeconfig=/root/super.kubeconfig
  # should show correct info now
  ```
### troubleshoot deployment not scaling after scale command
* check control plane pods 
  ```
  kubectl get pods -n kube-system
  ```
* fix any issues with static control pods
  ```
  cd /etc/kubernetes/manifests
  ```
* for instance, replace wrong entries in kube-controller-manager.yaml
  ```
  sed -i 's/kube-contro1ler-manager/kube-controller-manager/g' kube-controller-manager.yaml
  ```
### upgrade the current version of kubernetes from 1.16 to 1.17.0 exactly using the kubeadm utility. make sure that the upgrade is carried out one node at a time starting with the master node. to minimize downtime, the deployment gold-nginx should be rescheduled on an alternate node before upgrading each node
* master node
  ```bash
  kubectl drain node master --ignore-daemonsets
  apt-get install kubeadm=1.17.0-00
  kubeadm  upgrade plan v1.17.0
  kubeadm  upgrade apply v1.17.0
  apt-get install kubelet=1.17.0-00
  
  # make master node schedulable
  kubectl uncordon master
  kubectl drain node01 --ignore-daemonsets
  ```
* node01
  ```
  apt-get install kubeadm=1.17.0-00
  kubeadm upgrade node --kubelet-version v1.17.0
  apt-get install kubelet=1.17.0-00
  ```
* master
  ```bash
  kubectl uncordon node01
  
  # make sure this is scheduled on master node
  kubectl get pods -o wide | grep gold
  ```
### print the names of all deployments in the deployment_1 namespace in the following format: DEPLOYMENT CONTAINER_IMAGE READY_REPLICAS NAMESPACE \n \<deployment name\> \<container image used\> \<ready replica count\> \<Namespace\>. the data should be sorted by the increasing order of the deployment name
```
kubectl -n deployment_1 get deployment -o custom-columns=DEPLOYMENT:.metadata.name,
CONTAINER_IMAGE:.spec.template.spec.containers[].image,
READY_REPLICAS:.status.readyReplicas,
NAMESPACE:.metadata.namespace --sort-by=.metadata.name > /opt/deployment_1_data
```
### expose the hr-web-app as service hr-web-app-service application on port 30082 on the nodes on the cluster. the web application listens on port 8080
```
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: hr-web-app
  name: hr-web-app-service
spec:
  ports:
  - port: 8080
    protocol: TCP
    targetPort: 8080
    nodePort: 30082
  selector:
    app: hr-web-app
  type: NodePort
status:
  loadBalancer: {}
```
### create a Persistent Volume with the given specification. 
* volume Name: pv-analytics
* storage: 100Mi
* access modes: ReadWriteMany
* host Path: /pv/data-analytics
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-analytics
spec:
  capacity:
    storage: 100Mi
  accessModes:
  - ReadWriteMany
  hostPath:
    path: /pv/data-analytics
```
### create a new ConfigMap named cm-3392845. use the spec: 
* configMap name: cm-3392845
* data: DB_NAME=SQL3322
* data: DB_HOST=sql322.mycompany.com
* data: DB_PORT=3306
```
kubectl create configmap cm-3392845 --from-literal=DB_NAME=SQL3322 --from-literal=DB_HOST=sql322.mycompany.com --from-literal=DB_PORT=3306
```
### create a new Secret named db-secret-xxdf:
* Secret Name: db-secret-xxdf
* Secret 1: DB_Host=sql01
* Secret 2: DB_User=root
* Secret 3: DB_Password=password123
```
kubectl create secret generic db-secret-xxdf --from-literal=DB_Host=sql01 --from-literal=DB_User=root --from-literal=DB_Password=password123
```
### create a Persistent Volume with the given specification 
* volume name: pv-analytics
* storage: 100Mi
* access modes: ReadWriteMany
* host path: /pv/data-analytics
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-analytics
spec:
  hostPath:
    path: /pv/data-analytics
  capacity:
    storage: 100Mi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Recycle
```
### get the Common Name (CN) configured on the Kube API Server Certificate
```
openssl x509 -in file-path.crt -text -noout
```
### sign new certificate by the CA for etcd and configure it to be used by the kube-api server
```
openssl x509 -req -in /etc/kubernetes/pki/apiserver-etcd-client.csr -CA /etc/kubernetes/pki/etcd/ca.crt -CAkey /etc/kubernetes/pki/etcd/ca.key -CAcreateserial -out /etc/kubernetes/pki/apiserver-etcd-client.crt
```
### deployment doesn't scale, fix issue
check:
* state of control plane components
  ```
  kubectl get pod -n kube-system
  ```
* if some component is not running
  ```
  kubectl logs -n kube-system kube-controller-manager-master
  ```
  * possible issues:
    * wrong volumes mounted
    * wrong executable name
      * to solve check /etc/kubernetes/manifests
### a new deployment called alpha-mysql has been deployed in the alpha namespace. however, the pods are not running. troubleshoot and fix the issue. the deployment should make use of the persistent volume alpha-pv to be mounted at /var/lib/mysql and should use the environment variable MYSQL_ALLOW_EMPTY_PASSWORD=1 to make use of an empty root password. _important: do not alter the persistent volume_


### Create a pod called secret-1401 in the admin1401 namespace using the busybox image. The container within the pod should be called secret-admin and should sleep for 4800 seconds. The container should mount a read-only secret volume called secret-volume at the path /etc/secret-volume. The secret being mounted has already been created for you and is called dotfile-secret.
```
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: secret-1401
  name: secret-1401
  namespace: admin1401
spec:
  containers:
  - image: busybox
    name: secret-admin
    resources: {}
    command: ["sleep","4800"]
    volumeMounts:
    - name: secret-volume
      readOnly: true
      mountPath: /etc/secret-volume
  volumes:
  - name: secret-volume
    secret:
      secretName: dotfile-secret
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```
