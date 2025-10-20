# 🧠 ELK Stack Setup (Elasticsearch + Logstash + Kibana + Filebeat)

## 📌 1. What is ELK?

ELK Stack is a set of tools used to collect, process, store, and visualize logs efficiently.
It consists of:

* **E → Elasticsearch** – Database for storing and searching log data
* **L → Logstash** – Processes and formats logs before sending them to Elasticsearch
* **K → Kibana** – Used for visualization and dashboard analytics

## ⚙️ 2. Why Use ELK?

* Centralized logging system
* Real-time log analysis and visualization
* Easy troubleshooting and alerting
* Scalable and open-source solution

## 🧩 3. Agent: Filebeat

Filebeat acts as the log shipper (agent).
It collects logs from servers and sends them to Logstash or Elasticsearch.

**Workflow:**

```
Log Source → Filebeat → Logstash (Grok Filter) → Elasticsearch → Kibana
```

## 🐳 4. Docker Compose Setup

Create a file named `docker-compose.yml`:

```yaml
version: "3.8"

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
      - "5601:5601"

  logstash:
    image: docker.elastic.co/logstash/logstash:${VERSION}
    container_name: logstash
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
      - ELASTIC_USER=${ELASTIC_USER}
      - ELASTIC_PASSWORD=${ELASTIC_PASSWORD}
    ports:
      - "5044:5044"
    volumes:
      - ./logstash/pipeline/logstash.conf:/usr/share/logstash/pipeline/logstash.conf

volumes:
  esdata:
```

## 🧾 5. Logstash Configuration

Create `logstash/pipeline/logstash.conf`:

```ruby
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

## 🔐 6. Environment File

Create a `.env` file in the same directory:

```env
VERSION=9.1.5
ELASTIC_USER=elastic
ELASTIC_PASSWORD=YourElasticPassword
KIBANA_PASSWORD=YourKibanaPassword
```

## 🚀 7. Run the Stack

```bash
docker compose up -d
```

✅ Once containers are running:

* Visit [http://localhost:5601](http://localhost:5601) for Kibana
* Login with username: `elastic` and password from `.env`

⚠️ If Kibana authentication error occurs (AuthorizationException):

```bash
docker exec -it kibana bash
# Inside container, reset kibana_system user password if needed
```

## 📡 8. Filebeat Configuration

Install Filebeat (if not using Docker):

```bash
sudo apt install filebeat -y
```

Enable and configure Filebeat:

* Enable log collection (e.g., nginx logs)
* Enable Logstash output
* Set host and port (5044)

Example path: `/etc/filebeat/filebeat.yml`

Restart Filebeat service:

```bash
sudo service filebeat restart
```

## 📊 9. View Logs in Kibana

1. Open Kibana → **Management → Index Patterns**
2. Create index: `nginx-access-log-*`
3. Go to **Discover** or **Dashboard** to visualize logs

## 🧠 Quick Notes

* Use **Grok filter** in Logstash for custom log parsing.
* Filebeat can send directly to Elasticsearch or via Logstash.
* Keep Elasticsearch memory at least **2GB** for stable performance.

## 🔰 Example Workflow Summary

```
Filebeat → Logstash → Elasticsearch → Kibana → Dashboard Visualization
```
