# Programming Languages for DevOps

You don't need to be a software engineer — you need to read code, write scripts, and debug when something breaks. This guide covers what you'll actually encounter day to day.

---

## The Short Answer — What to Learn and When

| Language | Priority | Used For |
|----------|----------|----------|
| Bash | Essential — learn now | Scripts, CI/CD steps, automation |
| Python | Essential — learn early | AWS automation, APIs, tooling |
| YAML | Essential — learn now | K8s manifests, CI/CD pipelines, Docker Compose |
| HCL | Week 4 | Terraform infrastructure |
| Go | Good to know | Reading Terraform/Docker/K8s source, writing CLIs |
| JSON | Essential — learn now | API responses, AWS configs |
| JavaScript | Optional | AWS Lambda, CDK |

---

## Bash

Why: already on every Linux server, no install needed, used in every CI/CD pipeline and deployment script.

**Variables and strings:**
```bash
NAME="devops"
echo "Hello $NAME"
echo "Today is $(date +%Y-%m-%d)"   # command substitution
```

**Conditionals:**
```bash
if [ "$DISK_USAGE" -gt 80 ]; then
  echo "Disk usage is high"
elif [ "$DISK_USAGE" -gt 60 ]; then
  echo "Disk usage is moderate"
else
  echo "Disk usage is fine"
fi
```

**Loops:**
```bash
for file in *.log; do
  echo "Processing $file"
done

while IFS= read -r line; do
  echo "$line"
done < servers.txt
```

**Functions:**
```bash
function check_service() {
  local service=$1
  if systemctl is-active --quiet "$service"; then
    echo "$service is running"
  else
    echo "$service is NOT running"
  fi
}

check_service nginx
check_service docker
```

**Exit codes and error handling:**
```bash
set -euo pipefail   # exit on error, undefined vars, failed pipes — use this always

command && echo "success" || echo "failed"
if ! curl -sf http://localhost:8080/health; then
  echo "Health check failed"
  exit 1
fi
```

**A real DevOps script — deployment health check:**
```bash
#!/bin/bash
set -euo pipefail

SERVICE=$1
MAX_RETRIES=10
COUNT=0

echo "Waiting for $SERVICE to be healthy..."
until curl -sf "http://localhost:8080/health" > /dev/null; do
  COUNT=$((COUNT + 1))
  if [ "$COUNT" -ge "$MAX_RETRIES" ]; then
    echo "Service failed to become healthy after $MAX_RETRIES attempts"
    exit 1
  fi
  echo "Attempt $COUNT/$MAX_RETRIES — retrying in 5s..."
  sleep 5
done

echo "$SERVICE is healthy"
```

Resources: https://www.shellcheck.net (paste your script, it catches bugs)

---

## Python

Why: most common language for DevOps automation. AWS SDK (boto3), REST APIs, YAML/JSON processing, writing custom tools.

**Setup:**
```bash
sudo apt install python3 python3-pip python3-venv
python3 -m venv .venv
source .venv/bin/activate
pip install boto3 requests pyyaml
```

**Variables, lists, dicts:**
```python
name = "devops"
servers = ["web-01", "web-02", "web-03"]
config = {"region": "us-east-1", "instance_type": "t3.micro"}

print(config["region"])
for server in servers:
    print(server)
```

**Functions:**
```python
def check_health(url: str) -> bool:
    import requests
    try:
        r = requests.get(url, timeout=5)
        return r.status_code == 200
    except requests.RequestException:
        return False
```

**Reading and writing files:**
```python
# Read
with open("config.txt") as f:
    content = f.read()

# Write
with open("output.txt", "w") as f:
    f.write("deployment complete\n")
```

**Working with JSON and YAML:**
```python
import json
import yaml

# JSON
data = json.loads('{"status": "ok"}')
print(data["status"])

# YAML
with open("values.yaml") as f:
    config = yaml.safe_load(f)
print(config["replicaCount"])
```

**AWS automation with boto3:**
```python
import boto3

ec2 = boto3.client("ec2", region_name="us-east-1")

# List running instances
response = ec2.describe_instances(
    Filters=[{"Name": "instance-state-name", "Values": ["running"]}]
)
for reservation in response["Reservations"]:
    for instance in reservation["Instances"]:
        name = next(
            (t["Value"] for t in instance.get("Tags", []) if t["Key"] == "Name"),
            "unnamed",
        )
        print(f"{instance['InstanceId']}  {instance['InstanceType']}  {name}")
```

**Environment variables (never hardcode credentials):**
```python
import os

db_password = os.environ["DB_PASSWORD"]   # raises KeyError if not set — intentional
region = os.getenv("AWS_REGION", "us-east-1")  # with default
```

---

## Go

Why: Terraform, Docker, Kubernetes, and most modern DevOps tools are written in Go. You need to read Go code and understand its patterns. Writing Go is optional but useful for CLI tools.

Install: `sudo apt install golang-go` or download from https://go.dev

**Types and variables:**
```go
name := "devops"          // type inferred
var port int = 8080       // explicit type
```

**Error handling (the most important Go pattern to understand):**
```go
result, err := someFunction()
if err != nil {
    fmt.Fprintf(os.Stderr, "error: %v\n", err)
    os.Exit(1)
}
```

Go has no exceptions. Every function that can fail returns an error as the last return value. Always check it.

**A simple CLI tool in Go:**
```go
package main

import (
    "fmt"
    "net/http"
    "os"
)

func main() {
    url := os.Args[1]
    resp, err := http.Get(url)
    if err != nil {
        fmt.Fprintf(os.Stderr, "request failed: %v\n", err)
        os.Exit(1)
    }
    defer resp.Body.Close()
    fmt.Printf("Status: %d\n", resp.StatusCode)
}
```

**Build and run:**
```bash
go run main.go https://example.com     # run directly
go build -o checker main.go            # compile to single binary
./checker https://example.com          # run binary (no Go runtime needed)
```

The single binary output is the main reason DevOps tools are written in Go — you can copy one file to any Linux server and run it.

---

## YAML

Why: you will write YAML every day — Kubernetes manifests, Docker Compose files, CI/CD pipelines, Helm values. YAML errors are extremely common and can be subtle.

**Key rules:**
```yaml
# Indentation: 2 spaces, never tabs
# Strings: quotes optional unless special characters
# Lists use -
# Dicts use key: value

name: flaskapp
replicas: 3
enabled: true     # boolean, not string

env:
  - name: DB_HOST
    value: "postgres"     # quoted because of special chars? no, just for clarity
  - name: PORT
    value: "8080"         # NOTE: always quote numbers used as strings

labels:
  app: flaskapp
  version: "1.0.0"

# Multi-line strings
command: |
  echo "starting"
  python app.py

# Inline list
ports: [80, 443, 8080]

# Inline dict
resources: {cpu: "100m", memory: "128Mi"}
```

**Common YAML mistakes:**
```yaml
# WRONG: tab indentation
name:	flaskapp   # tab character — YAML parsers reject this

# WRONG: missing space after colon
name:flaskapp      # not parsed as key-value

# WRONG: inconsistent indentation
spec:
  containers:
   - name: app    # 3 spaces here, 2 elsewhere — will break
```

**Validate YAML before applying:**
```bash
python3 -c "import yaml, sys; yaml.safe_load(sys.stdin)" < values.yaml
```

---

## JSON

Why: AWS CLI output, API responses, Terraform outputs, ECR lifecycle policies — all JSON.

**Read it:**
```json
{
  "InstanceId": "i-0abc123",
  "State": {"Name": "running"},
  "Tags": [{"Key": "Name", "Value": "web-01"}]
}
```

**Use `jq` to parse JSON in Bash** (install: `sudo apt install jq`):
```bash
# Get a field
aws ec2 describe-instances | jq '.Reservations[0].Instances[0].InstanceId'

# Filter running instances
aws ec2 describe-instances | jq '.Reservations[].Instances[] | select(.State.Name=="running") | .InstanceId'

# Get tag value
aws ec2 describe-instances | jq '.Reservations[].Instances[].Tags[] | select(.Key=="Name") | .Value'
```

---

## Quick Reference — When to Use What

| Task | Use |
|------|-----|
| Automate a deployment step | Bash |
| Call the AWS API programmatically | Python + boto3 |
| Process a YAML config file | Python + pyyaml |
| Write a Kubernetes manifest | YAML |
| Write Terraform infrastructure | HCL |
| Parse AWS CLI output in a script | Bash + jq |
| Write a portable CLI tool | Go |
| Write an AWS Lambda function | Python or JavaScript |

---

## Where to Go Next

- **Bash**: https://www.shellcheck.net, https://www.bash.academy
- **Python**: https://docs.python.org/3/tutorial, https://boto3.amazonaws.com/v1/documentation/api/latest/index.html
- **Go**: https://go.dev/tour (interactive, takes ~4 hours)
- **jq**: https://jqlang.github.io/jq/manual
