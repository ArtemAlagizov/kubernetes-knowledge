### aliases
```
alias kc='kubectl'
alias kgp='kubectl get pods'
alias kgs='kubectl get svc'
alias kgc='kubectl get componentstatuses'
alias kctx='kubectl config current-context'
alias kcon='kubectl config use-context'
alias kgc='kubectl config get-context'
```
### commands
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
