## spin up 5 node cluster locally [based on [this](https://github.com/mmumshad/kubernetes-the-hard-way)]
### setup
2 masters, 2 workers, 1 loadbalancer

### prerequisites
  * virtual box installed
  * vagrant installed

### steps
#### provision VMs
```bash
cd spin-up-5-node-cluster/vagrant
vargant up
```
#### install tools to perform administrative tasks, such as creating certificates, kubeconfig files and distributing them to the different VMs
* generate ssh key pair for master-1
  ```bash
  # generate key pair for master-1
  ssh-keygen
  ```
* distribute it to other nodes
  ```
  vargant@master-1:~$ ssh master-2
  cat >> ~/.ssh/authorized_keys <<EOF
  ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQD......8+08b vagrant@master-1
  EOF
  ```
* install kubectl on master-1
  ```bash
  wget https://storage.googleapis.com/kubernetes-release/release/v1.13.0/bin/linux/amd64/kubectl
  chmod +x kubectl
  sudo mv kubectl /usr/local/bin/
  
  # verify installation
  kubectl version --client
  ```
