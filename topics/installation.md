## installation

### cases
* high performance => ssd backed storage
* multiple concurrent connections => network based storage
* shared access across multiple pods => persistent shared volumes
* label nodes with specific disk types
  * use node selectors to assign applications to nodes with specific disk types

### nodes
* virtual or physical machines
* minimum of 4 nodes cluster (m3.medium on aws)
* master nodes preferably shouldn't host workloads (taint)
