# Week 1 Mini Project — System Health Check Script

**Target OS: Ubuntu 22.04 LTS or 24.04 LTS**

## Objective

Write a Bash script that checks the health of a Linux server and outputs a report. Push the project to GitHub following the proper PR workflow.

---

## Requirements

Your script (`healthcheck.sh`) must check and report:

1. **Disk usage** — warn if any partition is above 80% full
2. **Memory usage** — show used vs total RAM
3. **CPU load** — show the 1-minute load average
4. **Running services** — check if nginx and sshd are running (if nginx is not installed, report "NOT INSTALLED" and skip)
5. **Network** — check if the server can reach the internet (ping 8.8.8.8)

### Expected Output Format

```
=== Server Health Check: 2024-03-01 10:00:00 ===

[DISK]
  /       : 45% used — OK
  /var    : 82% used — WARNING

[MEMORY]
  Used: 1.2G / Total: 4.0G (30%)

[CPU]
  Load average (1m): 0.42 — OK

[SERVICES]
  nginx  : RUNNING
  sshd   : RUNNING

[NETWORK]
  Internet connectivity: OK

=== End of Report ===
```

---

## Bonus Challenges

- [ ] Send output to a log file with a timestamp in the filename
- [ ] Add color output (red for warnings, green for OK)
- [ ] Add a `--help` flag that explains usage
- [ ] Schedule the script to run every hour using `cron`

---

## Cron Syntax Reference

```
* * * * * command
│ │ │ │ │
│ │ │ │ └── Day of week (0-7, 0=Sunday)
│ │ │ └──── Month (1-12)
│ │ └────── Day of month (1-31)
│ └──────── Hour (0-23)
└────────── Minute (0-59)
```

Examples:
```bash
# Run every hour at minute 0
# Use ~/health.log — /var/log/ requires root write access
0 * * * * /home/$(whoami)/healthcheck.sh >> ~/health.log 2>&1

# Run every day at 6 AM
0 6 * * * /home/ankit/healthcheck.sh

# Run every 15 minutes
*/15 * * * * /home/ankit/healthcheck.sh
```

Edit your cron jobs with: `crontab -e`

---

## GitHub Workflow

1. Create a new public repo on GitHub: `devops-healthcheck`
2. Clone it locally
3. Work on a branch: `feature/healthcheck-script`
4. Commit with a proper message: `feat: add server health check script`
5. Push and open a Pull Request with a proper description
6. Merge using Squash and Merge
7. Tag the release: `v1.0.0`

---

## Hints

```bash
# Disk usage percentage
df -h | awk 'NR>1 {print $5, $6}'

# Memory
free -h

# Load average
cat /proc/loadavg | awk '{print $1}'

# Check if a service is running
systemctl is-active nginx

# Ping test
ping -c 1 8.8.8.8 > /dev/null 2>&1 && echo "OK" || echo "FAIL"

# Current date/time
date "+%Y-%m-%d %H:%M:%S"
```

---

## Checklist Before Submitting

- [ ] Script runs without errors on a fresh Ubuntu server
- [ ] Correct file permissions (`chmod 755 healthcheck.sh`)
- [ ] `.gitignore` excludes any log files
- [ ] README explains what the script does and how to run it
- [ ] Committed with a meaningful commit message
- [ ] PR has a proper description
