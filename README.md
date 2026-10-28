# DevOps Lab 3 — Infrastructure Automation, Logging & Monitoring

**Author:** Shashank Sendil Kumar

**Repository:** [github.com/shashank12004/devops-lab-3](https://github.com/shashank12004)

---

## Overview

This repository contains the complete implementation for **DevOps Lab 3**, demonstrating automated infrastructure configuration, container provisioning and horizontal scaling, centralized log processing, and real-time metric monitoring.

The lab is structured into four primary modules:

1. **Task 1 — Infrastructure Automation with Ansible:** Automated web server setup and idempotent application deployment[cite: 11].
2. **Task 2 — Infrastructure Provisioning with Terraform:** Dynamic Docker container deployment and load balancer auto-configuration with variable-based scaling[cite: 14].
3. **Task 3 — Centralized Logging with the ELK Stack:** End-to-end log aggregation pipeline using Nginx, Filebeat, Logstash, Elasticsearch, and Kibana[cite: 22].
4. **Monitoring — Metric Collection with Prometheus & Grafana:** System and application performance monitoring with Node Exporter, Nginx Exporter, Prometheus, and Grafana dashboards[cite: 1].

---

## Repository Structure

```tree
devops-lab3/
├── ansible/
│   ├── inventory.ini
│   └── site.yml
├── terraform/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   └── nginx.conf.tpl
├── elk/
│   ├── docker-compose.yml
│   ├── filebeat/
│   │   └── filebeat.yml
│   └── logstash/
│       └── pipeline/
│           └── logstash.conf
└── monitoring/
    ├── docker-compose.yml
    ├── nginx/
    │   └── default.conf
    ├── prometheus/
    │   └── prometheus.yml
    └── grafana/
        └── provisioning/
            └── datasources/
                └── prometheus.yml

```

---

## Port Mapping Reference Table

| Service | Host Port | Container Port | Purpose |
| --- | --- | --- | --- |
| **Terraform Nginx LB** | `8080` | `80` | Load balancer entry point for web replicas[cite: 15, 17] |
| **ELK Nginx Web** | `8081` | `80` | Access log generator[cite: 24] |
| **Logstash** | `5044` | `5044` | Beats receiver pipeline[cite: 24] |
| **Elasticsearch** | `9200` | `9200` | Search engine & index storage[cite: 24] |
| **Kibana** | `5601` | `5601` | Log exploration dashboard UI[cite: 24] |
| **Monitoring Nginx Web** | `8082` | `80` | Target web server[cite: 3] |
| **Prometheus** | `9090` | `9090` | Time-series metric database & PromQL UI[cite: 3] |
| **Grafana** | `3000` | `3000` | Metric visualization dashboard UI[cite: 3] |
| **Nginx Exporter** | `9113` | `9113` | Exposes Nginx `stub_status` as Prometheus metrics[cite: 3, 4] |

---

## Task 1: Infrastructure Automation Using Ansible

### Objective

Install Ansible on the control node, configure target inventory, write an automated playbook (`site.yml`), and apply it to ensure idempotent web server setup[cite: 11].

### Key Files & Commands

#### `ansible/inventory.ini`

```ini
[webservers]
localhost ansible_connection=local
```[cite: 13]

#### `ansible/site.yml`
```yaml
---
- name: Configure web server and deploy application
  hosts: webservers
  become: true
  vars:
    app_message: "Deployed with Ansible by Akash Singh Sagar"

  tasks:
    - name: Update apt cache
      ansible.builtin.apt:
        update_cache: true
        cache_valid_time: 3600

    - name: Install nginx and git
      ansible.builtin.apt:
        name: [nginx, git]
        state: present

    - name: Deploy index page
      ansible.builtin.copy:
        content: "<h1>{{ app_message }}</h1>\n"
        dest: /var/www/html/index.html
        mode: "0644"
      notify: Restart nginx

    - name: Ensure nginx is enabled and running
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: true

  handlers:
    - name: Restart nginx
      ansible.builtin.service:
        name: nginx
        state: restarted
```[cite: 13]

### Execution Steps
```bash
# Verify connectivity
ansible all -i inventory.ini -m ping

# Run syntax check and apply playbook
ansible-playbook -i inventory.ini site.yml --syntax-check
ansible-playbook -i inventory.ini site.yml

# Re-run to verify idempotency (changed=0)
ansible-playbook -i inventory.ini site.yml
```[cite: 13]

---

## Task 2: Scalable Infrastructure Provisioning Using Terraform

### Objective
Provision a scalable web cluster using Terraform and the Docker provider, placing multiple Nginx replica containers behind an auto-configured Nginx reverse proxy load balancer[cite: 14].

### Key Configurations

#### `terraform/variables.tf`
```hcl
variable "replica_count" {
  type        = number
  description = "Number of web server containers"
  default     = 2
}

variable "lb_port" {
  type        = number
  description = "Host port for the load balancer"
  default     = 8080
}
```[cite: 15]

#### `terraform/nginx.conf.tpl`
```nginx
events {}
http {
    upstream web {
%{ for s in servers ~}
        server ${s}:80;
%{ endfor ~}
    }
    server {
        listen 80;
        location / {
            proxy_pass http://web;
        }
    }
}
```[cite: 17]

### Execution & Scaling Commands
```bash
cd terraform

# Initialize, format, validate, and plan
terraform init
terraform fmt
terraform validate
terraform plan -out=tfplan

# Deploy initial stack (2 replicas)
terraform apply tfplan

# Test load balancer round-robin distribution
for i in {1..4}; do curl -s http://localhost:8080; echo; done

# Scale out cluster to 4 replicas
terraform apply -var="replica_count=4" -auto-approve

# Clean up infrastructure
terraform destroy -var="replica_count=4" -auto-approve
```[cite: 18, 19, 20, 21]

---

## Task 3: Centralized Logging Using the ELK Stack

### Objective
Implement central log harvesting, grok-based parsing, indexed storage, and visualization using Filebeat, Logstash, Elasticsearch, and Kibana[cite: 22].

### Architecture Pipeline
$$\text{Nginx Container } (\texttt{access.log}) \longrightarrow \text{Filebeat} \longrightarrow \text{Logstash Grok Filter} \longrightarrow \text{Elasticsearch} \longrightarrow \text{Kibana UI}$$
[cite: 22]

### Key Configurations

#### `elk/filebeat/filebeat.yml`
```yaml
filebeat.inputs:
  - type: filestream
    id: nginx-access
    paths:
      - /var/log/nginx/access.log
    fields:
      service: nginx
    fields_under_root: true

output.logstash:
  hosts: ["logstash:5044"]
```[cite: 24]

#### `elk/logstash/pipeline/logstash.conf`
```ruby
input {
  beats {
    port => 5044
  }
}

filter {
  grok {
    match => { "message" => '%{IPORHOST:client_ip} - %{DATA:user} \[%{HTTPDATE:log_time}\] "%{WORD:method} %{DATA:path} HTTP/%{NUMBER:http_version}" %{NUMBER:status:int} %{NUMBER:bytes:int} "%{DATA:referrer}" "%{DATA:user_agent_raw}"' }
  }

  date {
    match  => ["log_time", "dd/MMM/yyyy:HH:mm:ss Z"]
    target => "@timestamp"
  }
}

output {
  elasticsearch {
    hosts => ["http://elasticsearch:9200"]
    index => "nginx-logs-%{+YYYY.MM.dd}"
  }
}
```[cite: 25]

### Execution & Verification Commands (PowerShell / Windows)
```powershell
# Set required Elasticsearch memory limit
wsl -d docker-desktop sysctl -w vm.max_map_count=262144

# Start ELK stack
cd "$HOME\devops-lab3\elk"
docker compose up -d

# Verify cluster health
curl.exe http://localhost:9200/_cluster/health?pretty

# Generate sample web traffic (mix of 200 OK and 404 Not Found)
1..100 | ForEach-Object {
    $rand = Get-Random -Minimum 0 -Maximum 5
    curl.exe -s -o NUL http://localhost:8081/
    curl.exe -s -o NUL "http://localhost:8081/missing-$rand"
}

# Verify indices and parsed document fields in Elasticsearch
curl.exe "http://localhost:9200/_cat/indices/nginx-logs-*?v"
curl.exe "http://localhost:9200/nginx-logs-*/_search?size=1&pretty"
```[cite: 23, 26]

---

## Monitoring Task: Metric Collection with Prometheus & Grafana

### Objective
Expose metric endpoints for host and application status using Node Exporter and Nginx Exporter, aggregate time-series data using Prometheus, and display real-time performance panels in Grafana[cite: 1].

### Key Configurations

#### `monitoring/nginx/default.conf`
```nginx
server {
    listen 80;
    location / {
        root /usr/share/nginx/html;
        index index.html;
    }
}

server {
    listen 8080;
    location /stub_status {
        stub_status;
    }
}
```[cite: 2, 3]

#### `monitoring/prometheus/prometheus.yml`
```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: node
    static_configs:
      - targets: ["node-exporter:9100"]

  - job_name: nginx
    static_configs:
      - targets: ["nginx-exporter:9113"]
```[cite: 4]

### Dashboard PromQL Panel Configuration Table

| Panel Title | PromQL Query | Visualization Type |
| :--- | :--- | :--- |
| **Targets Up** | `up` | Stat[cite: 10] |
| **CPU Usage (%)** | `100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[1m])) * 100)` | Time series[cite: 10] |
| **Memory Usage (%)** | `(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100` | Gauge[cite: 10] |
| **Disk Usage (%)** | `100 - (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"} * 100)` | Gauge[cite: 10] |
| **Network Received** | `rate(node_network_receive_bytes_total{device!="lo"}[1m])` | Time series[cite: 10] |
| **Nginx Requests/sec** | `rate(nginx_http_requests_total[1m])` | Time series[cite: 10] |
| **Nginx Active Connections** | `nginx_connections_active` | Stat[cite: 10] |

### Execution & Verification Commands
```powershell
cd "$HOME\devops-lab3\monitoring"

# Launch monitoring stack
docker compose up -d

# Verify all targets (prometheus, node, nginx) report UP
curl.exe "http://localhost:9090/api/v1/query?query=up"

# Send load traffic to test dashboard reactivity
1..200 | ForEach-Object { Invoke-WebRequest -UseBasicParsing http://localhost:8082/ | Out-Null }

# Check exposed metrics endpoint
curl.exe http://localhost:9113/metrics
```[cite: 5, 6, 7, 8]

---

## Result Summary

* **Task 1:** Successfully automated Nginx web server installation and index page deployment with Ansible, verifying idempotency across consecutive runs[cite: 13].
* **Task 2:** Provisioned Docker containers dynamically via Terraform, scaling from 2 to 4 web replicas while auto-updating the upstream load balancer configuration[cite: 21].
* **Task 3:** Implemented a full ELK logging stack; Filebeat harvested container access logs, Logstash structured raw strings into Grok fields, Elasticsearch indexed the data, and Kibana rendered error queries and request breakdown visualizations[cite: 26].
* **Monitoring:** Deployed Prometheus and Node/Nginx Exporters to measure host and web traffic metrics, visualizing CPU, Memory, Disk, Network, and Request rate metrics on live Grafana dashboards[cite: 10].

```
