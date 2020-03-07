### aliases
```bash
alias kc='kubectl'
alias kgp='kubectl get pods'
alias kgs='kubectl get svc'
alias kgc='kubectl get componentstatuses'
alias kctx='kubectl config current-context'
alias kgc='kubectl config get-context'
alias kswitch='kubectl config use-context'
alias knamespace='kubectl config set-context `kubectl config current-context` --namespace'

# get all services with in a cluster and the nodePorts they use (if any)
alias ksvc="kubectl get --all-namespaces svc -o json | jq -r '.items[] | [.metadata.name,([.spec.ports[].nodePort | tostring ] | join(\"|\"))] | @csv'"
# shortcuts for frequent kubernetes commands
alias kpods="kubectl get po"
alias kinspect="kubectl describe"
function krun() { name=$1; shift; image=$1; shift; kubectl run -it --generator=run-pod/v1 --image $image $name -- $@; }
function klogs() { kubectl logs $*;}
function kexec(){ pod=$1; shift; kubectl exec -it $pod -- $@; }
```

### internal structure of a resource
```bash
# get first level field types
kubectl explain pod 

# get all fields types
kubectl explain pod --recursive

# get particular field type
kubectl explain pod.spec.affinity --recursive
```

### autocomplition
```bash
# autocomplition of commands
$ kubectl g TAB
$ kubectl get
$ kubectl get de TAB
=> kubectl get deployments

# autocomplition of resource names in the current context
$ kubectl get deployment TAB TAB
=> kube-dns kube-dns-autoscaler kube-cert-manager
$ kubectl get deployment kube-d TAB
=> kubectl get deployment kube-dns
```


### short names
|  short                      | long                      | 
|:-------------------------|:---------------------------|
|csr |    certificatesigningrequests|
|cs	|componentstatuses|
|cm	|configmaps|
|ds	|daemonsets|
|deploy	|deployments|
|ep	|endpoints|
|ev	|events|
|hpa |    horizontalpodautoscalers|
|ing |ingresses|
|limits	|limitranges|
|ns	|namespaces|
|no	|nodes|
|pvc	|persistentvolumeclaims|
|pv	|persistentvolumes|
|po	|pods|
|pdb	|poddisruptionbudgets|
|psp	|podsecuritypolicies|
|rs	|replicasets|
|rc	|replicationcontrollers|
|quota	|resourcequotas|
|sa	|serviceaccounts|
|svc	|services|

### commands
* add extra columns to the output
  ```
  kubectl get pods -o wide
  ```
* save all currently deployed resources to a file
  ```
  kubectl get all --all-namespaces --export -o yaml > all-deployed-resources.yaml
  ```
* creeate all resources from a folder
  ```
  kubectl apply -f .
  ```
* get inside a running container
  ```
  kubectl get pods
  kubectl exec -it <pod-name> --container <container-name> -- /bin/bash
  ```
* set default namespace
  ```bash
  # option 0
  kubectl config set-context --current --namespace=<namespace-name>
  # option 1
  kubectl config set-context $(kubectl config current-context) --namespace=<namespace-name>
  
  # verify
  kubectl config view | grep namespace
  ```
* update image of a deployment
  ```
  kubectl set image deployment/nginx-deployment <container-name>=nginx:1.9.1 --record
  ```
* schedule pod on a specific node
  ```
  apiVersion: v1
  kind: Pod
  metadata:
    creationTimestamp: null
    labels:
      run: nginx
    name: nginx
  spec:
    nodeName: node03
    containers:
    - image: nginx
      name: nginx
      resources: {}
    dnsPolicy: ClusterFirst
    restartPolicy: Always
  status: {}
  ```
* see logs for a service
  ```
  sudo journalctl -u kube-controller-manager.service -f
  ```
## selectors
  * select all image names from all pods
    ```
    kubectl get po -o jsonpath="{range .items[*]}{range .spec.containers[*]}{.image}, {end}"
    ```
  * select all pods with name and containers in the pods:
    ```
    kubectl get po -o jsonpath="{range .items[*]} {.metadata.name} => {range .spec.containers[*]}{.image} {.name}, {end}" | tr "," "\n"
    ```
  * select all init containers
    ```
    kubectl get po -o jsonpath="{range.items[*]} {.metadata.name} => {range.spec.initContainers[*]}{.image} {.name}, {end}" | tr "," "\n"
    ```
  * get all images for pod named purple
    ```
    kubectl get po -o jsonpath="{range.items[?(@.metadata.name=='purple')]} {range .spec.containers[*]}{.image}, {end}" | tr "," "\n"
    ```
  * get pod named purple containers status
    ```
    kgp -o jsonpath="{range.items[?(@.metadata.name=='purple')]}{range.status.containerStatuses[*]}{.name} => {.state}, {.end}" | tr "," "\n"
    ```
  * get node names for pods
    ```
    kubectl get po --all-namespaces -o jsonpath="{range.items[*]} {.metadata.name} => {.spec.nodeName}, {.end}" | tr "," "\n"
    ```
  * get taints on master node
    ```
    k get node --all-namespaces -o jsonpath="{range.items[?(@.metadata.name=='master')]}{.spec.taints}, {.end}" | tr "," "\n"
    ```
  * count outcome of a kubectl command
    ```
    kubectl get clusterrole --all-namespaces | wc -l
    ```
