# DHT

## Data Structure

### Ring based - Swift and Cassandra

* Node
    * id
    * ipAddress
    * successors - a set of IDs

* Routing table
    * epoch
    * nodes

```json
{
    "epoch" : 1539085313,
    "nodes" : [ {
        "id" : 1,        
        "ipAddress" : "192.168.0.10",
        "successors" : [ 20, 33 ] 
    }, {
        "id" : 20,        
        "ipAddress" : "192.168.0.11",
        "successors" : [ 23, 43 ] 
    }, {
        "id" : 33,        
        "ipAddress" : "192.168.0.12",
        "successors" : [ 43, 54 ]  
    } ]
}
```

| Node Id | IP Address | Successors |
|-|-|-|
| 1 | 192.168.0.10 | {20, 33} |
| 20 | 192.168.0.11 | {33, 43} |
| 33 | 192.168.0.12 | {43, 54} |

### Elastic DHT

* Node
    * bucket
    * id
    * ipAddress
    * replicas

* Routing table
    * epoch
    * nodes
  
```json
{
    "epoch" : 1539085313,
    "nodes" : [ {
        "bucket" : 15,
        "id" : 1,        
        "ipAddress" : "192.168.0.10",
        "successors" : [ 20, 33 ] 
    }, {
        "bucket" : 31,
        "id" : 20,        
        "ipAddress" : "192.168.0.11",
        "successors" : [ 23, 43 ] 
    }, {
        "bucket" : 47,
        "id" : 33,        
        "ipAddress" : "192.168.0.12",
        "successors" : [ 43, 54 ] 
    },
    ...
    {
        "bucket" : 160,
        "id" : 1,        
        "ipAddress" : "192.168.0.10",
        "successors" : [ 20,33 ] 
    } ]
}
```

| Bucket | Node Id | IP Address | Replicas |
|-|-|-|-|
| 15 | 1 | 192.168.0.10 | {20, 33} |
| 31 | 20 | 192.168.0.11 | {33, 43} |
| 47 | 33 | 192.168.0.12 | {43, 54} |
|..||||
| 160 | 1 | 192.168.0.10 | {20, 33}|

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
* Internal port. Port for internal communication between nodes
* External port. Port for client access
* Request timeout