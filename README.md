# 1. Kiến trúc tổng quan 
```
Prometheus Server 1 ──┐
                       ├──> Alertmanager 1 ───┐
Prometheus Server 2 ──┘                       │
                                              ├──> Alertmanager Trung tâm
Alertmanager 2 ───────────────────────────────┘
```

# 2. Cấu hình Alertmanager tập trung 

* File: alertmanager-central.yaml

```yaml
global:
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_from: 'alertmanager@yourdomain.com'
  smtp_auth_username: 'your-email@gmail.com'
  smtp_auth_password: 'your-password'

route:
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 1h
  receiver: 'default-receiver'
  
  # Routes cho các cảnh báo từ các cluster khác nhau
  routes:
  - match:
      cluster: 'cluster-1'
    receiver: 'cluster-1-team'
  - match:
      cluster: 'cluster-2'
    receiver: 'cluster-2-team'

receivers:
- name: 'default-receiver'
  email_configs:
  - to: 'admin@yourdomain.com'
    send_resolved: true

- name: 'cluster-1-team'
  email_configs:
  - to: 'team1@yourdomain.com'
    send_resolved: true
  slack_configs:
  - api_url: 'https://hooks.slack.com/services/xxx'
    channel: '#alerts-cluster-1'

- name: 'cluster-2-team'
  email_configs:
  - to: 'team2@yourdomain.com'
    send_resolved: true
  webhook_configs:
  - url: 'http://webhook-server:8080/alerts'
    send_resolved: true
```

# 3. Cấu hình Alertmanager 1

* File: alertmanager-1.yaml

```yaml
global:
  resolve_timeout: 5m

route:
  receiver: 'default-receiver'
  group_by: ['alertname', 'cluster']
  routes:
  - match:
      severity: 'critical'
    receiver: 'critical-receiver'
  - match_re:
      service: ^(foo1|foo2|baz)$
    receiver: 'special-receiver'
    group_wait: 10s
    group_interval: 5m
    repeat_interval: 4h
    continue: true
  
  # Route đẩy tất cả cảnh báo về trung tâm
  - receiver: 'central-alertmanager'
    group_wait: 10s
    group_interval: 5s
    repeat_interval: 30s
    continue: true  # Quan trọng: tiếp tục xử lý các route sau

receivers:
- name: 'default-receiver'
  webhook_configs:
  - url: 'http://localhost:8080/webhook'
    send_resolved: true

- name: 'critical-receiver'
  webhook_configs:
  - url: 'http://critical-webhook:8080/'
    send_resolved: true

- name: 'special-receiver'
  webhook_configs:
  - url: 'http://special-webhook:8080/'
    send_resolved: true

- name: 'central-alertmanager'
  webhook_configs:
  - url: 'http://central-alertmanager:9093/api/v2/alerts'
    send_resolved: true
    # Thêm thông tin cluster
    http_config:
      basic_auth:
        username: 'alert-forwarder'
        password: 'your-password'
```

* File: alertmanager-2.yml

```yaml
global:
  resolve_timeout: 5m

route:
  receiver: 'default-receiver'
  group_by: ['alertname']
  
  # Route xử lý cục bộ
  routes:
  - match:
      severity: 'warning'
    receiver: 'local-warning-receiver'
    group_wait: 10s
    repeat_interval: 5m
  
  # Route đẩy về trung tâm
  - receiver: 'central-alertmanager'
    group_wait: 5s
    group_interval: 10s
    repeat_interval: 30s
    continue: true

receivers:
- name: 'default-receiver'
  webhook_configs:
  - url: 'http://local-webhook:8080/'
    send_resolved: true

- name: 'local-warning-receiver'
  email_configs:
  - to: 'local-admin@cluster2.com'
    send_resolved: true

- name: 'central-alertmanager'
  webhook_configs:
  - url: 'http://central-alertmanager:9093/api/v2/alerts'
    send_resolved: true
    http_config:
      basic_auth:
        username: 'cluster2-forwarder'
        password: 'cluster2-password'
      # Thêm headers để nhận diện cluster
      headers:
        X-Cluster-ID: 'cluster-2'
        X-Forwarded-By: 'alertmanager-2'
```

# 4. Cấu hình Prometheus cho từng cluster
```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting:
  alertmanagers:
  - static_configs:
    - targets: ['alertmanager-1:9093']

rule_files:
  - "alerts/cluster1/*.yml"

scrape_configs:
  # Cấu hình scrape targets
  - job_name: 'node'
    static_configs:
    - targets: ['node1:9100', 'node2:9100']
    labels:
      cluster: 'cluster-1'
```

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting:
  alertmanagers:
  - static_configs:
    - targets: ['alertmanager-2:9093']

rule_files:
  - "alerts/cluster2/*.yml"

scrape_configs:
  - job_name: 'node'
    static_configs:
    - targets: ['node3:9100', 'node4:9100']
    labels:
      cluster: 'cluster-2'
```

* FILE: docker-compose

```yaml
version: '3.8'

services:
  alertmanager-central:
    image: prom/alertmanager:latest
    container_name: alertmanager-central
    ports:
      - "9093:9093"
    volumes:
      - ./alertmanager-central.yml:/etc/alertmanager/alertmanager.yml
      - ./data/alertmanager:/alertmanager
    command:
      - '--config.file=/etc/alertmanager/alertmanager.yml'
      - '--storage.path=/alertmanager'
      - '--web.external-url=http://alertmanager-central:9093'
    networks:
      - monitoring

  # Thêm nginx làm reverse proxy nếu cần authentication
  nginx-proxy:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./ssl:/etc/nginx/ssl
    depends_on:
      - alertmanager-central
    networks:
      - monitoring

networks:
  monitoring:
    driver: bridge
```
