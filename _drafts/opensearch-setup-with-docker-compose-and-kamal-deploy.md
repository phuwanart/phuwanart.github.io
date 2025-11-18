---
layout: post
title: OpenSearch setup with Docker Compose and Kamal Deploy
categories: Posts
tags:
  - docker
  - kamal
  - rails
  - opensearch
---

## Getting Started

### Create Rails Application

#### Searchkick

```ruby
Searchkick.client = OpenSearch::Client.new(
  host: ENV.fetch("OPEN_SEARCH_HOST", "https://blog-opensearch:9200"),
  user: "admin",
  password: ENV.fetch("OPENSEARCH_INITIAL_ADMIN_PASSWORD", "SamplePassword1!"),
  retry_on_failure: true,
  transport_options: {
    ssl: {
      verify: false
    },
    request: {
      timeout: 250
    }
  }
)

Searchkick.index_prefix = Rails.application.class.module_parent_name.underscore
```
{: file="config/initializers/searchkick.rb"}


### [Docker Compose](https://opensearch.org/downloads/)

```yaml
---
services:
  opensearch-node1:
    image: opensearchproject/opensearch:latest
    container_name: opensearch-node1
    environment:
      - cluster.name=opensearch-cluster
      - node.name=opensearch-node1
      - discovery.seed_hosts=opensearch-node1,opensearch-node2
      - cluster.initial_cluster_manager_nodes=opensearch-node1,opensearch-node2
      - bootstrap.memory_lock=true
      - OPENSEARCH_JAVA_OPTS=-Xms512m -Xmx512m
      - OPENSEARCH_INITIAL_ADMIN_PASSWORD=${OPENSEARCH_INITIAL_ADMIN_PASSWORD}
    ulimits:
      memlock:
        soft: -1
        hard: -1
      nofile:
        soft: 65536
        hard: 65536
    volumes:
      - ./certs:/usr/share/opensearch/config/certs
      - opensearch-data1:/usr/share/opensearch/data
    ports:
      - 9200:9200
      - 9600:9600
    networks:
      - opensearch-net
  opensearch-node2:
    image: opensearchproject/opensearch:latest
    container_name: opensearch-node2
    environment:
      - cluster.name=opensearch-cluster
      - node.name=opensearch-node2
      - discovery.seed_hosts=opensearch-node1,opensearch-node2
      - cluster.initial_cluster_manager_nodes=opensearch-node1,opensearch-node2
      - bootstrap.memory_lock=true
      - OPENSEARCH_JAVA_OPTS=-Xms512m -Xmx512m
      - OPENSEARCH_INITIAL_ADMIN_PASSWORD=${OPENSEARCH_INITIAL_ADMIN_PASSWORD}
    ulimits:
      memlock:
        soft: -1
        hard: -1
      nofile:
        soft: 65536
        hard: 65536
    volumes:
      - ./certs:/usr/share/opensearch/config/certs
      - opensearch-data2:/usr/share/opensearch/data
    networks:
      - opensearch-net
  opensearch-dashboards:
    image: opensearchproject/opensearch-dashboards:latest
    container_name: opensearch-dashboards
    ports:
      - 5601:5601
    expose:
      - '5601'
    environment:
      OPENSEARCH_HOSTS: '["https://opensearch-node1:9200","https://opensearch-node2:9200"]'
    networks:
      - opensearch-net

volumes:
  opensearch-data1:
  opensearch-data2:

networks:
  opensearch-net:
```
{: file="docker-compose.yml"}

```sh
docker compose up -d
```

```sh
docker compose down
```

### Kamal

```yaml
accessories:
  opensearch:
    image: opensearchproject/opensearch:latest
    host: 157.245.197.146
    port: 9200
    env:
      clear:
        discovery.type: single-node
        OPENSEARCH_INITIAL_ADMIN_PASSWORD: SamplePassword1!
    directories:
      - data:/usr/share/opensearch/data
```
{: file="config/deploy.yml"}


## Conclusion

## References

- https://youtu.be/uEv8qxQv7OM?si=oAhhnVjx2SY9p141
- https://github.com/Instadeploy/logging-opensearch
