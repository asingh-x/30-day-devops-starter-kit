# Week 5 Project: Add Observability to the Production Stack

## Overview

In Week 4 you built a production AWS environment: VPC, ALB, ASG, EC2 instances running a Python Flask app managed by Terraform. The infrastructure is solid. But you cannot operate it yet — you have no visibility into whether it is healthy, no structured log trail when something breaks, and no automated notifications when things go wrong.

This project adds full observability to that stack. When you finish, you will be able to answer the following questions about your running application at any time:

- What is the current request rate, error rate, and latency?
- Are there errors in the application logs in the last hour?
- Has the ALB returned any 5xx errors?
- Is the ASG CPU elevated?
- Who gets notified when any of these conditions are true, and how?

The project is divided into four parts. Each part builds on the previous one. Work through them in order.

---

## Architecture

```mermaid
flowchart TD
    Internet --> ALB[Application Load Balancer]
    ALB --> ASG[Auto Scaling Group\nEC2 instances\nFlask app on port 8080]

    subgraph EC2Instance[EC2 Instance - per instance]
        APP[Flask App\nGunicorn]
        CWAgent[CloudWatch Agent]
        APP -->|JSON logs| LogFile[/var/log/flaskapp.log]
        CWAgent -->|ships logs| LogFile
        APP -->|/metrics endpoint| PromScrape[Prometheus scrape\nport 8080]
    end

    subgraph ObsStack[Observability EC2 - t3.micro]
        Prom[Prometheus]
        Graf[Grafana]
        AM[Alertmanager]
        Prom --> Graf
        Prom --> AM
    end

    PromScrape -->|scrape| Prom
    CWAgent -->|PutLogEvents| CWLogs[CloudWatch Logs\n/production/flaskapp]
    ALB -->|built-in metrics| CWMetrics[CloudWatch Metrics\nAWS/ApplicationELB]
    ASG -->|built-in metrics| CWMetrics
    CWMetrics --> CWAlarms[CloudWatch Alarms]
    CWAlarms -->|SNS| Email[Email notification]
    AM -->|webhook| Slack[Slack channel]
    CWLogs --> Insights[Logs Insights queries]
```

---

## Prerequisites

- Week 4 capstone complete: Terraform stack deployed with VPC, ALB, ASG, EC2
- AWS CLI configured with credentials that have IAM, CloudWatch, SNS, and EC2 permissions
- A second EC2 instance (t3.micro) launched in the same VPC for the Prometheus/Grafana stack
- Docker and Docker Compose installed on the observability EC2 instance
- A Slack workspace where you can create a webhook URL (free plan is fine)

---

## Part 1: Application Metrics with Prometheus

### 1.1 Add prometheus-flask-exporter to the Flask app

Update the Flask app (`/opt/devops-app/app.py` on each EC2 instance) to expose a `/metrics` endpoint:

```bash
pip3 install prometheus-flask-exporter
```

Minimal change to `app.py`:

```python
from flask import Flask, jsonify
from prometheus_flask_exporter import PrometheusMetrics
import socket

app = Flask(__name__)
metrics = PrometheusMetrics(app)

# Static metric: labels that never change for this instance
metrics.info("app_info", "Flask app metadata", version="1.0", hostname=socket.gethostname())


@app.route("/")
def index():
    return f"Hello from {socket.gethostname()}\n"


@app.route("/health")
def health():
    return jsonify({"status": "ok", "hostname": socket.gethostname()}), 200
```

`PrometheusMetrics(app)` automatically:
- Registers a `/metrics` endpoint
- Tracks request count, request latency, and in-flight requests per endpoint and HTTP method
- Labels every metric with `endpoint` and `method`

Restart the app:

```bash
sudo systemctl restart devops-app
```

Verify the endpoint works:

```bash
curl http://localhost:8080/metrics
```

You should see output like:

```
# HELP flask_http_request_duration_seconds Flask HTTP request duration in seconds
# TYPE flask_http_request_duration_seconds histogram
flask_http_request_duration_seconds_bucket{endpoint="/",le="0.005",method="GET",status="200"} 3.0
...
```

### 1.2 Open the security group for Prometheus scraping

The Prometheus server (on the observability EC2 instance) needs to reach port 8080 on the app EC2 instances. The Week 4 security group only allows port 8080 from the ALB security group.

Add a Terraform rule in `security_groups.tf` to allow port 8080 from the observability instance's private IP (or from a dedicated security group if you prefer):

```hcl
# In the ec2 security group resource, add this ingress rule
ingress {
  description = "Allow Prometheus scraping from observability server"
  from_port   = 8080
  to_port     = 8080
  protocol    = "tcp"
  cidr_blocks = ["<observability-instance-private-ip>/32"]
}
```

Apply the change:

```bash
cd week-4/terraform
terraform plan
terraform apply
```

### 1.3 Run Prometheus and Grafana on the observability instance

SSH into the observability EC2 instance. Create a working directory:

```bash
mkdir -p ~/monitoring
cd ~/monitoring
```

Create `prometheus.yml`:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "alert-rules.yml"

alerting:
  alertmanagers:
    - static_configs:
        - targets: ["alertmanager:9093"]

scrape_configs:
  - job_name: "flask-app"
    static_configs:
      # Add all private IPs of your ASG instances here
      # In production you would use EC2 service discovery
      - targets:
          - "<app-instance-1-private-ip>:8080"
          - "<app-instance-2-private-ip>:8080"
    metrics_path: "/metrics"
```

Create `alert-rules.yml`:

```yaml
groups:
  - name: flask-app
    rules:
      - alert: FlaskHighErrorRate
        expr: |
          sum(rate(flask_http_request_total{status=~"5.."}[2m]))
          /
          sum(rate(flask_http_request_total[2m]))
          > 0.05
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "Flask app error rate above 5%"
          description: "{{ $value | humanizePercentage }} of requests are returning 5xx errors."
```

Create `alertmanager.yml`:

```yaml
global:
  resolve_timeout: 5m

route:
  group_by: ["alertname"]
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  receiver: "slack"

receivers:
  - name: "slack"
    slack_configs:
      - api_url: "${SLACK_WEBHOOK_URL}"
        channel: "#alerts"
        send_resolved: true
        title: '{{ if eq .Status "firing" }}FIRING{{ else }}RESOLVED{{ end }}: {{ .CommonLabels.alertname }}'
        text: "{{ range .Alerts }}{{ .Annotations.description }}\n{{ end }}"
```

Create `.env`:

```bash
# .env — never commit this file
GRAFANA_ADMIN_PASSWORD=<choose-a-strong-password>
SLACK_WEBHOOK_URL=https://hooks.slack.com/services/YOUR/WEBHOOK/URL
```

```bash
echo ".env" >> .gitignore
```

Create `docker-compose.yml`:

```yaml
services:
  prometheus:
    image: prom/prometheus:v2.51.0
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - ./alert-rules.yml:/etc/prometheus/alert-rules.yml
      - prometheus-data:/prometheus

  alertmanager:
    image: prom/alertmanager:v0.27.0
    ports:
      - "9093:9093"
    volumes:
      - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml
    command:
      - "--config.file=/etc/alertmanager/alertmanager.yml"
      - "--config.expand-env=true"
    environment:
      - SLACK_WEBHOOK_URL=${SLACK_WEBHOOK_URL}

  grafana:
    image: grafana/grafana:10.4.0
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_ADMIN_PASSWORD}
    volumes:
      - grafana-data:/var/lib/grafana

volumes:
  prometheus-data:
  grafana-data:
```

Start the stack:

```bash
docker compose up -d
```

Verify Prometheus can reach the Flask app targets:

```bash
# Open in your browser (with SSH tunnel if needed)
# http://<observability-instance-public-ip>:9090/targets
```

Both Flask app instances should show as State: UP.

### 1.4 Create Grafana panels for the Flask app

Add Prometheus as a Grafana data source at `http://prometheus:9090`, then build a new dashboard with these three panels:

**Panel 1: Request rate**

```promql
sum(rate(flask_http_request_total[5m])) by (instance)
```

**Panel 2: Error rate (5xx)**

```promql
sum(rate(flask_http_request_total{status=~"5.."}[5m])) by (instance)
```

**Panel 3: Request latency p50, p95, p99**

```promql
histogram_quantile(0.50, sum(rate(flask_http_request_duration_seconds_bucket[5m])) by (le, instance))
histogram_quantile(0.95, sum(rate(flask_http_request_duration_seconds_bucket[5m])) by (le, instance))
histogram_quantile(0.99, sum(rate(flask_http_request_duration_seconds_bucket[5m])) by (le, instance))
```

Add all three as separate queries on a single Time Series panel. Label them in Grafana using the Legend field: `p50`, `p95`, `p99`.

---

## Part 2: Structured Logging to CloudWatch

### 2.1 Update the Flask app to write JSON logs to a file

Install the logging library on each EC2 instance:

```bash
pip3 install python-json-logger
```

Update `app.py` to write JSON logs to both stdout (for systemd journal) and a file (for the CloudWatch agent):

```python
import logging
import sys
from flask import Flask, jsonify, request
from prometheus_flask_exporter import PrometheusMetrics
from pythonjsonlogger import jsonlogger
import socket

app = Flask(__name__)
metrics = PrometheusMetrics(app)
metrics.info("app_info", "Flask app metadata", version="1.0", hostname=socket.gethostname())

# Configure JSON logger writing to both stdout and a file
logger = logging.getLogger("flaskapp")
logger.setLevel(logging.INFO)

formatter = jsonlogger.JsonFormatter(
    fmt="%(asctime)s %(levelname)s %(name)s %(message)s",
    datefmt="%Y-%m-%dT%H:%M:%SZ"
)

# stdout handler (goes to systemd journal)
stdout_handler = logging.StreamHandler(sys.stdout)
stdout_handler.setFormatter(formatter)

# file handler (picked up by CloudWatch agent)
file_handler = logging.FileHandler("/var/log/flaskapp.log")
file_handler.setFormatter(formatter)

logger.addHandler(stdout_handler)
logger.addHandler(file_handler)


@app.route("/")
def index():
    logger.info("request received", extra={"endpoint": "/", "method": request.method})
    return f"Hello from {socket.gethostname()}\n"


@app.route("/health")
def health():
    return jsonify({"status": "ok", "hostname": socket.gethostname()}), 200


@app.route("/error")
def simulate_error():
    logger.error(
        "simulated application error",
        extra={"endpoint": "/error", "error_code": "SIM_001"}
    )
    return jsonify({"error": "something went wrong"}), 500
```

Create the log file and set correct ownership before restarting:

```bash
sudo touch /var/log/flaskapp.log
sudo chown ec2-user:ec2-user /var/log/flaskapp.log
sudo systemctl restart devops-app
```

### 2.2 Configure the CloudWatch agent

Install the CloudWatch agent on each EC2 instance (if not already installed from Day 23):

```bash
wget https://s3.amazonaws.com/amazoncloudwatch-agent/ubuntu/amd64/latest/amazon-cloudwatch-agent.deb
sudo dpkg -i amazon-cloudwatch-agent.deb
```

Create the agent configuration at `/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json`:

```json
{
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/var/log/flaskapp.log",
            "log_group_name": "/production/flaskapp",
            "log_stream_name": "{instance_id}",
            "retention_in_days": 30,
            "timestamp_format": "%Y-%m-%dT%H:%M:%SZ",
            "multi_line_start_pattern": "^\\{"
          }
        ]
      }
    }
  }
}
```

The EC2 IAM role needs `CloudWatchAgentServerPolicy` attached. Add this to `asg.tf` in your Terraform configuration:

```hcl
resource "aws_iam_role_policy_attachment" "ec2_cloudwatch" {
  role       = aws_iam_role.ec2.name
  policy_arn = "arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy"
}
```

Apply the Terraform change, then start the agent on each instance:

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config \
  -m ec2 \
  -s \
  -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json
```

Verify the agent is running:

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a status
```

Generate some log entries, then verify they appear in CloudWatch:

```bash
# Generate traffic including errors
for i in {1..10}; do curl http://localhost:8080/; done
for i in {1..3}; do curl http://localhost:8080/error; done

# Check logs arrived in CloudWatch (may take 1-2 minutes)
aws logs describe-log-streams \
  --log-group-name "/production/flaskapp" \
  --region us-east-1
```

### 2.3 Query 5xx errors in Logs Insights

In the CloudWatch console, open Logs Insights, select the `/production/flaskapp` log group, set the time range to the last 1 hour, and run:

```
fields @timestamp, level, message, endpoint
| filter level = "ERROR"
| sort @timestamp desc
| limit 50
```

You should see the error entries you generated. If you triggered the `/error` endpoint, the `endpoint` field will show `/error` in each result row.

To count errors by endpoint:

```
filter level = "ERROR"
| stats count(*) as error_count by endpoint
| sort error_count desc
```

---

## Part 3: Alerting

### 3.1 CloudWatch Alarm: ALB 5xx errors

Create an SNS topic for production alerts:

```bash
TOPIC_ARN=$(aws sns create-topic \
  --name "production-alerts" \
  --region us-east-1 \
  --query 'TopicArn' \
  --output text)

aws sns subscribe \
  --topic-arn $TOPIC_ARN \
  --protocol email \
  --notification-endpoint your@email.com \
  --region us-east-1

echo "Confirm the subscription in your email before continuing."
```

Get the ALB ARN from Terraform output:

```bash
cd week-4/terraform
ALB_ARN=$(terraform output -raw alb_arn)

# CloudWatch uses a shortened version of the ARN as the dimension value
# Extract everything after "loadbalancer/"
ALB_DIMENSION=$(aws elbv2 describe-load-balancers \
  --load-balancer-arns $ALB_ARN \
  --query 'LoadBalancers[0].LoadBalancerArn' \
  --output text | sed 's|.*:loadbalancer/||')
```

Create the alarm:

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "alb-5xx-errors-production" \
  --alarm-description "P1: ALB returning 5xx errors > 10 in 5 minutes. Check application logs: /production/flaskapp. Runbook: https://wiki.example.com/runbooks/alb-5xx" \
  --namespace "AWS/ApplicationELB" \
  --metric-name "HTTPCode_ELB_5XX_Count" \
  --dimensions Name=LoadBalancer,Value=$ALB_DIMENSION \
  --statistic Sum \
  --period 300 \
  --evaluation-periods 1 \
  --threshold 10 \
  --comparison-operator GreaterThanThreshold \
  --treat-missing-data notBreaching \
  --alarm-actions $TOPIC_ARN \
  --ok-actions $TOPIC_ARN \
  --region us-east-1
```

Note: `--treat-missing-data notBreaching` is correct here because no data means no 5xx errors — the ALB simply had no traffic. Contrast this with the CPU alarm where missing data means the instance may have died.

### 3.2 CloudWatch Alarm: ASG CPU utilisation

Get the ASG name from Terraform output:

```bash
ASG_NAME=$(terraform output -raw asg_name)
```

Create the alarm:

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "asg-high-cpu-production" \
  --alarm-description "P2: ASG average CPU above 75% for 10 minutes. Consider scaling up manually or checking for runaway processes. Runbook: https://wiki.example.com/runbooks/high-cpu" \
  --namespace "AWS/EC2" \
  --metric-name "CPUUtilization" \
  --dimensions Name=AutoScalingGroupName,Value=$ASG_NAME \
  --statistic Average \
  --period 300 \
  --evaluation-periods 2 \
  --threshold 75 \
  --comparison-operator GreaterThanThreshold \
  --treat-missing-data breaching \
  --alarm-actions $TOPIC_ARN \
  --ok-actions $TOPIC_ARN \
  --region us-east-1
```

Using `AutoScalingGroupName` as the dimension means the alarm averages CPU across all instances in the ASG. This is more useful than alarming on a single instance — one instance running hot because it is about to be replaced is not worth waking someone up.

### 3.3 Prometheus Alert Rule: Flask error rate to Slack

The alert rule in `alert-rules.yml` on your observability instance was already configured in Part 1:

```yaml
- alert: FlaskHighErrorRate
  expr: |
    sum(rate(flask_http_request_total{status=~"5.."}[2m]))
    /
    sum(rate(flask_http_request_total[2m]))
    > 0.05
  for: 2m
  labels:
    severity: warning
  annotations:
    summary: "Flask app error rate above 5%"
    description: "{{ $value | humanizePercentage }} of requests are returning 5xx errors."
```

The Alertmanager configuration routes this to Slack via webhook. The webhook URL comes from the `SLACK_WEBHOOK_URL` environment variable set in `.env` — it is never stored in the config file itself.

To create a Slack webhook:
1. Go to `https://api.slack.com/apps`
2. Create a new app, choose "From scratch"
3. Under "Incoming Webhooks", activate and add a webhook to your chosen channel
4. Copy the webhook URL (starts with `https://hooks.slack.com/services/...`)
5. Add it to `~/monitoring/.env` as `SLACK_WEBHOOK_URL=<url>`

Restart Alertmanager to pick up the new env var:

```bash
cd ~/monitoring
docker compose restart alertmanager
```

Test the alert by temporarily lowering the threshold to 0.001 (0.1%), generating a single request, and watching it fire within 2 minutes. Restore the threshold to 0.05 when done.

---

## Part 4: CloudWatch Dashboard

Create a CloudWatch Dashboard named `flaskapp-production` that gives an at-a-glance view of the production stack.

You will need:
- The ALB dimension value (`$ALB_DIMENSION` from Part 3.1)
- The target group ARN dimension: `aws elbv2 describe-target-groups --query 'TargetGroups[0].TargetGroupArn'`
- The ASG name (`$ASG_NAME` from Part 3.2)

Get the target group dimension:

```bash
TG_ARN=$(terraform output -raw target_group_arn)
TG_DIMENSION=$(echo $TG_ARN | sed 's|.*:targetgroup/|targetgroup/|')
```

Create the dashboard:

```bash
aws cloudwatch put-dashboard \
  --dashboard-name "flaskapp-production" \
  --dashboard-body '{
    "widgets": [
      {
        "type": "metric",
        "x": 0, "y": 0, "width": 12, "height": 6,
        "properties": {
          "title": "ALB Request Count",
          "view": "timeSeries",
          "metrics": [[
            "AWS/ApplicationELB", "RequestCount",
            "LoadBalancer", "'"$ALB_DIMENSION"'",
            {"stat": "Sum", "period": 60, "label": "Requests/min"}
          ]],
          "period": 60
        }
      },
      {
        "type": "metric",
        "x": 12, "y": 0, "width": 12, "height": 6,
        "properties": {
          "title": "ALB 5xx Error Count",
          "view": "timeSeries",
          "metrics": [[
            "AWS/ApplicationELB", "HTTPCode_ELB_5XX_Count",
            "LoadBalancer", "'"$ALB_DIMENSION"'",
            {"stat": "Sum", "period": 60, "label": "5xx errors/min", "color": "#d62728"}
          ]],
          "period": 60
        }
      },
      {
        "type": "metric",
        "x": 0, "y": 6, "width": 12, "height": 6,
        "properties": {
          "title": "Target Group Healthy Host Count",
          "view": "timeSeries",
          "metrics": [[
            "AWS/ApplicationELB", "HealthyHostCount",
            "LoadBalancer", "'"$ALB_DIMENSION"'",
            "TargetGroup", "'"$TG_DIMENSION"'",
            {"stat": "Average", "period": 60, "label": "Healthy hosts"}
          ]],
          "period": 60,
          "yAxis": {"left": {"min": 0}}
        }
      },
      {
        "type": "metric",
        "x": 12, "y": 6, "width": 12, "height": 6,
        "properties": {
          "title": "ASG EC2 CPU Utilization",
          "view": "timeSeries",
          "metrics": [[
            "AWS/EC2", "CPUUtilization",
            "AutoScalingGroupName", "'"$ASG_NAME"'",
            {"stat": "Average", "period": 300, "label": "CPU %"}
          ]],
          "period": 300,
          "yAxis": {"left": {"min": 0, "max": 100}}
        }
      }
    ]
  }' \
  --region us-east-1
```

Open the dashboard in the console:

```
https://console.aws.amazon.com/cloudwatch/home#dashboards:name=flaskapp-production
```

Generate some traffic to populate the panels:

```bash
# From your local machine
ALB_DNS=$(cd week-4/terraform && terraform output -raw alb_dns_name)

for i in {1..50}; do
  curl -s http://$ALB_DNS/ > /dev/null
  sleep 1
done
```

All four panels should show data within 2-3 minutes.

---

## Completion Checklist

Work through this checklist after completing all four parts.

### Part 1: Prometheus Metrics

- [ ] Flask app `/metrics` endpoint returns Prometheus-format output: `curl http://localhost:8080/metrics`
- [ ] Prometheus UI at port 9090 shows both app instances as State: UP under `/targets`
- [ ] Grafana shows live request rate data — verify by generating traffic and watching the panel update
- [ ] Grafana error rate panel shows non-zero values after hitting the `/error` endpoint
- [ ] Grafana latency panel shows p50/p95/p99 as separate lines

### Part 2: Structured Logging

- [ ] `cat /var/log/flaskapp.log` on an app instance shows valid JSON lines
- [ ] CloudWatch log group `/production/flaskapp` exists: `aws logs describe-log-groups --log-group-name-prefix /production/flaskapp`
- [ ] Log streams exist (one per instance): `aws logs describe-log-streams --log-group-name /production/flaskapp`
- [ ] Log retention is set to 30 days: check in the console under Log groups
- [ ] Logs Insights query for ERROR returns results after hitting the `/error` endpoint

### Part 3: Alerting

- [ ] SNS topic `production-alerts` exists and email subscription is confirmed
- [ ] CloudWatch alarm `alb-5xx-errors-production` exists in OK or INSUFFICIENT_DATA state
- [ ] CloudWatch alarm `asg-high-cpu-production` exists in OK or INSUFFICIENT_DATA state
- [ ] Test SNS delivery: `aws sns publish --topic-arn $TOPIC_ARN --message "test" --subject "Test"` — verify email received
- [ ] Prometheus alert rule `FlaskHighErrorRate` appears in Prometheus UI at `/alerts` (state: Inactive is correct when no errors are occurring)
- [ ] Alertmanager UI at port 9093 is accessible and shows no active silences

### Part 4: Dashboard

- [ ] CloudWatch Dashboard `flaskapp-production` exists: `aws cloudwatch list-dashboards`
- [ ] All four dashboard panels display data (not "No data")
- [ ] Healthy host count shows 2 (both ASG instances passing health checks)
- [ ] Generating traffic with `curl` is visible in the ALB Request Count panel within 2 minutes

### End-to-end test

Run this sequence and verify the full observability chain responds:

```bash
# 1. Generate errors
for i in {1..20}; do curl -s http://$ALB_DNS/error > /dev/null; done

# 2. Check Logs Insights: errors should appear within 2 minutes
# Run in CloudWatch Logs Insights against /production/flaskapp:
# fields @timestamp, level, message | filter level = "ERROR" | sort @timestamp desc

# 3. Check Prometheus: error rate panel in Grafana should spike
# http://<observability-ip>:3000

# 4. If error rate exceeds 5% sustained for 2 minutes, Alertmanager fires to Slack
# The threshold is 5% — 20 errors across all requests may or may not exceed it
# depending on total traffic. Reduce threshold temporarily to test if needed.

# 5. Check CloudWatch Dashboard: ALB 5xx panel should show the errors
# https://console.aws.amazon.com/cloudwatch/home#dashboards:name=flaskapp-production
```

---

## Teardown

After completing the project, tear down the infrastructure to avoid ongoing costs.

```bash
# 1. Stop the observability stack
ssh <observability-instance>
cd ~/monitoring
docker compose down -v

# 2. Terminate the observability EC2 instance manually in the console
# (or via CLI if you launched it with Terraform)

# 3. Delete CloudWatch resources
aws cloudwatch delete-alarms \
  --alarm-names "alb-5xx-errors-production" "asg-high-cpu-production" \
  --region us-east-1

aws cloudwatch delete-dashboards \
  --dashboard-names "flaskapp-production" \
  --region us-east-1

aws logs delete-log-group \
  --log-group-name "/production/flaskapp" \
  --region us-east-1

# 4. Delete the SNS topic
aws sns delete-topic --topic-arn $TOPIC_ARN --region us-east-1

# 5. Destroy the Week 4 Terraform stack
cd week-4/terraform
terraform destroy -auto-approve
```

Verify nothing billable remains:

```bash
aws ec2 describe-instances \
  --filters "Name=instance-state-name,Values=running" \
  --query 'Reservations[*].Instances[*].[InstanceId,Tags[?Key==`Name`].Value|[0]]' \
  --output table

aws elbv2 describe-load-balancers --query 'LoadBalancers[*].LoadBalancerName'

aws cloudwatch list-dashboards --query 'DashboardEntries[*].DashboardName'
```
