# L2 DevOps Practical Assignment

A locally runnable Docker Compose environment demonstrating:

- Docker containerization
- Docker Compose orchestration
- Nginx reverse proxy
- Docker bridge networking
- Centralized logging with Logstash + Elasticsearch + Kibana
- Prometheus metrics
- Grafana dashboard
- Container health checks and restart policies

## 1. Architecture

```text
Browser
   |
   | http://localhost:8080
   v
Nginx :80
   |
   | Docker DNS: app:80
   v
Sample App (Traefik whoami)

Application/Nginx stdout
   |
   | Docker syslog logging driver
   v
Logstash :5000
   |
   v
Elasticsearch :9200
   |
   v
Kibana :5601

Metrics:
Nginx -> nginx-prometheus-exporter -> Prometheus -> Grafana
Containers -> cAdvisor -> Prometheus -> Grafana
```

## 2. Why this sample application?

The assignment explicitly says application development is not required. The Traefik `whoami`
image is a lightweight pre-built web application that returns request and container
information. A local Dockerfile is still provided so the application is built as part of
the Compose environment.

## 3. Prerequisites

- Docker Desktop or Docker Engine
- Docker Compose v2
- At least 4 GB RAM available to Docker
- Ports 3000, 5000, 5601, 8080, 9090 and 9200 available

Check:

```bash
docker --version
docker compose version
```

## 4. Start the complete environment

From the repository root:

```bash
docker compose up -d --build
```

Check status:

```bash
docker compose ps
```

Follow startup logs:

```bash
docker compose logs -f
```

## 5. Access the services

| Service | URL | Purpose |
|---|---|---|
| Nginx / Application | http://localhost:8080 | Main application entrypoint |
| Kibana | http://localhost:5601 | Search/view centralized logs |
| Prometheus | http://localhost:9090 | Metrics |
| Grafana | http://localhost:3000 | Monitoring dashboard |
| Elasticsearch | http://localhost:9200 | Log storage/API |

Grafana credentials:

```text
Username: admin
Password: admin
```

## 6. Validate Nginx reverse proxy

Run:

```bash
curl http://localhost:8080
```

The response should contain information from the `whoami` application.

The browser only accesses Nginx. Nginx resolves the Docker DNS name `app` and forwards
the request to the application container on port 80.

Validate Nginx health:

```bash
curl http://localhost:8080/nginx-health
```

Expected:

```text
healthy
```

## 7. Docker networking

Three logical bridge networks are used:

### frontend

Connects:

```text
nginx <-> app
```

### logging

Connects:

```text
app -> Logstash -> Elasticsearch -> Kibana
nginx -> Logstash
```

### monitoring

Connects:

```text
nginx -> nginx-exporter -> Prometheus
cadvisor -> Prometheus
Prometheus -> Grafana
```

This limits unnecessary connectivity between services and demonstrates separation of
traffic by purpose.

Inspect networks:

```bash
docker network ls
docker network inspect devops-l2-assignment_frontend
docker network inspect devops-l2-assignment_logging
docker network inspect devops-l2-assignment_monitoring
```

Compose normally creates the network names from the project directory.

## 8. Logging flow

The application and Nginx write logs to stdout/stderr.

Docker uses the `syslog` logging driver and sends the records to:

```text
Logstash:5000/TCP
```

Logstash receives the records through its `syslog` input and writes them to:

```text
Elasticsearch index:
devops-logs-YYYY.MM.dd
```

Kibana reads Elasticsearch and makes the logs searchable.

Generate traffic:

```bash
for i in $(seq 1 20); do curl -s http://localhost:8080 > /dev/null; done
```

Check Logstash:

```bash
docker compose logs logstash --tail=50
```

Check Elasticsearch:

```bash
curl http://localhost:9200/_cat/indices?v
```

You should see an index beginning with:

```text
devops-logs-
```

### Kibana validation

Open:

```text
http://localhost:5601
```

Create a data view:

```text
Stack Management
  -> Data Views
  -> Create data view
```

Use:

```text
devops-logs-*
```

Select `@timestamp` as the time field if prompted.

Then open:

```text
Discover
```

Search/filter for:

```text
service:sample-app
```

or:

```text
tag:nginx
```

## 9. Metrics flow

### Nginx metrics

Nginx exposes its `stub_status` endpoint internally:

```text
nginx:80/nginx_status
```

The exporter converts those metrics into Prometheus format:

```text
nginx -> nginx-exporter:9113 -> Prometheus
```

### Container metrics

cAdvisor reads Docker container CPU and memory statistics:

```text
Docker containers -> cAdvisor -> Prometheus
```

Prometheus configuration is in:

```text
prometheus/prometheus.yml
```

Validate Prometheus targets:

```text
http://localhost:9090/targets
```

Both `nginx` and `cadvisor` targets should show `UP`.

You can also query:

```text
nginx_connections_active
```

and:

```text
rate(nginx_http_requests_total[5m])
```

## 10. Grafana validation

Open:

```text
http://localhost:3000
```

Login:

```text
admin / admin
```

The Prometheus datasource is automatically provisioned.

The dashboard is automatically provisioned:

```text
DevOps
  -> DevOps L2 - Local Infrastructure
```

Dashboard panels include:

- Container CPU usage
- Container memory usage
- Nginx active connections
- Nginx request rate

Generate traffic to create visible metrics:

```bash
for i in $(seq 1 100); do
  curl -s http://localhost:8080 > /dev/null
done
```

## 11. Health checks

Health checks are configured for:

- Sample application
- Nginx
- Elasticsearch
- Logstash
- Kibana
- Prometheus
- Grafana

Check:

```bash
docker compose ps
```

A healthy service should show:

```text
healthy
```

Health checks are combined with Compose `depends_on` conditions where startup ordering
matters.

## 12. Common troubleshooting

### Application is not reachable

```bash
docker compose ps
docker compose logs app
docker compose logs nginx
```

Test from inside Nginx:

```bash
docker exec nginx wget -q -O - http://app
```

If this fails, inspect the frontend network:

```bash
docker network inspect devops-l2-assignment_frontend
```

### Nginx cannot resolve app

Check:

```bash
docker exec nginx getent hosts app
```

The Docker Compose service name `app` is used as the DNS hostname.

### Elasticsearch is unhealthy

```bash
docker compose logs elasticsearch
curl http://localhost:9200/_cluster/health
```

Elasticsearch is configured as a single node with security disabled for this local
evaluation environment.

### Logs are not appearing in Kibana

Check:

```bash
docker compose logs logstash
docker compose logs elasticsearch
curl http://localhost:9200/_cat/indices?v
```

Generate new application requests:

```bash
curl http://localhost:8080
```

Then verify the `devops-logs-*` index.

### Prometheus target is DOWN

Open:

```text
http://localhost:9090/targets
```

Then inspect:

```bash
docker compose logs prometheus
docker compose logs nginx-exporter
docker compose logs cadvisor
```

### Grafana has no data

Check Prometheus first:

```text
http://localhost:9090/targets
```

If targets are UP, open Grafana:

```text
Connections -> Data sources -> Prometheus
```

The configured URL inside the Docker network is:

```text
http://prometheus:9090
```

## 13. Recreate the environment

To stop:

```bash
docker compose down
```

To recreate containers:

```bash
docker compose down
docker compose up -d --build
```

To remove containers and persistent data:

```bash
docker compose down -v
```

`down -v` deletes Elasticsearch, Prometheus and Grafana data, so it should only be used
when a clean reset is required.

## 14. Security considerations

This is intentionally a local evaluation environment.

For production:

- Enable Elasticsearch/Kibana authentication and TLS.
- Do not expose Elasticsearch directly to the public network.
- Store Grafana credentials using secrets rather than plain Compose environment values.
- Run containers with least privilege.
- Avoid `privileged: true` for cAdvisor where an alternative deployment is suitable.
- Restrict exposed ports with a firewall/security group.
- Pin and regularly update image versions.
- Add resource limits.
- Use HTTPS/TLS at the reverse proxy.
- Use structured JSON logging and explicit retention policies.

## 15. Rollback approach

Images are version-pinned where practical. For a failed application image deployment:

1. Identify the previously working image tag.
2. Update the application image/Dockerfile to that known version.
3. Rebuild:
   ```bash
   docker compose up -d --build app
   ```
4. Validate:
   ```bash
   curl http://localhost:8080
   docker compose ps
   ```
5. Review logs and metrics in Kibana/Grafana.

## 16. Submission checklist

- [x] Sample pre-built application
- [x] Dockerfile
- [x] Docker Compose
- [x] Nginx reverse proxy
- [x] Docker networking
- [x] Logstash
- [x] Elasticsearch
- [x] Kibana
- [x] Centralized searchable logs
- [x] Prometheus
- [x] Grafana
- [x] Monitoring dashboard
- [x] Health checks
- [x] Restart policies
- [x] Persistent storage
- [x] Validation commands
- [x] Troubleshooting guide
- [x] Security considerations
- [x] Rollback approach
