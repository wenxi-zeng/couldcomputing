# DHT

## Data Structure

### Ring based - Swift and Cassandra

* Physical Node
    * id
    * ip address
    * port
    * status
    * virtual nodes - set of virtual nodes

    | Id | IP Address | Port | Status | Virtual Nodes |
    |-|-|-|-|-|
    | 1 | 192.168.0.10 | 9090 | active | {15, 63} |
    | 2 | 192.168.0.11 | 9090 | active | {31} |
    | 3 | 192.168.0.12 | 9090 | inactive | {47} |

* Virtual Node
    * hash
    * physical node id

    | Hash | Physical Node Id |
    |-|-|
    | 15 | 1 |
    | 31 | 2 | 
    | 47 | 3 | 
    | 63 | 1 | 

* Routing table
    * epoch
    * virtual nodes

### Elastic DHT

* Physical Node
    * id
    * ip address
    * port
    * status
    * virtual nodes - set of virtual nodes

    | Id | IP Address | Port | Status |
    |-|-|-|-|-|
    | 11 | 192.168.0.10 | 9090 | active |
    | 22 | 192.168.0.11 | 9090 | active |
    | 33 | 192.168.0.12 | 9090 | inactive |

* Node
    * bucket
    * replicas

    | Bucket | Node Id |
    |-|-|
    | 0 | 22, 35, 86 |
    | 1 | 55, 33, 56 |
    | 2 | 22, 44, 63 |
    |..||
    | 160 | 22, 33, 4 |

* Routing table
    * epoch
    * nodes
  

### Ceph

```json
{
    "epoch" : 1539085313,
    "root" : {
        "type" : "row",
        "id" : "R01",
        "weight" : 20,
        "children" : [ {
            "type" : "cabinet",
            "id" : "C01",
            "weight" : 10,
            "children" : [ {
                "type" : "disk",
                "id" : "D01",
                "weight" : 1,
                "children": [],
                "ipAddress": "192.168.0.10"
            } ]
        } ]
    }
}
```

* Node
    * type - row, cabinet, or disk
    * id 
    * weight
    * children - set of nodes

* PhysicNode - subclass of Node
    * ipAddress

* Map
    * epoch
    * root - root node

## APIs

### Proxy (for centralized cases)

1. Get table
   
    * Input: 
        * none
    * Output: 
        * table   

2. Update table
   
    * Input: 
        * none
    * Output: 
        * updated table.

3. Load balancing

    **For elastic DHT:**
    * Input:
        * bucket - bucket needs to be moved
        * node id - node that bucket is moved to
    * Output:
        * updated table 
  
    **For Ceph:**
    * Request is directly sent to data node. Data node informs proxy to propagate the changes.

    **For others:**
    * Input:
        * type - increase or decrease
        * node id - required if type = decrease. the id of the node needs to be removed
    * Output:
        * updated table 

4. Node failure
   
    * Input:
        * node id - the id of the failed node
    * Output:
        * updated table 


5. Expand table (Elastic DHT only)
    
    * Input:
        * number of additional slots
    * Output:
        * updated table

6. Shrink table (Elastic DHT only)
    
    * Input:
        * number of shrink slots
    * Output:
        * updated table
        * message if failed

### Data node

1. Read

    * Input:
        * filename
    * Output:
        * message [file found | not found | no response]

2. Write

    * Input:
        * filename
    * Output:
        * message [success | failed | no response ]

3. Update table
    Handle update request sent from server

    * Input:
        * table - updated table
    * Output:
        * message [success | failed | no response ]

5. System info
   
    * Input: 
        * none
    * Output:
        * properties of node in string format

6. Log

    * Input:
        * on/off
    * Output:
        * none

7. Load balancing (For Ceph and Distributed cases)
   
    **For Ceph**
    * Input:
        * weight - new weight
    * Output:
        * updated map

    **For Cassandra**
    Increase/decrease the range by changing the id of this node
    * Input:
        * type - increase or decrease

    **For Elastic DHT**
    * Input:
        * bucket - bucket needs to be moved
        * node id - node that bucket is moved to
    * Output:
        * updated table

8. Expand table (Distributed Elastic DHT only)
    
    * Input:
        * number of additional slots
    * Output:
        * updated table

9.  Shrink table (Distributed Elastic DHT only)
    
    * Input:
        * number of shrink slots
    * Output:
        * updated table
        * message if failed 

### Configuration

* Number of hash slots
* Number of replicas
* Proxy (if centralized case)
* Data nodes
* Port
* Request timeout