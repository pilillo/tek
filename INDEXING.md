# Indexing

## Analyzers

To efficiently query text, analyzers are used to process it before storing it in searchable data structures.
This includes, in order:
* *character filters* which are applied before tokenization to add/remove/edit the text based on character filters;
* *tokenizers* which split input strings into tokens (e.g. words);
* *token filters* which can add, remove, edit tokens (e.g. to lowercase, remove stop words, etc);

```
POST /_analyze
{
  "analyzer": "standard",
  "text": "This is a test"
}
```

which will return a list of processed tokens:
```
{
  "tokens": [
      {
          ...
      }
  ]
}
```

you can also customize the individual steps with, for instance:

```
POST /_analyze
{
  "text": "This is a test",
  "char_filter": [],
  "tokenizer": "standard",
  "filter" : [
      "lowercase",
      "stop",
      "snowball"
  ]
}
```

There exist standard analyzers, i.e., built-in combinations of character filter, tokenizers and token filters. Specifically:

* `standard` splits in words and removes punctuation, converts all to lowercase and provides a `stop` token filter to remove stop words, but it is disabled by default;
* `simple`, similar to `standard`, but splits for anything else than a letter, and lowercasing is performed by the tokenizer and not the token filter to increase performance;
* `whitespace` splits into tokens by whitespace, does not lowercase letters;
* `keyword` does not split, does not lowercase, and does not remove stop words; it does emit the original text as a single token;
* `pattern` uses a regular expression to match token separators (i.e., any match is used to split the text into tokens), and does lowercase letters by default;

And additional ones, for instance related to language specific aspects. When built-in analyzers are not enough, it is possible to create custom ones, e.g.:

```
PUT /<index_name>
{
  "settings": {
    "analysis": {
      "analyzer": {
        "my_analyzer": {
          "type": "custom",
          "char_filter": ["html_strip"],
          "tokenizer": "standard",
          "filter": [
            "lowercase",
            "stop",
            "snowball"
          ]
        }
      }
    }
  }
}
```

You can also customize the analyzer components, for instance by specifying for which language the stop words should refer to:

```
PUT /<index_name>
{
  "settings": {
    "analysis": {
      "char_filter":{},
      "tokenizer":{},
      "filter": {
        "stop_english": {
          "type": "stop",
          "stopwords": "_english_"
        }
      },
      "analyzer": {
        "my_analyzer": {
          "type": "custom",
          "char_filter": ["html_strip"],
          "tokenizer": "standard",
          "filter": [
            "lowercase",
            "stop_english",
            "snowball"
          ]
        }
      }
    }
  }
}
```

Mind that analyzers can not be added to existing indices, but only to new ones. To achieve this, the index can be temporarily disabled and then enabled again by:

```
POST /<index_name>/_close
... # updating the index settings
POST /<index_name>/_open
```

## Quering terms

In the context of search, tokens are generally referred to as terms. Extracted tokens are generally organized in an inverted index, i.e., a mapping from tokens to documents, where each document is identified by a unique ID and is associated to relevance information related to the search term. 

## Mapping

Mapping defines the document schema, i.e., the fields and their data types, as well as how they are indexed. Mapping can be:
* *dynamically* generated from the data
* *explicitly* defined by the user

for instance:

```
PUT /<index>
{
  "mappings":{
    "properties":{
      "field1":{"type":"text"}
      "nested_field_1":{
        "properties":{
          "field2":{"type":"float"}
        }
      }
    }
  }
}
```

which can be retrieved with:

```
GET /<index>/_mapping
```

The mapping can be then modified to add new fields with: 

```
PUT /<index>/_mapping
{
  "properties":{
    "new_field":{ "type":"date" }
  }
}
```

The existing field mappings can however not be modified, since this may result in issues with the existing documents. Modifying a field data type may imply modifying the kind of data structure used to index it.
Therefore modifying the mapping is only possible by deleting the index and creating a new one. For this purpose ES provides a dedicated API:

```
POST /_reindex
{
  "source":{
    "index":"<index_name>"
  },
  "dest":{
    "index":"<new_index_name>"
  }
}
```

During reindexing, the source index is read and all documents are copied to the new index. A query can be specified to filter the documents to be copied, such as:

```
POST /_reindex
{
  "source":{
    "index":"<index_name>"
    "query":{
      "match":{
        "field1":"value1"
      }
    }
  },
  "dest":{
    "index":"<new_index_name>"
  }
}
```

The fields to be copied can be limited using the "_source" parameter:

```
POST /_reindex
{
  "source":{
    "index":"<index_name>"
    "_source":["field1","field2"]
  },
  "dest":{
    "index":"<new_index_name>"
  }
}
```

A script can be provided to customize the process, for instance to filter documents based on a field value or to change their data type.

```
POST /_reindex
{
  "source":{
    "index":"<index_name>"
  },
  "dest":{
    "index":"<new_index_name>"
  }
  "script":{
    "source":"""
      # remove field
      ctx._source.remove('field_to_remove')
      # convert field to string
      ctx._source.field_to_change = ctx._source.field_to_change.toString()
      # rename field
      ctx._source.new_field_name = ctx._source.remove('field_to_remove')
    """
  }
}
```

In the third example, a field is renamed. While this is possible, it may not make sense to reindex an entire index for this purpose.
A solution might be to create field aliases:

```
PUT /<index>/_mapping
{
  "properties":{
    "alias_name" :{
      "type":"alias",
      "path":"field_name"
    } 
  }
}
```

For those cases where a field may be queried in different ways, ES supports the possibility to define multi-field mappings, such as:

```
PUT /<index>
{
  "mappings":{
    "properties":{
      "field1":{
        "type":"text"
        "fields":{
          "other_name":{
            "type":"keyword"
          }
        }
      }
    }
  }
}
```

where the field "other_name" is a multi-field. Mind that by convention this is generally named as the field type (i.e., `keyword` instead of `other_name`). In this case we can search `field1` as either `field1` or `field1.keyword`, respectively using match query (full text search) and a term query (exact match). This is the most common case although not the only one.

## Data Types

The full list is available on the official documentation [here](https://www.elastic.co/guide/en/elasticsearch/reference/current/sql-data-types.html), and lists:


| Data type     |      Notes       |
|---------------|------------------|
| boolean       |                  |
| byte          |                  |
| short         |                  |
| integer       |                  |
| long          |                  |
| double        | precision 15     |
| float         | precision 7      |
| half_float    | precision 3      |
| scaled_float  | precision 15     |
| keyword       | exact matching   |
| text          |                  |
| binary        |                  |
| date          |                  |
| ip            |                  |
| object        | nested documents |
| nested        | object arrays    |

where:
* *keyword* is used for exact value matching, such as in sorting, filtering and aggregations; keyword fields are processed by the keyword analyzer which is a no-op analyzer;
* *text* is used for full text search;
* *object* is used for any json object; although they are flattened using the dot notation before being indexed in lucene; this means losing the original structure in case of object arrays, since each field in the array will be indexed as a separate field;
* *nested* is an object that keeps the relationship between the fields of object arrays; they are in fact stored as hidden documents;
* *date* expects a date in the ISO 8601 format (e.g., 2021-12-25T23:59:59Z, with Z indicating UTC, or 2021-12-25T23:59:59+01:00 to indicate the offset from UTC), with or without time (in which case it is assumed to be at midnight) or expressed in milliseconds since the epoch; unless else is specified these are assumed to be in UTC; these are finally stored as long valued milliseconds since the epoch; 

Mind also that there is no such thing as an array type, since any field may contain any number of values. 
So the following are both accepted:

```
POST /<index>/_doc
{
  "field1": "value1"
}
```

```
POST /<index>/_doc
{
  "field1": ["value1", "value2"]
}
```

with the mapping being simply:

```
{
  "<index>":{
    "mappings":{
      "properties":{
        "field1":{"type":"text"}
      }
    }
  }
}
```

What happens under the hood is that the values are concatenated (with a space in between) into a single string that is processed by the analyzer and the resulting tokens are indexed in the inverted index as usual. Non-text fields are not analyzed and are indexed directly in the data-specific structures. Also, arrays must contains values of the same type, unless a mapping already exist for the document, coercion is enabled (at either field level with `"coerce": true` or index level with `"settings":{"index.mapping.coerce":false}`) and the values can be actually coerced into the same type. Arrays may also contain nested arrays, which are however flattened during indexing.

The following parameters can be set:

* `format` parameter can be defined for non-standard date formats, such as `dd/MM/yyyy` or `yyyy-MM-dd'T'HH:mm:ss.SSSZ` or `epoch_second`.
* `coerce` parameter can be used to specify whethere values should be coerced into the same type as the field.
* `null_value` parameter can be used to specify a value (of same type) to be used when a field is missing in a document;
* `copy_to` parameter can be used to specify another field of the mapping where to copy the value of the analyzed field to;
* `doc_values` parameter can be used to specify whether the field should be stored in the document values data structure of lucene (default is `false`);
* `norm` parameter can be used to set whether normalization factors should be used in relevance scoring, as this takes additional space and overhead;
* `index` parameter can be used to disable indexing for a field to save space and reduce overhead;