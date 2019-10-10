---
layout: post
title: "Elasticsearch: Shard Allocation Failed"
date: 2019-10-10 12:55
---

Elasticsearch is great, can handle (data) node failures really well depending 
on your index settings, replication etc... mostly automagically!

However sometimes when you have a network blip, or hardware issues you can run 
into problems and it sort of recovers.

Here's some helpful-ish commands you can run:

```
curl -s -XGET 'localhost:9200/_cluster/health?pretty'
```
Tells you the state of your cluster, it should be green and you should have no 
unassigned shards.

---

```
curl -s -XGET 'localhost:9200/_cluster/allocation/explain?pretty'
```
Tells you why a shard allocation has failed. You need to sort the underlying 
issues that this highlights.

---

```
curl -H 'Content-Type: application/json' -XPUT localhost:9200/_cluster/settings -d '{ "transient" :{ "cluster.routing.allocation.node_concurrent_recoveries" : 16 } }'
```
This makes recoveries go faster by letting the cluster do more recoveries 
simultaneously.

---

```
curl -H 'Content-Type: application/json' -XPOST 'localhost:9200/_cluster/reroute?retry_failed&pretty'
```
If you’ve sorted all your issues but there’s still unassigned shards for some 
reason it could be that you’ve hit the max_retry limit (default: 5) and you 
need to manually kick it into action.
