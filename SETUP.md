# Setup

## Quick setup using the provided docker compose

```docker-compose up -d```

## Setup using a downloaded release

After configuring the `elasticsearch.yaml` run `<installation-folder>/bin/elasticsearch`.

## Architecture

An ES cluster is a collection of ES nodes.
An ES node is an instance that can store data, i.e. documents, json-formatted data stored in an index.
An index is thus a collection of logically-related documents.
Data is distributed across a cluster by subdividing the index in multiple shards. Each shard is an independent apache lucene index, so an ES index is a collection of apache lucene indices. As such, a shard may store a max of about 2 Billion documents. The reason for sharding is to achieve horizontal scalability, as well as improved performance given that queries can be run on multiple nodes simultaneously. Additionally, shard replication provides ES with fault tolerance, so that if a node fails another one within its *replication group* can take over to serve its queries. For this replicas are never stored on the same node. Moreover, snapshots are taken of the index at regular intervals, so that if a node fails, it can be restored to its previous state.
An ES node will always be part of a cluster, with the first starting instance creating a new cluster and following ones joining it. In fact, the cluster has a master/slave architecture, with a master node managing operations on the index and data nodes serving the data. The master is elected via a voting process out of a pool of elegible master nodes. In large clusters it is important to separate the master and data nodes, as the master node is responsible for managing the cluster and the data nodes for serving queries.

## Cluster Information

* `GET /_cluster/health` to retrieve info about the ES cluster;
* `GET /_cat/nodes?v` to get node info in a tabular fashion (with v for verbose);
* `GET /_nodes` to get node info in a json format;
* `GET /_cat/indices?v` to get index info in a tabular fashion;
* `GET /_cat/shards?v` to get shards info in a tabular fashion;

## Node settings

The `elasticsearch.yaml` file contains the settings for the node, among which:
* `cluster.name` is the name of the cluster the node belongs to;
* `node.name` is the name of the node;
* `path.data` specifies where the data is stored;
* `path.logs` specifies where the logs are stored;

## Node roles
Nodes may be associated with specific roles:
* `node.master` is a flag indicating whether the node can be elected to a master node;
* `node.data` is a flag indicating whether the node can be used to serve data;
* `node.ingest` is a flag indicating whether the node can be used to ingest data (i.e., to create and update indices);
* `node.ml` is a flag indicating whether the node can be used as machine learning node (i.e., to run machine learning jobs); this is useful to separate a machine learning node from a data node.
* `xpack.ml.enabled` is a flag indicating whether the node can serve machine learning requests; this is useful to separate a machine learning node from a data node.
* a `coordination role` a node with all roles set to false becomes automatically a coordination node (i.e., that does distribute queries to data nodes and aggregates results).
* `node.voting_only` a node that can vote for a master node but it is not allowed to become a master node itself; this is only used in large clusters;