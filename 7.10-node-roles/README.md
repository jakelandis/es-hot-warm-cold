# Hot Warm Cold with node attributes

Prior to 7.10, hot, warm, cold architectures were implemented with node attributes and [shard allocation filtering](https://www.elastic.co/guide/en/elasticsearch/reference/current/shard-allocation-filtering.html)

7.10+ you can still use node attributes, but data tiers is the new prefered implementation. 

This is an example of how to implement hot, warm, cold with node attributes. This purpose of this example to set a foundation
for data tiers and why you would want to use them


## Pre-reqs

* Docker and docker-compose installed 

## Example

```
docker-compose up
```

Navigate to http://localhost:5601/app/dev_tools#/console

TODO