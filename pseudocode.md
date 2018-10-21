# Pseudo Code

## Ring

### Decrease load

```java

decreaseLoad(dh, node) {
    hi = node.getHash();
    hf = hi - dh;

    if (notValid(hf, node)) return;

    toNode = LookupTable.get(node.getIndex() + NUMBER_OF_REPLICAS);
    transfer(hf, hi, node, toNode);   // transfer(start, end, from, to). (start, end]

    node.setHash(hf);
    LookupTable.update(); // commit the change, gossip to other nodes
}

```

### Increase load

```java
increaseLoad(dh, node) {
    hi = node.getHash();
    hf = hi + dh;

    if (notValid(hf, node)) return;

    fromNode = LookupTable.get(node.getIndex() + NUMBER_OF_REPLICAS);
    requestTransfer(hi, hf, fromNode, node); // requestTransfer(start, end, from, to). (start, end]

    node.setHash(hf);
    LookupTable.update(); // commit the change, gossip to other nodes
}

```

### Node join

```java
nodeJoin(node) {
    index = LookupTable.find(node); // where the new node is inserted to
    LookupTable.addNode(index, node); // only add the node to table, not gossiping the change yet

    successor = LookupTable.next(node);     
    startNode = LookupTable.get(index - NUMBER_OF_REPLICAS);
    endNode = LookupTable.next(startNode);    

    for (i = 0; i < NUMBER_OF_REPLICAS; i++) {        
        hi = startNode.getHash();
        hf = endNode.getHash();

        requestTransfer(hi, hf, successor, node); // requestTransfer(start, end, from, to). (start, end]

        startNode = endNode;
        endNode = LookupTable.next(startNode);
        successor = LookupTable.next(successor);
    }
    
    LookupTable.update(); // commit the change, gossip to other nodes
}

```

### Node leave/failure

```java
nodeLeave(node) {
    index = LookupTable.find(node); // where the new node is inserted to
    LookupTable.remove(index); // only remove from table, not gossiping the change yet

    successor = LookupTable.get(index);  
    predecessor = LookupTable.pre(successor);   
    startNode = LookupTable.get(index - NUMBER_OF_REPLICAS);
    endNode = LookupTable.next(startNode);    

    for (i = 0; i < NUMBER_OF_REPLICAS; i++) {        
        hi = startNode.getHash();
        hf = endNode.getHash();

        requestReplication(hi, hf, predecessor, successor); // requestTransfer(start, end, from, to). (start, end]

        startNode = endNode;
        endNode = LookupTable.next(startNode);
        predecessor = succesor;
        successor = LookupTable.next(successor);
    }
    
    LookupTable.update(); // commit the change, gossip to other nodes
}
```

## Ceph

### Change weight

```java
changeWeight(weight, node) {
    node.setWeight(weight);
    ClusterMap.update(); // not only update the node, 
                         // but cascade the change up to the root.
                         // notify proxy after updated.
}
```

### Add node

```java
addNode(node, cluster) {
    cluster.add(node);
    ClusterMap.update(); // not only update the node, 
                         // but cascade the change up to the root.
                         // notify proxy after updated.
}
```

### Node removal and failure 

```java
markAsDown(node) {
    node.setStatus(STATUS_INACTIVE);
    ClusterMap.update(); // propagate new map to datanodes.
}
```

### Handle weight change and node addition

```java
onMapUpdate(newMap){
    transferList = new HashMap<>(); // list for weight change and node addition

    for (pg : LocalPGCache.getList()) {
        node = Rush(pg.getId(), pg.getR(), newMap);
        
        // Rush returns a different node than current,
        // because of weight change and node addition.
        if (node != currentNode) {
            // in case the node is failed.
            while (node.getStatus() != STATUS_ACTIVE) {
                pg.R++;                
                node = Rush(pg.getId(), pg.getR(), newMap);
            }

            transferList.get(node).add(pg);
        }
    }
    
    for (node : transferList) {
        transfer(transferList.get(node), node);
    }
}
```

### Handle node removal and failure

```java
onNodeFailure(failedNode, newMap){
    replciationList = new HashMap<>(); // where the new replicas should be sent

    // try every PG stored in this node.
    for (pg : LocalPGCache.getList()) {

        // starting from R = 0
        count = 0;
        r = 0;

        while (count < NUMBER_OF_REPLICAS) {
            node = Rush(pg.getId(), r++, newMap);            
            
            if (node == failedNode) {
                do {                
                    node = Rush(pg.getId(), r++, newMap);
                } while (node.getStatus() != STATUS_ACTIVE)

                replciationList.get(node).add(pg);
                break;
            }

            count++;
        }
    }
    
    for (node : replciationList) {
        replicate(replciationList.get(node), node);
    }
}
```

| PG Id | r |
|-|-|
| PG01 | 0 |
| PG02 | 0 |
| PG08 | 1 |
| PG10 | 2 |
| ... | ... |