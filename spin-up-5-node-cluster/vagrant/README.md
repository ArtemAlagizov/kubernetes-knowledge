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
* as an example it is done on master-1
* generate ssh key pair 
  ```bash
  ssh-keygen
  ```
* distribute it to other nodes to easily ssh/scp to other nodes
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
#### provision a Certificate Authority that can be used to sign additional TLS certificates
* create a CA certificate, then generate a Certificate Signing Request and use it to create a private key
  ```
  # remove the following line at /etc/ssl/openssl.conf
  RANDFILE                = $ENV::HOME/.rnd
  
  openssl genrsa -out ca.key 2048
  openssl req -new -key ca.key -subj "/CN=KUBERNETES-CA" -out ca.csr
  openssl x509 -req -in ca.csr -signkey ca.key -CAcreateserial  -out ca.crt -days 1000
  ```
* generate client and server certificates for each Kubernetes component and a client certificate for the Kubernetes admin user
  * generate the admin client certificate and private key
    ```bash
    # generate private key for admin user
    openssl genrsa -out admin.key 2048

    # generate CSR for admin user
    openssl req -new -key admin.key -subj "/CN=admin/O=system:masters" -out admin.csr

    # sign certificate for admin user using CA servers private key
    openssl x509 -req -in admin.csr -CA ca.crt -CAkey ca.key -CAcreateserial  -out admin.crt -days 1000
    ```
