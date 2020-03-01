## networking
### linux routing basics
  * to see current routes use **route**
  * to list and modify interfaces on the host
    ```
    ip link
    ```
  * to see ip addresses assigned to those interfaces
    ```
    ip addr
    ```
  * set ip address on an interface
    ```
    ip addr add 192.168.1.10/24 dev eth0
    ```
  * to add a route
    ```
    ip route add 192.168.2.0/24 via 192.168.1.1
  * for systems to communicate with each other routing through gateways needs to be configured on all systems
  * to set a default router through which all the outgoing requests will go
    ```
    ip route add default via 192.168.2.1
    ```
  * to enable packet forwarding
    ```
    vi /proc/sys/net/ipv4/ip_forward
    # 0 is default => no forwarding
    ```
  * to persist packet forwarding
    ```
    vi /etc/sysctl.conf
    # net.ipv4.ip_forward = 1
    ```
### linux dns basics
* point host to dns server
  ```
  vi /etc/resolv.conf
  # nameserver 192.168.1.100
  # nameserver 8.8.8.8
  # search mysite.com prod.mysite.com => will search for host names appending the search entries
  ```
* resolver first checks /etc/hosts and if doesn't find an entry there goes to nameserver
  * order can be changed in **/etc/nsswitch.conf** **hosts:**
    ```
    hosts:          files mdns4_minimal [NOTFOUND=return] dns
    ```
* record types on dns server
  * A => ip to host names
  * AAAA => ip6 to host names 
  * CNAME => mapping one name to another name
* tools to test dns
  * ping
  * nslookup => queries dns server
  * dig
### network namespaces
* create new network namespace 
  ```
  ip netns add <namespace-name>
  ```
* list network namespaces
  ```
  ip netns
  ```
* execute a command inside a namespace
  ```
  ip netns exec <namespace-name> <command, ie ip link>
  # or
  ip -n <namespace-name> link
  ```
* connect 2 namespaces (red and blue) by virtual cable
  ```bash
  # create virtual cable
  ip link add veth-red type veth peer name veth-blue
  
  # attach cable ends to namespaces
  ip link set veth-red netns red
  ip link set veth-blue netns blue
  
  # assign ip addresses in these namespaces
  ip -n red addr add 192.168.15.1 dev veth-red
  ip -n blue addr add 192.168.15.2 dev veth-blue
  
  # bring up the interfaces within the 2 namespaces
  ip -n red link set veth-red up
  ip -n blue link set veth-blue up
  ```
* connect 
 
