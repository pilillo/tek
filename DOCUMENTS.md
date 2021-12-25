# Documents

## Creating and deleting index

An index can be created with default sharding and replication settings, with:

`PUT /<index_name>`

or can be otherwise set with specific settings as follows:  
```
PUT /<index_name>
{
  "settings": {
    "index": {
      "number_of_shards": 1,
      "number_of_replicas": 0
    }
  }
}
```

And finally `DELETE /<index_name>` to delete the index.

## Creating and deleting a document

* `POST /<index_name>/_doc` to create a document with an automatically generated id;
* `POST /<index_name>/_doc/<document_id>` to create a document with a specific id;
* `POST /<index_name>/_doc/<document_id>` to retrieve a document by its id;
* `POST /<index_name>/_doc/<document_id>` to delete a document;

## Updating a document

To update a document:
```
POST /<index_name>/_doc/<document_id>
{
  "doc": {
    "field1": "value1",
    "field2": "value2",
    "field3": "value3"
  }
}
```

for instance with `field1` and `field2` specified with their new field values and `field3` being added to the document.
As visible in the result message, the result is `updated` and the `_version` field was incremented. Under the hood, the document is updated by replacing the old document with the new one.

You can also update using a script, such as:
```
POST /<index_name>/_doc/<document_id>
{
  "script": {
    "source": "ctx._source.field1 = params.field1",
    "params": {
      "field1": "value1"
    }
  }
}
```

You can also upsert (update or insert) a document with:

```
POST /<index_name>/_doc/<document_id>
{
  "script": {
    "source": "ctx._source.field1 = params.field1",
    "params": {
      "field1": "value1"
    }
  }
  "upsert": {
    "field1": "value1",
    "field2": "value2",
    "field3": "value3"
  }
}
```

this way the update script is added if the document exists, otherwise the document defined in the upsert section is inserted.
Consequently, the result message will have its `result` set to either `created` or `updated` and the `_version` field will be incremented.

Mind also that upon any document modification a primary term (i.e., `_primary_term`) and a `sequence number` (i.e., `_seq_no`) are added and returned in the result message. The primary term specifies the number of times the primary shard has changed. The sequence number is a monotonically increasing value that is incremented by the primary shard (lead) for each operation on the document. The purpose is to enable ES to order write operations and thus recover from failures occurring on primary shards. To speed up recovery in large clusters, ES also uses global and local `checkpoints`, respectively mantained by the replication group and by the replica shard. The global checkpoint is the sequence number up to which all active shards are at least aligned to. Any operation having a sequence number lower than the global checkpoint were already performed on old shards and can be safely ignored. This way, any failing primary shard can be recovered by checking its last sequence number to the global one. Similarly, for local checkpoints, the replica shard will only perform operations that are greater than its local checkpoint. This mechanism is a form of so called optimistic concurrency control.

The primary term and the sequence number are thus also the means of managing concurrency control in ES and can be provided to any update operation to make sure that the operation is not performed on an old version of the document. For instance:

```
POST /<index_name>/_doc/<document_id>?if_primary_term=<primary_term>&if_seq_no=<seq_no>
{
  "doc": {
    "field1": "value1",
    "field2": "value2",
    "field3": "value3"
  }
}
```

which will only be updated if the provided primary term and sequence number match that of the latest version on ES.
This prevents those situations in which a document has been modified by another process during its retrieval and update.
For those cases the update operation will fail and a new retrieval and update will have to take place at application level.

## Updating and deleting by query

You can update by query by defining a query and an update script, for instance:
```
POST /<index_name>/_update_by_query
{
  "query": {
    "match": {
      "field1": "search_value"
    }
  },
  "script": {
    "source": "ctx._source.field1 = params.field1",
    "params": {
      "field1": "value1"
    }
  }
}
```

which will update all documents where field1 is equal to the provided search value.

A similar operation can be performed to delete by query, for instance:
```
POST /<index_name>/_delete_by_query
{
  "query": {
    "match": {
      "field1": "search_value"
    }
  }
}
```

## Bulk operations
Batch processing of multiple `index`, `create`, `update` and `delete` operations can be performed by using the `_bulk` endpoint. The idea is to provide a list of operations in a unique API call. The operations are specified in the `Content-Type: application/x-ndjson` (so do not forget to set your headers in your requests) format and executed in provided order. A failed action will not affect neither the others nor the result of the overall bulk operation, so it is necessary to inspect the returned message to understand the outcome of each operation. For instance:

```
POST /_bulk
{ "index" : { "_index" : "test", "_id" : "1" } }
{ "field1" : "value1" }
{ "delete" : { "_index" : "test", "_id" : "2" } }
{ "create" : { "_index" : "test", "_id" : "3" } }
{ "field1" : "value3" }
{ "update" : {"_id" : "1", "_index" : "test"} }
{ "doc" : {"field2" : "value2"} }
```

For operations to a specific index you can also use `POST /<index_name>/_bulk` directly.
Please have a look at the official documentation [here](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-bulk.html).