# Architecture

```text
Browser
   |
   v
Nginx :8080
   |
   v
Sample App :80

App/Nginx logs
   |
   v
Docker syslog -> Logstash :5000 -> Elasticsearch :9200 -> Kibana :5601

Nginx metrics -> nginx-exporter :9113 -> Prometheus :9090 -> Grafana :3000
Container metrics -> cAdvisor :8080 -> Prometheus
```
