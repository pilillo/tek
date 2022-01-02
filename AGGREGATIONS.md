# Aggregations

[Aggregations](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-aggregations.html) are useful to summarize data into metrics. 

Aggregations can be applied to the documents resulting from a query:

```
GET /<index_name>/_search
{
    "query": {
        "match": {
            "field1": "search_value"
        }
    },
    "size": 0,
    "aggs": {
        "agg_name": {
            "agg_type": {
                "field": "field1"
            }
        },
        "agg2_name": {
            "agg2_type": {
                "field": "field1"
            }
        }
    }
}
```

By default, aggregation searches contains both document hits and aggregation results.
To only return aggregation results, set `size` to 0.

ES distinguishes aggregations in:

* `metric aggregations` - that compute metrics on basic fields, such as a sum or average;
* `bucket aggregations` - that group documents by specific fields into buckets or bins, and calculate bucket-based metrics;
* `pipeline aggregations` - that computes aggregations on the result of other aggregations;


## Metric aggregations

For instance, a total of field1:

```
GET /<index_name>/_search
{
    "query":{
        ...
    }
    "size": 0,
    "aggs": {
        "mytotal": {
            "sum": {
                "field": "field1"
            }
        }
    }
}
```

## Bucket aggregations

For instance, term aggregations use use term queries to group documents based by a specific term field (e.g., `field1`):

```
GET /<index_name>/_search
{
    "aggs":{
        "mybuckets":{
            "terms": {
                "field": "field1"
            }
        }
    }
}
```

With bucket aggregations it is possible to nest aggregations and create per-bucket stats:

```
GET /<index_name>/_search
{
    "aggs":{
        "mybuckets":{
            "terms": {
                "field": "field1"
            },
            "aggs": {
                "mystats": {
                    "stats": {
                        "field": "field2"
                    }
                }
            }
        }
    }
}
```

You can also explicitly bin documents into buckets:

```
GET /<index_name>/_search
{
    "aggs":{
        "mybuckets": {
            "range":{
                "field": "field1",
                "ranges":[
                    {
                        to: 10
                    }
                    {
                        "from": 10,
                        "to":20
                    }
                    {
                        "from":20
                    }
                ]
            }
        }
    }
}
```

or by date:

```
GET /<index_name>/_search
{
    "aggs":{
        "mybuckets": {
            "date_range":{
                "field": "field1",
                "ranges":[
                    {
                        to:  "2021-12-01"
                    }
                    {
                        "from": "2021-12-01",
                        "to":  "2022-01-01"
                    }
                    {
                        "from": "2022-01-01"
                        "to": "2022-01-01||+1y"
                    }
                ]
            }
        }
    }
}
```