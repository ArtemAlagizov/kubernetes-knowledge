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
### commands
* add extra columns to the output
  ```
  kubectl get pods -o wide
  ```
* save all currently deployed resources to a file
  ```
  kubectl get all --all-namespaces -o yaml > all-deployed-resources.yaml
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
  ```
  kubectl config set-context --current --namespace=<namespace-name>
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
