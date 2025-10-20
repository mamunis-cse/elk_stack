🧠 ELK Stack Setup (Elasticsearch + Logstash + Kibana + Filebeat)
📌 1. What is ELK?

ELK Stack is a set of tools used to collect, process, store, and visualize logs efficiently.
It consists of:

E → Elasticsearch – Database for storing and searching log data

L → Logstash – Processes and formats logs before sending them to Elasticsearch

K → Kibana – Used for visualization and dashboard analytics

⚙️ 2. Why Use ELK?

Centralized logging system

Real-time log analysis and visualization

Easy troubleshooting and alerting

Scalable and open-source solution

🧩 3. Agent: Filebeat

Filebeat acts as the log shipper (agent).
It collects logs from servers and sends them to Logstash or Elasticsearch.

Workflow:

Log Source → Filebeat → Logstash (Grok Filter) → Elasticsearch → Kibana
