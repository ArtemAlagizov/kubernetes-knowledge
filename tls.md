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
* server
  * configured on servers
