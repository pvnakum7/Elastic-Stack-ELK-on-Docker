**Elasticsearch cluster**
Scaling out Elasticsearch

This project supports a single-node Elasticsearch cluster by default. Following the instructions in this page, you will be able to scale out that cluster by adding extra nodes.

ES multi replicas

image source: Elasticsearch: The Definitive Guide » Replica Shards
Prerequisites
Increase vm.max_map_count

One must increase the vm.max_map_count kernel setting on all Docker hosts running Elasticsearch in order to pass the bootstrap checks triggered by the production mode. To do this, follow the recommended instructions from the Elastic documentation: Install Elasticsearch with Docker.
Configuration
Compose mode

In a local setup, Elasticsearch can bootstrap a cluster automatically by discovering the other nodes running on the same machine (ref. Bootstrapping a cluster » Auto-bootstrapping in development mode). Unfortunately, this mechanism cannot be leveraged when multiple Elasticsearch instances listen on the same ports (9200/9300), like inside containers.

Instead, it is necessary to follow the approach described at Start a multi-node cluster with Docker Compose, and have one separate Compose service per Elasticsearch instance:

# docker-compose.yml

  elasticsearch01:
    build:
      context: elasticsearch/
      args:
        ELASTIC_VERSION: ${ELASTIC_VERSION}
    volumes:
      - ./elasticsearch/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml:ro,z
      - elasticsearch01:/usr/share/elasticsearch/data:z
    ports:
      - 9200:9200
    environment:
      ES_JAVA_OPTS: -Xms512m -Xmx512m
      ELASTIC_PASSWORD: ${ELASTIC_PASSWORD:-}
      # Set a deterministic node name.
      node.name: elasticsearch01
      # Use other cluster nodes for unicast discovery.
      discovery.seed_hosts: elasticsearch02,elasticsearch03
      # Define initial masters, assuming a cluster size of at least 3.
      cluster.initial_master_nodes: elasticsearch01,elasticsearch02,elasticsearch03
    networks:
      - elk

  elasticsearch02:
    build:
      context: elasticsearch/
      args:
        ELASTIC_VERSION: ${ELASTIC_VERSION}
    volumes:
      - ./elasticsearch/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml:ro,z
      - elasticsearch02:/usr/share/elasticsearch/data:z
    environment:
      ES_JAVA_OPTS: -Xms512m -Xmx512m
      # Set a deterministic node name.
      node.name: elasticsearch02
      # Use other cluster nodes for unicast discovery.
      discovery.seed_hosts: elasticsearch01,elasticsearch03
      # Define initial masters, assuming a cluster size of at least 3.
      cluster.initial_master_nodes: elasticsearch01,elasticsearch02,elasticsearch03
    networks:
      - elk

  elasticsearch03:
    build:
      context: elasticsearch/
      args:
        ELASTIC_VERSION: ${ELASTIC_VERSION}
    volumes:
      - ./elasticsearch/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml:ro,z
      - elasticsearch03:/usr/share/elasticsearch/data:z
    environment:
      ES_JAVA_OPTS: -Xms512m -Xmx512m
      # Set a deterministic node name.
      node.name: elasticsearch03
      # Use other cluster nodes for unicast discovery.
      discovery.seed_hosts: elasticsearch01,elasticsearch02
      # Define initial masters, assuming a cluster size of at least 3.
      cluster.initial_master_nodes: elasticsearch01,elasticsearch02,elasticsearch03
    networks:
      - elk

volumes:
  elasticsearch01:
  elasticsearch02:
  elasticsearch03:

Swarm mode

Both the discovery.seed_hosts and cluster.initial_master_nodes settings are necessary to bootstrap a cluster. It is possible to leverage the Docker internal DNS together with unicast Zen discovery mechanism in order to discover current cluster nodes. For that, simply set the discovery.seed_hosts Elasticsearch setting to the name of your Elasticsearch task, either in the elasticsearch.yml configuration file or via an environment variable.

For more information, see Important discovery and cluster formation settings.

Example (Swarm mode):

# docker-stack.yml

  elasticsearch:
    environment:
      # Force publishing on the 'elk' overlay.
      network.publish_host: _eth0_
      # Set a deterministic node name.
      node.name: elk_elasticsearch.{{.Task.Slot}}
      # Use internal Docker round-robin DNS for unicast discovery.
      discovery.seed_hosts: tasks.elasticsearch
      # Define initial masters, assuming a cluster size of at least 3.
      cluster.initial_master_nodes: elk_elasticsearch.1,elk_elasticsearch.2,elk_elasticsearch.3

Port mapping

The default docker-compose file uses static host port mapping for the elasticsearch service. This prevents scaling services because a single port can be mapped only once on the host. Instead, you have to either disable port mapping completely, or let Docker map container ports to random host ports in order to prevent clashes.

Example:

# docker-compose.yml

  elasticsearch:
    ports:
      # map to a random host port instead of a static port, eg. 32000:9200
      - '9200'

Data persistence

    Warning
    This section needs an update. The node.max_local_storage_nodes setting was removed in Elastic v8 and sharing the data path is no longer supported.

In the default configuration, multiple Elasticsearch nodes are not allowed to share the same data volume. This limitation can be removed by setting the node.max_local_storage_nodes setting to the number of Elasticsearch nodes in the cluster.

Example:

# docker-compose.yml

  elasticsearch:
    environment:
      node.max_local_storage_nodes: '3'

Scaling out

Start the ELK stack with multiple elasticsearch replicas:

$ docker-compose up -d --scale elasticsearch=3
Creating docker-elk_elasticsearch_1 ... done
Creating docker-elk_elasticsearch_2 ... done
Creating docker-elk_elasticsearch_3 ... done
Creating docker-elk_kibana_1        ... done
Creating docker-elk_logstash_1      ... done

Verification

The cluster should bootstrap:

$ docker-compose logs elasticsearch02

(prettyfied log entry)

{
  "@timestamp": "2022-11-17T17:00:45.768Z",
  "log.level": "INFO",
  "message": "master node changed {previous [], current [{elasticsearch02}{nNcPzsX_Suqhf8a1e_-reg}{yoPMF3tcRwiAoAVA4dARvQ}{elasticsearch02}{172.21.0.2}{172.21.0.2:9300}{cdfhilmrstw}]}, added {{elasticsearch02}{nNcPzsX_Suqhf8a1e_-reg}{yoPMF3tcRwiAoAVA4dARvQ}{elasticsearch02}{172.21.0.2}{172.21.0.2:9300}{cdfhilmrstw}, {elasticsearch03}{83n5XFMrSnuF4Qo5rQUB7Q}{nQphI1KSRkSBGaApKIMyjA}{elasticsearch03}{172.21.0.3}{172.21.0.3:9300}{cdfhilmrstw}}, term: 2, version: 27, reason: ApplyCommitRequest{term=2, version=27, sourceNode={elasticsearch02}{nNcPzsX_Suqhf8a1e_-reg}{yoPMF3tcRwiAoAVA4dARvQ}{elasticsearch02}{172.21.0.2}{172.21.0.2:9300}{cdfhilmrstw}{ml.allocated_processors_double=4.0, ml.machine_memory=4088406016, ml.max_jvm_size=536870912, xpack.installed=true, ml.allocated_processors=4}}",
  "ecs.version": "1.2.0",
  "service.name": "ES_ECS",
  "event.dataset": "elasticsearch.server",
  "process.thread.name": "elasticsearch[elasticsearch01][clusterApplierService#updateTask][T#1]",
  "log.logger": "org.elasticsearch.cluster.service.ClusterApplierService",
  "elasticsearch.node.name": "elasticsearch01",
  "elasticsearch.cluster.name": "docker-cluster"
}

All nodes will show up in Kibana's Monitoring app:

ES cluster Kibana
Add a custom footer
Pages 3

Home
Elasticsearch cluster

    Scaling out Elasticsearch
    Prerequisites
    Increase vm.max_map_count
    Configuration
    Compose mode
    Swarm mode
    Port mapping
    Data persistence
    Scaling out
    Verification

    External applications

Add a custom sidebar
Clone this wiki locally
