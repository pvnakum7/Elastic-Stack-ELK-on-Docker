# Connecting external applications

This page describes how to consume the ELK services from applications started outside of the ELK stack Compose.

## General procedure

### About the default bridge network

Docker Compose creates a bridge network used by the services from the ELK stack. This network is named after the directory that contains the `docker-compose.yml` file (without hyphens or underscores), followed by the predefined network name `_elk`.

For instance, if you cloned the repository inside a directory called `elk-stack`, the name of the bridge network dedicated to the ELK stack will be `elkstack_elk`.

To list the networks available to Docker, execute the following command:

```console
$ docker network ls
NETWORK ID          NAME              DRIVER              SCOPE
4e7f482eec96        bridge            bridge              local
16a574ecd253        dockerelk_elk     bridge              local
6507e865135e        host              host                local
87535838ad3a        none              null                local
```

### Connecting containers to an arbitrary network

In order to be able to communicate with the services from the ELK stack, external containers must be connected to the same bridge network as the one used by these services.

From the Docker CLI, this translates to:

```console
$ docker run --name myapp --network dockerelk_elk nginx:alpine
```

In a Docker Compose file, one can specify [`external` networks](https://docs.docker.com/compose/compose-file/#external-1) in order to use networks created outside of the current Compose:

```yaml
# docker-compose.yml

services:
  myapp:
    image: nginx:alpine
    networks:
      - elk

networks:
  elk:
    external:
      name: dockerelk_elk
```

In both cases, one can verify that the name of the ELK services can actually be resolved from the newly created container:

```console
$ docker exec myapp nslookup logstash
Server:         127.0.0.11
Address:        127.0.0.11#53

Non-authoritative answer:
Name:   logstash
Address: 172.18.0.3
```

## Examples

### Cerebro

[Cerebro](https://github.com/lmenezes/cerebro) is a web administration tool for Elasticsearch. It replaces the older [kopf](https://github.com/lmenezes/elasticsearch-kopf) plugin.

Launch the application with the following Docker Compose file:

```yaml
# docker-compose.yml

version: '3'

services:
  cerebro:
    image: yannart/cerebro
    ports:
      - "9000:9000"
    networks:
      - elk

networks:
  elk:
    external:
      name: dockerelk_elk
```

Open your browser at the following address: http://localhost:9000, and use the following host address in the *Hosts* field: `http://elasticsearch:9200`.

![cerebro_1](https://cloud.githubusercontent.com/assets/3299086/23918699/40bb19d2-08f4-11e7-8229-24f33f501746.png)
![cerebro_2](https://cloud.githubusercontent.com/assets/3299086/23918698/40b9a872-08f4-11e7-8e50-309253436007.png)