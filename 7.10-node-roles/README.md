# Hot Warm Cold with node attributes

Prior to 7.10, hot, warm, cold architectures were implemented with node attributes and [shard allocation filtering](https://www.elastic.co/guide/en/elasticsearch/reference/current/shard-allocation-filtering.html)

7.10+ you can still use node attributes, but data tiers is the new prefered implementation. 

This is an example of how to implement hot, warm, cold with node attributes. This purpose of this example to set a foundation
for data tiers and why you would want to use them. It builds on the 7.9-node-attributes example to help contrast the change introduced in 7.10.


## Pre-reqs

* Docker and docker-compose installed 

## Example

```
docker-compose up
```
```
GET /_cat/nodes?v
GET _nodes?filter_path=nodes.*.name,nodes.*.roles
```

results in
```
ip         heap.percent ram.percent cpu load_1m load_5m load_15m node.role master name
172.25.0.3           64          58   0    0.01    0.29     0.54 msw       -      node2
172.25.0.2           65          58   0    0.01    0.29     0.54 cms       -      node3
172.25.0.4           52          58   0    0.01    0.29     0.54 hms       *      node1

{
  "nodes": {
    "tdj688tRQ3q60rjJCcAiRw": {
      "name": "node2",
      "roles": [
        "data_content",
        "data_warm",
        "master"
      ]
    },
    "iCy6gd8XREaAVki3fqh7VA": {
      "name": "node1",
      "roles": [
        "data_content",
        "data_hot",
        "master"
      ]
    },
    "evZR4elWQDKj2n1tSpmDSA": {
      "name": "node3",
      "roles": [
        "data_cold",
        "data_content",
        "master"
      ]
    }
  
```

Note - there was no need to define any custom attributes. As of 7.10 data_hot, data_warm, data_cold are node roles which are NOT arbitrarily named values. They are a list of the roles the node performs, where hot, warm, and cold are now official roles of a node. The data_content role to instruct where to place any non-data streams. In this case we will allow normal index to live on any node in the cluster. 

Similar to the 7.9 example, we will create a data stream and ILM policy. However now that have the formal notion of hot,warm,cold we can use that to simplify the setup. 

Let's setup the ILM policy first.

```
PUT _ilm/policy/test_policy
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_docs": 5
          }
        }
      },
      "warm": {
        "actions" : {}
      },
      "cold": {
        "min_age": "1m",
        "actions" : {}
      }
    }
  }
}


PUT _cluster/settings
{
  "transient": {
    "indices.lifecycle.poll_interval": "1s"
  }
}
```

What happened to the warm, and cold phases ? When using node roles and data streams the migration is automatic ! 

Let's put the index template: 

```
PUT _index_template/test_indexes
{
  "index_patterns": ["test_*"],
  "data_stream" : {},
  "template": {
    "settings": {
        "index.lifecycle.name" : "test_policy",
        "index.number_of_replicas" : 0
    }
  }
}
```

We no longer need custom shard allocation filtering, when using node roles and data streams the allocation to the data_hot is automatic !

Now, let's re-create those documents as we did with 7.9

```
POST test_one/_doc
{ "@timestamp" : 0 }
POST test_two/_doc
{ "@timestamp" : 0 }
GET _cat/shards?h=index,prirep,state,docs,node&v
``` 

Results in 
```
index               prirep state   docs node
.ds-test_one-000001 p      STARTED    0 node1
.ds-test_two-000001 p      STARTED    0 node1
```

Both data streams were allocated on the data_hot nodes. 

Lets create 5 more documents for `test_one` data stream
```
POST test_one/_doc
{ "@timestamp" : 0 }
POST test_one/_doc
{ "@timestamp" : 0 }
POST test_one/_doc
{ "@timestamp" : 0 }
POST test_one/_doc
{ "@timestamp" : 0 }
POST test_one/_doc
{ "@timestamp" : 0 }
```

give it a couple seconds for the ILM poll to find that number of documents have been exceeded and rollover the index

```
GET _cat/shards?h=index,prirep,state,docs,node&v
```

results in
```
index                prirep state   docs node
ilm-history-3-000001 p      STARTED      node3
ilm-history-3-000001 r      STARTED      node2
.ds-test_one-000001  p      STARTED    6 node2
.ds-test_one-000002  p      STARTED    0 node1
.ds-test_two-000001  p      STARTED    1 node1
```

Where ilm-history is normal index, it was allocated to node2 and node3 since they are part of the content tier. The original index was rolled over and the new index was allocated to data_hot node and the elder index was moved to the warm node. 

Let's wait another minute...

```
GET _cat/shards?h=index,prirep,state,docs,node&v
```
results in
```
index                prirep state   docs node
ilm-history-3-000001 p      STARTED      node3
ilm-history-3-000001 r      STARTED      node2
.ds-test_one-000001  p      STARTED    6 node3
.ds-test_one-000002  p      STARTED    0 node1
.ds-test_two-000001  p      STARTED    1 node1
```

After a minute the elder backing index was moved to node3, the data_cold node. 
