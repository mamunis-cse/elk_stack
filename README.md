# ELK Stack — (Step-by-step)

> This file is ready to be used in your Git repository. It contains every step, configuration, and command needed to run the ELK Stack using Docker Compose.

---

## Contents

1. Overview
2. Project Folder Structure
3. `.env` File
4. `docker-compose.yml`
5. Logstash Pipeline: `logstash.conf`
6. Filebeat Config: `filebeat.yml`
7. How to Run (Commands)
8. Common Issues & Fixes (like Kibana authentication)
9. Git Add & Push Instructions

---

## 1) Overview

**ELK = Elasticsearch + Logstash + Kibana**

* **Elasticsearch:** Acts as a database and search engine.
* **Logstash:** Processes and formats logs using GROK filters.
* **Kibana:** Visualizes data and dashboards.
* **Filebeat:** A lightweight agent that ships logs from servers to Logstash or Elasticsearch.

**Workflow:**

```
Log source → Filebeat → Logstash (GROK filter) → Elasticsearch → Kibana
```

---

## 2) Recommended Project Structure

```
elk-project/
├── .env
├── docker-compose.yml
├── logstash/
│   └── pipeline/
│       └── logstash.conf
├── filebeat/
│   └── filebeat.yml
└── README.md  (this file)
```

---

## 3) `.env` File

> ⚠️ Note: Use strong passwords. If you push to GitHub, consider adding `.env` to `.gitignore`.

```env
VERSION=9.1.5
ELASTIC_USER=elastic
ELASTIC_PASSWORD=ChangeMeElasticPass
KIBANA_PASSWORD=ChangeMeKibanaPass
```

---

## 4) `docker-compose.yml`

```yaml
version: '3.8'

services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:${VERSION}
    container_name: elasticsearch
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=true
      - ELASTIC_PASSWORD=${ELASTIC_PASSWORD}
      - ES_JAVA_OPTS=-Xms1g -Xmx1g
    volumes:
      - esdata:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"

  kibana:
    image: docker.elastic.co/kibana/kibana:${VERSION}
    container_name: kibana
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
      - ELASTICSEARCH_USERNAME=kibana_system
      - ELASTICSEARCH_PASSWORD=${KIBANA_PASSWORD}
    ports:
      - "5605:5601"
    depends_on:
      - elasticsearch

  logstash:
    image: docker.elastic.co/logstash/logstash:${VERSION}
    container_name: logstash
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
      - ELASTIC_USER=${ELASTIC_USER}
      - ELASTIC_PASSWORD=${ELASTIC_PASSWORD}
    ports:
      - "5044:5044"      # Beats input
    volumes:
      - ./logstash/pipeline/logstash.conf:/usr/share/logstash/pipeline/logstash.conf:ro
    depends_on:
      - elasticsearch

  filebeat:
    image: docker.elastic.co/beats/filebeat:${VERSION}
    container_name: filebeat
    user: root
    volumes:
      - ./filebeat/filebeat.yml:/usr/share/filebeat/filebeat.yml:ro
      - /var/log:/var/log:ro
    depends_on:
      - logstash

volumes:
  esdata:
```

> **Note:** Kibana port mapped as `5605:5601` — Access via `http://localhost:5605`

---

## 5) `logstash.conf` (Example for Nginx Access Logs)

`logstash/pipeline/logstash.conf`

```conf
input {
  beats {
    port => 5044
  }
}

filter {
  grok {
    match => {
      "message" => [
        "%{IPORHOST:remote_addr} - %{DATA:remote_user} \[%{HTTPDATE:time_local}\] \"%{DATA:request}\" %{NUMBER:status:int} %{NUMBER:bytes:int} \"%{DATA:referrer}\" \"%{DATA:agent}\""
      ]
    }
    tag_on_failure => []
  }

  grok {
    match => { "request" => "%{WORD:method} %{URIPATHPARAM:url} HTTP/%{NUMBER:http_version:float}" }
    tag_on_failure => []
  }

  date {
    match => ["time_local", "dd/MMM/YYYY:H:m:s Z"]
    remove_field => ["time_local"]
  }

  mutate {
    convert => { "bytes" => "integer" "status" => "integer" }
    rename => { "bytes" => "body_bytes_sent" }
    add_field => { "service" => "nginx" }
  }
}

output {
  elasticsearch {
    hosts => ["http://elasticsearch:9200"]
    user => "${ELASTIC_USER}"
    password => "${ELASTIC_PASSWORD}"
    index => "nginx-access-log-%{+YYYY.MM.dd}"
  }
}
```

---

## 6) `filebeat.yml` (Basic Example)

```yaml
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/nginx/access.log

output.logstash:
  hosts: ["logstash:5044"]
```

Then restart Filebeat container:

```bash
docker compose restart filebeat
```

---

## 7) Run the Stack

```bash
docker compose up -d
```

Check container status:

```bash
docker ps
```

Visit:

* **Elasticsearch:** [http://localhost:9200](http://localhost:9200)
* **Kibana:** [http://localhost:5605](http://localhost:5605)

---

## 8) Common Issues

**Kibana Authorization Error**
If you see: `authorization_exception`, login into Kibana container and reset `kibana_system` password.

```bash
docker exec -it kibana /bin/bash
bin/kibana-keystore add elasticsearch.password
```

Then restart Kibana:

```bash
docker compose restart kibana
```

---

✅ **Now your ELK Stack is ready!**
Once all services are running, check Kibana → *Index Patterns* → verify your Filebeat/Logstash indices and start building dashboards.
