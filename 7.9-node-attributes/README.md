# Hot Warm Cold with node attributes

Prior to 7.10, hot, warm, cold architectures were implemented with node attributes and [shard allocation filtering](https://www.elastic.co/guide/en/elasticsearch/reference/current/shard-allocation-filtering.html)

7.10+ you can still use node attributes, but data tiers is the new prefered implementation. 

This is an example of how to implement hot, warm, cold with node attributes.


## Pre-reqs

* Docker and docker-compose installed 

## Example

```
docker-compose up
```

Navigate to http://localhost:5601/app/dev_tools#/console

```
GET /_cat/nodes?v
GET _nodes?filter_path=nodes.*.name,**.attributes.season
```
results in
```
# GET /_cat/nodes?v
ip         heap.percent ram.percent cpu load_1m load_5m load_15m node.role master name
172.23.0.4           41          54   0    0.71    0.62     0.68 dilmrt    -      node2
172.23.0.3           25          54   0    0.71    0.62     0.68 dilmrt    -      node1
172.23.0.2           66          54   0    0.71    0.62     0.68 dilmrt    *      node3

# GET _nodes?filter_path=nodes.*.name,**.attributes.season
{
  "nodes" : {
    "DrH4nY6HSdiC1a247JpPSw" : {
      "name" : "node2",
      "attributes" : {
        "season" : "spring"
      }
    },
    "rWzGEFH5TyydAxnvPsQIzw" : {
      "name" : "node1",
      "attributes" : {
        "season" : "summer"
      }
    },
    "NAGUor5EQ02whnKQATiFhQ" : {
      "name" : "node3",
      "attributes" : {
        "season" : "winter"
      }
    }
  }
}
```
Here we have 3 nodes. Each node has an node attribute `season` where the values are`summer`, `spring`, and `winter` (defined in the elasticsearch.yml file used to start this instance).  These attribute name and values are arbitrary and don't have any meaning beyond whatever I choose to tag the node. Most people will likely choose the values are `hot`, `warm`, and `cold` but I chose to not to use them to emphasize that the name and value are arbitray and do not actually carry any specific meaning.

Not having a standardized set of reserved names for these attributes is one motivating factors for the introduction of data tiers in 7.10. As a developer of the system, I have no idea what you may pick to represent `hot`, `warm`, `cold` limiting my ability to design features against these concepts. 


Let's write a couple documents (which creates the indexes) and see which node hosts the shards.

```
POST test_one/_doc
{ "@timestamp" : 0 }
POST test_two/_doc
{ "@timestamp" : 0 }
GET _cat/shards?h=index,prirep,state,docs,node&v
```

Will result in something like
```
index                          prirep state   docs node
test_two                       r      STARTED    1 node1
test_two                       p      STARTED    1 node3
test_one                       p      STARTED    1 node1
test_one                       r      STARTED    1 node2

```

The primary shard for `test_one` was allocated to `node1`, and the replica was allocated to `node2`

The primary shard for `test_two` was allocated `node3` and the replica was allocated `node1`

Any of the nodes could have been assigned the primary or replicas since we don't have not introduced any allocation rules (except primary replicas are never co-located on same host). Basically the allocation (assignment of a index/shards to a host) can happen on any of these nodes.

Let's delete those index'es, and implement a hot/warm/cold architecture. 

```
DELETE test_*
```

We will use a data stream in favor of standard indexes since a data stream represents a time series and is backed by a set of indexes instead of a single index. So within the same data stream, I can have indices on the hot node (summer), indices on the warm node (spring), and indices on the cold node (winter).

We will want the very first index to land on the hot (summer) node(s), and then after some criteria (index size or number of docs), we want that index to move to the (warm) node(s), and then again after some more critera to the code.  We will use ILM to move the index's around and an index templates to ensure the first index lands on the hot (summer) node.

Lets start with the ILM policy:

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
        "actions": {
          "allocate": {
            "require": {
              "season": "spring"
            }
          }
        }
      },
      "cold": {
        "min_age": "1m",
         "actions": {
          "allocate": {
            "require": {
              "season": "winter"
            }
          }
        }
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
Here we will rollover to a new index after 5 documents, then immediatly move the index to the warm (spring) node(s), then after 1 minute on the warm node the index will get moved to cold (winter) node(s). Also, ILM has a default poll interval of 10 minutes ... so for demostration purposes we will crank that down to polling every 1 second (don't do that in prod :) )

Let's create an index template that will get applied for these test_* indices, will create the data stream, and assign the first index allocation to the hot (summer) nodes.

```
PUT _index_template/test_indexes
{
  "index_patterns": ["test_*"],
  "data_stream" : {},
  "template": {
    "settings": {
        "index.routing.allocation.require.season": "summer",
        "index.lifecycle.name" : "test_policy"
    }
  }
}

```

Now, let's re-create those documents 

```
POST test_one/_doc
{ "@timestamp" : 0 }
POST test_two/_doc
{ "@timestamp" : 0 }
GET _cat/shards?h=index,prirep,state,docs,node&v
``` 

Now we see
```
index                          prirep state      docs node
.ds-test_one-000001            p      STARTED       1 node1
.ds-test_one-000001            r      UNASSIGNED      
.ds-test_two-000001            p      STARTED       1 node1
.ds-test_two-000001            r      UNASSIGNED      
```

These .ds-[name]-[number] are the backing indices for the data stream. Note that both primaries have been correctly assigned to node1, which is the hot (summer) node. This is not just a coincidence, this is becuase of the `"index.routing.allocation.require.season": "summer"` from the index template. 

Here we notice that the replica is not assigned becuase we only 1 hot summer node, and the primary and replica can not be co-located on the same node. We could either add more nodes with the `summer` node attribute or we can just use zero replicas. For this demo, let's just use zero replicas for simplicity. 

Let's delete the data streams (which also delete the backing indices):

```
DELETE _data_stream/test_*
GET _cat/shards?h=index,prirep,state,docs,node&v
```

and update the template to use zero replicas

```
PUT _index_template/test_indexes
{
  "index_patterns": ["test_*"],
  "data_stream" : {},
  "template": {
    "settings": {
        "index.routing.allocation.require.season": "summer",
        "index.lifecycle.name" : "test_policy",
        "index.number_of_replicas" : 0
    }
  }
}
```

Now, let's re-create those documents (again)

```
POST test_one/_doc
{ "@timestamp" : 0 }
POST test_two/_doc
{ "@timestamp" : 0 }
GET _cat/shards?h=index,prirep,state,docs,node&v
``` 

results in both primary shards on `node1` (hot/summer)
```
index                          prirep state   docs node
.ds-test_two-000001            p      STARTED    1 node1
.ds-test_one-000001            p      STARTED    1 node1

```

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
index                          prirep state   docs node
.ds-test_two-000001            p      STARTED    1 node1
.ds-test_one-000001            p      STARTED    6 node2
.ds-test_one-000002            p      STARTED    0 node1
```
ILM has created a new backing index `.ds-test_one-000002` (on the hot node), and moved the original index `.ds-test_one-000001` over to the warm node. 

Let's wait another minute...

```
GET _cat/shards?h=index,prirep,state,docs,node&v
```
results in
```
index                          prirep state   docs node
.ds-test_one-000001            p      STARTED    6 node3
.ds-test_two-000001            p      STARTED    1 node1
.ds-test_one-000002            p      STARTED    0 node1
```

Now the original index is on `node3` , the cold node becuase ILM waited 1 minute and then moved it there based on the policy we defined. 

`test_two` data stream's are still on the hot node since we haven't indexed any data there and the criteria is 5 documents before it rolls over and moves to the cold phase. 

Both the ILM policy and index template are re-usable across as many data streams as you like. 