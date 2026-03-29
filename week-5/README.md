# Week 5 — Observability: Metrics, Logs, and Alerting

## What This Week Covers

Deploying an application is the beginning, not the end. Week 5 focuses on what comes after deployment: understanding whether the application is actually healthy, capturing a structured record of what it did, and getting notified automatically when something goes wrong.

By the end of this week you will have instrumented a production Flask application with Prometheus metrics, structured JSON logging shipped to CloudWatch Logs, CloudWatch Alarms wired to email via SNS, and a Grafana dashboard showing live traffic data. This is the observability baseline expected for any application running in production.

---

## Prerequisites

Complete Weeks 1-4 before starting this week. Specifically, you need:

- Week 1: Linux fundamentals, systemd services, bash scripting
- Week 2: Docker, Docker Compose, Python Flask app
- Week 3: Kubernetes, Jenkins CI/CD pipelines
- Week 4: AWS (EC2, VPC, ALB, ASG, IAM), Terraform — the capstone stack must be deployed or deployable

The Week 5 capstone exercise extends the Week 4 Terraform stack directly. You do not need to have it running continuously, but you need the Terraform code and the ability to run `terraform apply`.

You also need:
- An AWS account with IAM permissions for EC2, CloudWatch, SNS, and Logs
- A Slack workspace where you can create an incoming webhook (free plan is sufficient)

---

## What You Will Build

By the end of Week 5, the Week 4 production stack will have the following observability layer added to it:

- A Prometheus + Grafana + Alertmanager stack running on a separate EC2 instance (t3.micro)
- The Flask app exposing a `/metrics` endpoint scraped by Prometheus every 15 seconds
- Grafana dashboards showing request rate, error rate, and latency percentiles (p50, p95, p99)
- Structured JSON logs written to `/var/log/flaskapp.log` and shipped to CloudWatch Logs via the CloudWatch agent
- A Logs Insights query that surfaces 5xx errors within seconds of them occurring
- Two CloudWatch Alarms (ALB 5xx errors and ASG CPU) sending email via SNS when they fire
- A Prometheus alert rule that fires to Slack when the Flask error rate exceeds 5%
- A CloudWatch Dashboard showing ALB traffic, error count, healthy host count, and CPU

---

## Week Schedule

| Day | Topic | File |
|---|---|---|
| Day 21 | Observability introduction: metrics, logs, traces, and why they are separate | [day-21-observability-intro.md](day-21-observability-intro.md) |
| Day 22 | Prometheus and Grafana: scraping metrics, PromQL, alerting rules, Alertmanager | [day-22-prometheus-grafana.md](day-22-prometheus-grafana.md) |
| Day 23 | Application logging: log levels, structured JSON logs, Docker/Kubernetes log commands, CloudWatch Logs, Logs Insights | [day-23-logging-cloudwatch.md](day-23-logging-cloudwatch.md) |
| Day 24 | CloudWatch Alarms: alarm states, SNS notifications, composite alarms, dashboards, custom metrics, alerting best practices | [day-24-cloudwatch-alarms-alerting.md](day-24-cloudwatch-alarms-alerting.md) |
| Day 25 | Capstone project: add full observability to the Week 4 production stack | [exercises.md](exercises.md) |

---

## Key Tools Introduced This Week

| Tool | Purpose | Where it runs |
|---|---|---|
| `prometheus-flask-exporter` | Automatically instruments a Flask app with request metrics | In the Flask app |
| `python-json-logger` | Structured JSON log output for Python applications | In the Flask app |
| CloudWatch Agent | Ships log files from EC2 instances to CloudWatch Logs | On each EC2 instance |
| CloudWatch Logs Insights | Query language for analysing logs stored in CloudWatch | AWS console / CLI |
| CloudWatch Alarms | Threshold-based alerting on any CloudWatch metric | AWS managed service |
| SNS | Pub/sub notification delivery — email, SMS, webhook | AWS managed service |
| Alertmanager | Routes Prometheus alerts to Slack, email, or PagerDuty | Docker Compose on EC2 |

---

## Cost Awareness

The observability components introduced this week add real cost to your AWS bill. Understand what you are paying for before leaving things running.

| Service | Cost driver | Approximate cost |
|---|---|---|
| CloudWatch Logs ingestion | Per GB ingested | $0.50/GB |
| CloudWatch Logs storage | Per GB stored per month | $0.03/GB/month |
| CloudWatch custom metrics | Per metric per month (first 10 free) | $0.30/metric/month |
| CloudWatch Alarms | Per alarm per month (first 10 free) | $0.10/alarm/month |
| EC2 (observability instance) | t3.micro per hour | ~$8/month |
| SNS email notifications | Per 100,000 emails | Essentially free at lab scale |

Set log retention to 30 days (done in the exercises) to avoid indefinite storage costs. Tear everything down after completing the capstone.
