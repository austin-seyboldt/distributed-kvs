# Distributed, Replicated, Causally/Eventually Consistent Key-value store
### Contributors: Austin Seyboldt, John Abendroth, Richard Thai, Matt Ngo
## Description
This project is an implementation of a distributed key-value store, which guarantees eventual consistency, as well as causal
consistency on a per-client basis. The system can be configured to partition keys among many shards and to replicate those
keys across a given number of nodes within each shard to provide durability and fault tolerance. Consistency is achieved
through a gossip protocol in which a replica attempts to exchange keys with every other replica every 500ms. Requests can be 
sent to any node in the system and the node will attempt to handle or forward requests if it is able to (which may not be 
possible if there's a network partition). A node will reject a request if it would violate causality (ie: the node hasn't 
received an updated version of the key yet).  
  
This program was written for a systems class. Much of the following text was taken from a TA document describing the usage.
  
## Building
Docker Network Management

Container Options and Environment:

- ADDRESS – a required environment variable that provides the address of the node being started  
- VIEW – a required environment variable that provides the address of each node in the store  
- REPL_FACTOR – a required environment variable that provides the replication factor, or number of replicas, for each shard.  
- ip – a required container property that assigns the given IP address to the container  
- p – a required container property that binds the given host port to the given container port  
- net – a required container property that connects the container to the given network  
- name – a convenience property so that we can distinguish between containers using a human readable name  

The following is a scenario where we have four nodes, and a Docker subnet named kv_subnet.  

```
$ docker network create --subnet=10.10.0.0/16 kv_subnet

    Build Docker image containing the key-value store implementation:

$ docker build -t kvs:4.0 

Run four instances at IP's 10.10.0.2, 10.10.0.3, 10.10.0.4 and 10.10.0.5, and listening to port 13800:

$ docker run -p 13800:13800                                                            \
             --net=kv_subnet --ip=10.10.0.2 --name="node1"                             \
             -e ADDRESS="10.10.0.2:13800"                                              \
             -e VIEW="10.10.0.2:13800,10.10.0.3:13800,10.10.0.4:13800,10.10.0.5:13800" \
             -e REPL_FACTOR=2                                                          \
             kvs:4.0

$ docker run -p 13801:13800                                                            \
             --net=kv_subnet --ip=10.10.0.3 --name="node2"                             \
             -e ADDRESS="10.10.0.3:13800"                                              \
             -e VIEW="10.10.0.2:13800,10.10.0.3:13800,10.10.0.4:13800,10.10.0.5:13800" \
             -e REPL_FACTOR=2                                                          \
             kvs:4.0

$ docker run -p 13802:13800                                                            \
             --net=kv_subnet --ip=10.10.0.4 --name="node3"                             \
             -e ADDRESS="10.10.0.4:13800"                                              \
             -e VIEW="10.10.0.2:13800,10.10.0.3:13800,10.10.0.4:13800,10.10.0.5:13800" \
             -e REPL_FACTOR=2                                                          \
             kvs:4.0

$ docker run -p 13803:13800                                                            \
             --net=kv_subnet --ip=10.10.0.5 --name="node4"                             \
             -e ADDRESS="10.10.0.5:13800"                                              \
             -e VIEW="10.10.0.2:13800,10.10.0.3:13800,10.10.0.4:13800,10.10.0.5:13800" \
             -e REPL_FACTOR=2                                                          \
             kvs:4.0
```

## Usage
### Endpoints
- /kvs/keys/<key> ==> GET/PUT requests on keys
- /kvs/key-count ==> GET
- /kvs/shards ==> GET
- /kvs/shards/<shard-id> ==> GET
- /kvs/view-change ==> PUT

### Administrative Operations

#### GET key count for a node and the stored replicas

- To get the total number of keys stored by a node, the shards it stores, and the number of keys per shard, send a GET request to the endpoint, **/kvs/key-count** at any node.

    - On success, the response should have status code 200. This example sends the request to node1.

    ```bash
    $ curl --request   GET                                        \
           --header    "Content-Type: application/json"           \
           --write-out "%{http_code}\n"                           \
           http://127.0.0.1:13800/kvs/key-count
    
           {
               "message"       : "Key count retrieved successfully",
               "key-count"     : 4,
               "shard-id"      : "1",
           }
           200
    ```

#### GET ID for each shard

- To get information for all shards, send a GET request to the endpoint, **/kvs/shards** at any node. The response should contain the id of each shard.

    - On success, the response should have status code 200.

    ```bash
    $ curl --request   GET                                        \
           --header    "Content-Type: application/json"           \
           --write-out "%{http_code}\n"                           \
           http://127.0.0.1:13800/kvs/shards
    
       {
               "message"       : "Shard membership retrieved successfully",
               "shards"        : ["1", "2"]
           }
           200
    ```

#### GET information for a specific shard

- To get the number of keys stored by a shard and what node each replica is stored on, send a GET request to the endpoint, **/kvs/shards/\<shard-id\>** at any node.

    - On success, the response should have status code 200.

    ```bash
    $ curl --request   GET                                        \
           --header    "Content-Type: application/json"           \
           --write-out "%{http_code}\n"                           \
           http://127.0.0.1:13800/kvs/shards/1
    
       {
               "message"       : "Shard information retrieved successfully",
               "shard-id"      : "1",
               "key-count"     : 4,
               "replicas"      : ["10.10.0.2:13800", "10.10.0.3:13800"]
           }
           200
    ```

#### PUT request for view change

- View changes operate under the assumption that the views are valid and all nodes are alive and able to communicated at the time of a view change.  

- To change the view, or add newly started nodes to the key-value store, send a PUT request to the endpoint, /kvs/view-change, with a JSON payload containing the list of addresses in the new view and the replication factor.
    - On success, the response will have status code 200 and JSON:

    ```bash
    $ curl --request   PUT                                                                                          \
           --header    "Content-Type: application/json"                                                             \
           --write-out "%{http_code}\n"                                                                             \
           --data      '{"view":"10.10.0.2:13800,10.10.0.3:13800,10.10.0.4:13800,10.10.0.5:13800","repl-factor":2}' \
           http://127.0.0.1:13800/kvs/view-change

           {
               "message" : "View change successful",
               "shards" : [
                   {
                       "shard-id" : "1",
                       "key-count": 4,
                       "replicas" : ["10.10.0.2:13800", "10.10.0.3:13800"]
                   }, {
                       "shard-id" : "2",
                       "key-count": 5,
                       "replicas" : ["10.10.0.4:13800", "10.10.0.5:13800"]
                   }
               ]
           }
           200
    ```


### Key-Value Operations

- For all key-value operations:

    - If a key-value operation, more specifically the read operation, carries **causal-context** that violates casual-consistency, the system will return status code 400 and the following JSON:
    
        - {"error":"Unable to satisfy request","message":"Error in GET"}

- For the below operation descriptions, it is assumed that shard1 (containing node1 and node2) does not store **sampleKey**, and that shard2 (containing node3 and node4) does store **sampleKey**.

#### Insert new key

- To insert a key named sampleKey, send a PUT request to **/kvs/keys/sampleKey** and include the causal context object as JSON.

    - If no value is provided for the new key, the key-value store should respond with status code 400. This example sends the request to node3.

    ```bash
    $ curl --request   PUT                                        \
           --header    "Content-Type: application/json"           \
           --write-out "%{http_code}\n"                           \
           --data      '{"causal-context":causal-context-object}' \
           http://127.0.0.1:13802/kvs/keys/sampleKey

           {
               "message"       : "Error in PUT",
               "error"         : "Value is missing",
               "causal-context": new-causal-context-object,
           }
           400
    ```

    - If the key has length greater than 50, the key-value store should respond with status code 400.  
    
    ```bash
    $ curl --request   PUT                                                              \
           --header    "Content-Type: application/json"                                 \
           --write-out "%{http_code}\n"                                                 \
           --data      '{"value":"sampleValue","causal-context":causal-context-object}' \
           http://127.0.0.1:13800/kvs/keys/loooooooooooooooooooooooooooooooooooooooooooooooong

           {
               "message"       : "Error in PUT",
               "error"         : "Key is too long",
               "address"       : "10.10.0.4:13800",
               "causal-context": new-causal-context-object,
           }
           400
    ```

    - On success, the key-value store should respond with status code 201. This example sends the request to node1.

    ```bash
    $ curl --request   PUT                                                              \
           --header    "Content-Type: application/json"                                 \
           --write-out "%{http_code}\n"                                                 \
           --data      '{"value":"sampleValue","causal-context":causal-context-object}' \
           http://127.0.0.1:13800/kvs/keys/sampleKey

           {
               "message"       : "Added successfully",
               "replaced"      : false,
               "address"       : "10.10.0.4:13800",
               "causal-context": new-causal-context-object,
           }
           201
    ```

#### Update existing key

- To update an existing key named sampleKey, send a PUT request to /kvs/keys/sampleKey and include the causal context object as JSON.
  
    - If no updated value is provided for the key, the key-value store should respond with status code 400. This example sends the request to node1.

    ```bash
    $ curl --request   PUT                                        \
           --header    "Content-Type: application/json"           \
           --write-out "%{http_code}\n"                           \
           --data      '{"causal-context":causal-context-object}' \
           http://127.0.0.1:13800/kvs/keys/sampleKey

           {
               "message"       : "Error in PUT",
               "error"         : "Value is missing",
               "address"       : "10.10.0.4:13800",
               "causal-context": new-causal-context-object,
           }
           400
    ```
    
    - The key-value store should respond with status code 200. This example sends the request to node3.

    ```bash
    $ curl --request   PUT                                                              \
           --header    "Content-Type: application/json"                                 \
           --write-out "%{http_code}\n"                                                 \
           --data      '{"value":"sampleValue","causal-context":causal-context-object}' \
           http://127.0.0.1:13802/kvs/keys/sampleKey

           {
               "message"       : "Updated successfully",
               "replaced"      : true,
               "causal-context": new-causal-context-object,
           }
           200
    ```

#### Read an existing key

- To get an existing key named sampleKey, send a GET request to /kvs/keys/sampleKey and include the causal context object as JSON.
  
    - If the key, sampleKey, does not exist, the key-value store should respond with status code 404. The example sends the request to node1.

    ```bash
    $ curl --request   GET                                        \
           --header    "Content-Type: application/json"           \
           --write-out "%{http_code}\n"                           \
           --data      '{"causal-context":causal-context-object}' \
           http://127.0.0.1:13800/kvs/keys/sampleKey

           {
               "message"       : "Error in GET",
               "error"         : "Key does not exist",
               "doesExist"     : false,
               "address"       : "10.10.0.4:13800",
               "causal-context": new-causal-context-object,
           }
           404
    ```

    - On success, assuming the current value of sampleKey is sampleValue, the key-value store should respond with status code 200. This example sends the request to node1.

    ```bash
    $ curl --request   GET                                        \
           --header    "Content-Type: application/json"           \
           --write-out "%{http_code}\n"                           \
           --data      '{"causal-context":causal-context-object}' \
           http://127.0.0.1:13800/kvs/keys/sampleKey

           {
               "message"       : "Retrieved successfully",
               "doesExist"     : true,
               "value"         : "sampleValue",
               "address"       : "10.10.0.4:13800",
               "causal-context": new-causal-context-object,
           }
           200
    ```


