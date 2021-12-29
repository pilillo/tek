# Search

The search API is available at `/<index_name>/_search` and query parameters can be defined either directly as a query string, such as `/<index_name>/_search?q=field1:value1 AND field2:value2`. This is clearly less expressive and less readable than providing a Json payload.
For this reason, ES provides a search domain-specific language (DSL) to define queries, such as:

```
GET /<index_name>/_search
{
  "query": {
    "ids": {
      "values": ["id1","id2"]
    }
  }
}
```

to retrieve all documents with the specified ids. 

## Query types

The search DSL distingueshes between leaf queries and compound queries, searching respectively for values in specific fields or in combinations of fields using for instance boolean operators (see doc [here](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl.html)). In addition, search queries return a relevance score that shows how well does the returned document match the provided query. Score calculation depends on whether the query clause is run in a *query* or a *filter* context, namely:

* in *filter* context no relevance score is calculated but a boolean evaluation is done to filter only matching results
* in *query* context a relevance score is calculated and returned in the search results, so the document overall relevance is affected;

Orthogonally to this, ES distingueshes between *term queries* (exact matches looking on the inverted index, without being analyzed beforehand) and full-text queries, whereas the *match query* is analyzed according to the same process used to index the document and then passed over to the index for the actual full-text search. 

For instance:

```
GET /<index_name>/_search
{
  "query": {
    "term": {
      "field1": "value1"
    }
  }
}
```

will look for documents where field1 is equal to the provided value. This also works on prefixes and even wildcard and regular expressions:

```
GET /<index_name>/_search
{
  "query": {
    "prefix": {
      "field1.keyword": "val"
    }
  }
}
```

```
GET /<index_name>/_search
{
  "query": {
    "wildcard": {
      "field1.keyword": "val*1"
    }
  }
}
```

You can alternatively use a `fuzziness` parameter to specify how many edits (as in Levenshtein edit distance) are allowed to the provided value to match. Similarly, you can optionally enable `transpositions`.

For multiple terms:

```
GET /<index_name>/_search
{
  "query": {
    "terms": {
      "field1.keyword": [
          "value1",
          "value2"
      ]
    }
  }
}
```

Term queries also include queries on numerical and date fields, for instance:

```
GET /<index_name>/_search
{
  "query": {
    "range": {
      "field1": {
          "gte" : 1,
          "lte" : 10
      },
      "field2": {
          "gte" : "2021/10/20",
          "lte" : "2021/12/31"
          "format" : "yyyy/MM/dd"
      }
    }
  }
}
```

On the other hand, the query:

```
GET /<index_name>/_search
{
  "query": {
    "match": {
      "field1": "value1 value2"
    }
  }
}
```

does match all documents matching any of the values specified in the search field. The behavior can be overridden by specifying the `operator` parameter, which can be set to `or` or `and`. In case the order of terms matters, the match query can be replaced by a `match_phrase` one, instead. This parameter is usable for proximity search, since a `slop` parameter can be specified to define the maximum distance between the provided terms.

Also, a document id can be provided to see whether the search query would match and retrieve the document:

```
GET /<index_name>/<document_id>/_search
{
  "query": {
    "match": {
      "field1": "value1"
    }
  }
}
```

Mind also that, an explanation parameter can be set (see doc [here](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-explain.html)) in the query to get a detailed explanation of the query:

```
GET /<index_name>/_search
{
  "explain": true,
  "query": {
    "match": {
      "field1": "value1"
    }
  }
}
```

Similarly, a profile parameter can be set (see doc [here](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-profile.html)) to retrieve timing information of individual components used for serving the query request:

```
GET /<index_name>/_search
{
  "profile": true,
  "query": {
    "match": {
      "field1": "value1"
    }
  }
}
```

## Boolean operators

```
GET /<index_name>/_search
{
  "query": {
    "bool" : {
      "must" : [
        { "match" : { "field1" : "value1" } },
        { "match" : { "field2" : "value2" } },
      ]
      "filter" [
          { "range" : { "field3" : { "lte": 10} } }
      ]
    }
  }
}
```

with the individual queries in the must that must match but also contribute to the overall relevance score.
On the contrary, the example filter clause is used to filter out documents not matching the query range.
Similar operators are `must_not` (behaving like a filter) and `should` (affecting relevance with a boost and not filtering out non-matching documents).

## Customizing returned results
You can customize the returned results by specifying the `fields` parameter. For instance, to return only the `field1` field:

```
GET /<index_name>/_search
{
  "fields": ["field1"]
}
```
 
or by directly accessing the `_source` document fields (by listing which ones to include and exclude from the result):

```
GET /<index_name>/_search
{
  "_source": {
    "includes": ["field1"],
    "excludes": []
  }
}
```

You can similarly limit the number of returned documents (e.g., 10):
```
GET /<index_name>/_search
{
  "size": 10,
  "query": {...}
}
```

You can also use pagination (also called cursor) to limit the number of returned documents for a result so as to return them in smaller chunks. A `from` parameter can be used to specify the offset of the first returned document, see official doc [here](https://www.elastic.co/guide/en/elasticsearch/reference/current/paginate-search-results.html).

Finally, results can be sorted by specific fields:

```
GET /<index_name>/_search
{
  "sort": [
    {
      "field1": {
        "order": "asc"
      }
    },
    {
      "field2": {
        "order": "desc"
      }
    }
  ]
}
```