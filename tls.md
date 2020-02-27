## TLS

### procedure
* actors
  * CertificateAuthority
  * server
  * admin
* steps
  * admin generates a key pair to establishe ssh connection to the server
  * server generates a key pair for securing the website
  * to encrypt http traffic server sends verification request with certificate to the CA
  * CA signs the certificate using their own private key and sends it back to the server
  * server configures the web app on the server to use the certificate
  * when request comes from the browser server sends the certificate
  * browser fetches public key from CA and verifies the quiality of the certificate
  * browser generates a symmetric key that it wishes to use for the further communication
  * browser fetches server public key and encrypts the symmetric key and sends it to the server
  * server decrypts the message and stores the symmetric key for further communication
  * optionally client can generate a key-pair, verifty it with CA and send it to the server
### types of certificates
* root 
  * configured at CA servers
* client
  * configured on clients
    * admin (admin.crt, admin.key)
    * kube-proxy (proxy.crt, proxy.key)
    * kube-scheduler (scheduler.crt, scheduler.key)
    * kube-controller-manager (controller-manager.crt, controller-manager.key)
* server
  * configured on servers
    * kube-apiserver (apiserver.crt, apiserver.key)
      * is a client of etcd as this is the only thing talking to etcd server
    * etcd server (etcd.crt, etcd.key)
    * kubelet server (kubelet.crt, kubelet.key)
    
 ### certificate creation
 * create certificate and key pair for CA using openssl
   * generate keys 
     ```
     openssl genrsa -out ca.key 2048
     ```
   * certificate signing request
     ```
     openssl req -new -key ca.key  -subj "/CN=KUBERNETES-CA" -out ca.csr
     ```
   * sign certificates
     ```
     openssl x509 -req -in ca.csr -signkey ca.key -out ca.crt
     ```
* create certificate and key pair for admin and CA certificate using openssl
  * generate keys 
    ```
    openssl genrsa -out admin.key 2048
    ```
  * certificate signing request
    ```
    openssl req -new -key ca.key  -subj "/CN=kube-admin/O=system:masters" -out admin.csr
    ```
  * sign certificates with CA, which makes it a vlid certifiate within the cluster
    ```
    openssl x509 -req -in -CA admin.csr -CAKey ca.key ca.key -out admin.crt
     ```
* in the same way create certificate and key pair for contol plane components
  * /CN=system-kube-scheduler
  * /CN=system-kube-control-manager
  * /CN=system-kube-proxy
* create certificate for server in the same way
  * for kube-api server extra params to pass is openssl.cnf, containing aliases and ip
  * for kubelet each node must generate its' own certificate (kubelet.yaml)
    * also generate certificate for communicating with kube-api server
      * use naming of system:node:node01
      * also add nodes to GROUP system:group

### view certificates
* get kube-api server certificates details
  ```
  openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout
  ```
  * CN
  * alternative names
  * issuer
  * expiration date
### check logs
* natively
  ```
  journalctl -u etcd.service -l
  ```
* in a pod
  ```
  kubectl logs etcd-master
  ```
