# Useful and Common Elasticsearch DevTools Commands

This page will be continuously updated to provide examples of useful commands to execute within DevTools. 

## Node and Cluster Commands

- Track all Nodes for Elevated or Extreme Usage
```
GET _cat/nodes?v&s=master,name&h=name,id,master,node.role,heap.percent,disk.used_percent,cpu
```

- Provide all Search Tasks for provided indices 
```
GET /_tasks?pretty&human&detailed=true&actions=indices:data/read/search`
```

- Obtain Node Tier Configuration
```
GET _cat/nodes?v&h=name,node.role
```

- List a cluster's node and assigned role
```
GET _cat/nodes?v&h=name,ip,node.role
```

- List a cluster's node and assigned role DETAILED
```
GET _cat/nodes?filter_path=nodes.*.roles,nodes.*.name
```

- View shard list
```
GET _cat/shards?v
```

- Determine node allocation
```GET _cat/allocation?v&s=node
```

- View full details of node and shard allocation state. Will also generate disk information for the nodes
```
GET _cat/shards?v=true&h=index,shard,prirep,state,node,unassigned.reason&s=state
```

- View overall cluster health status
```
GET _cluster/health
```

- View detailed explanations of node allocation; this will also help understand any issues or warnings that are reported from Stack Monitoring
```
GET _cluster/allocation/explain
```

- View any pending tasks that are staged for the nodes
```
GET _cluster/pending_tasks
```

## Index and Document Commands

- Manually create an Index
```
PUT /[index_name]
{
    "settings": {
        "number_of_shards": 1,
        "number_of_replicas": 0
    },
    {
        "mappings": {
            "properties": {
                "[field_name]": {"type": "text},
                "[field_name]": {"type": "keyword"},
                "[field_name]": {"type": "integer"}
            }
        }
    }
}
```

- Manually add a document to an index
```
POST /[index_name]/_doc
{
    "[field_name]": "[value]",
    "[field_name]": "[value]",
    "[field_name]": [integer]
}
```

- Manually delete a document
```
DELETE /[index_name]/_doc/[document_id]
```

- Search for match instances within an index. The _search is an API function to generate one or more queries that are sent to Elasticsearch. Search requests do not time out, so the timeout function will specify how long to query a shard and will only return results that have been collected at the end specified time period. 
```
GET /[index_name]/_search
```
```
GET /[index_name]/_search
{
    "timeout": "5s",  
    "query": {
        "match": {
            "user.name": "admin"
        }
    }
}
```

## Elasticsearch Cluster Upgrade 

- Disable shard allocation

```
PUT _cluster/settings
{
    "persistent": {
        "cluster.routing.allocation.enable": "primaries"
    }
}
```

NOTE that on a rolling upgrade of the nodes, shard allocation being disabled is performed automatically by the update manager in the background, then reenabled afterwards. The entire update process is designed to be "self-healing" and very fault tolerant. 

- Flush Indices

```
POST _flush
```

Any changes to the index currently in memory is automatically written to disk, or committed, and the transaction log is closed out. Performing this command before an update or a shard restart will minimize recovery and startup time for an index. 

- Halt any Machine Learning Jobs and Tasks

```
POST _ml/set_upgrade_mode?enabled=true
```

Putting the Machine Learning Jobs in Upgrade Mode was introduced in version 7.9.2. When in Upgrade Mode, the datafeeds to the jobs are paused, and any jobs stop processing new data. No new changes can be made and no new jobs can be created. Logically, the jobs remain open, and when the mode is turned off again after an upgrade is complete, the jobs will resume at the most recent automatic snapshot. This is done to help prevent the CPU and over overhead that comes from migrating and restarting. 