# Day 3 — Processes, Networking, and SSH

---

## Processes

A **process** is any running program. Every process has a unique **PID** (Process ID).

### Viewing Processes

```bash
ps aux                  # All running processes
ps aux | grep nginx     # Find a specific process
top                     # Live process monitor (q to quit)
htop                    # Better version of top (install: apt install htop)
```

`ps aux` columns:
```
USER   PID  %CPU  %MEM  VSZ  RSS  TTY  STAT  START  TIME  COMMAND
root   1234  0.0   0.1  123  456   ?   Ss    10:00  0:00  nginx
```

### Managing Processes

```bash
kill 1234              # Send SIGTERM (graceful stop)
kill -9 1234           # Send SIGKILL (force stop, last resort)
pkill nginx            # Kill by process name
killall nginx          # Kill all processes with this name

# Run in background
./script.sh &          # & sends process to background
nohup ./script.sh &    # Run and keep running after you log out

# Bring back to foreground
fg                     # Bring last background job to foreground
jobs                   # List background jobs
```

### systemd — Service Management

Most modern Linux systems use `systemd` to manage services (nginx, docker, jenkins, etc.)

```bash
sudo systemctl start nginx       # Start service
sudo systemctl stop nginx        # Stop service
sudo systemctl restart nginx     # Restart service
sudo systemctl reload nginx      # Reload config (no downtime)
sudo systemctl status nginx      # Check if running
sudo systemctl enable nginx      # Start automatically on boot
sudo systemctl disable nginx     # Don't start on boot
sudo systemctl is-active nginx   # Returns "active" or "inactive"

# View logs for a service
journalctl -u nginx              # All logs
journalctl -u nginx -f           # Follow logs (like tail -f)
journalctl -u nginx --since "1 hour ago"
```

---

## Networking Basics

### Check Network Configuration

```bash
ip addr                # Show all network interfaces and IP addresses
ip addr show eth0      # Show specific interface
ip route               # Show routing table
hostname               # Show hostname
hostname -I            # Show all IP addresses
```

### Test Connectivity

```bash
ping google.com                     # Test basic connectivity
ping -c 4 google.com                # Send exactly 4 packets
traceroute google.com               # Trace the network path
curl -I https://google.com          # Check HTTP response headers
wget https://example.com/file.zip   # Download a file
```

### Ports and Connections

```bash
netstat -tulpn          # Show all listening ports
ss -tulpn               # Modern replacement for netstat
lsof -i :80             # What process is using port 80?
lsof -i :443
```

Output explained:
```
Proto  Local Address    State   PID/Program
tcp    0.0.0.0:80       LISTEN  1234/nginx
tcp    0.0.0.0:443      LISTEN  1234/nginx
```

### DNS

```bash
nslookup google.com     # DNS lookup
dig google.com          # Detailed DNS lookup
dig google.com A        # Query A record (IPv4)
dig google.com MX       # Query MX record (mail)
cat /etc/resolv.conf    # DNS server configuration
cat /etc/hosts          # Local hostname overrides
```

### Firewall (UFW — Ubuntu)

```bash
sudo ufw status              # Check firewall status
sudo ufw enable              # Enable firewall
sudo ufw allow 22            # Allow SSH
sudo ufw allow 80            # Allow HTTP
sudo ufw allow 443           # Allow HTTPS
sudo ufw allow 8080/tcp      # Allow specific port
sudo ufw deny 3306           # Block MySQL from outside
sudo ufw delete allow 80     # Remove a rule
```

---

## SSH — Secure Shell

SSH lets you securely connect to remote servers.

### Two Authentication Methods

**1. Password** — simple but less secure
```bash
ssh ankit@192.168.1.100        # Connect with password prompt
ssh -p 2222 ankit@server.com   # Custom port
```

**2. SSH Key (recommended)** — more secure, no password needed

How it works:
- You generate a **key pair**: private key (stays on your machine) + public key (goes on server)
- Server checks if the connecting client has the matching private key
- Never share your private key

```bash
# Generate a key pair
ssh-keygen -t ed25519 -C "ankit@mycomputer"
# Creates: ~/.ssh/id_ed25519 (private) and ~/.ssh/id_ed25519.pub (public)

# Copy your public key to a server
ssh-copy-id ankit@192.168.1.100
# Note: ssh-copy-id requires password auth to work the first time.
# Once your key is copied, you can disable password auth (see PasswordAuthentication below).
# This adds your public key to ~/.ssh/authorized_keys on the server
# Note: ssh-copy-id requires you to authenticate with a password at least once.
# If password auth is disabled, manually append your public key to ~/.ssh/authorized_keys on the server.

# Now connect without a password
ssh ankit@192.168.1.100
```

### SSH Config File

Save your connection details so you don't need to type them every time:

```bash
# ~/.ssh/config
Host myserver
    HostName 192.168.1.100
    User ankit
    IdentityFile ~/.ssh/id_ed25519
    Port 22

Host prod
    HostName prod.mycompany.com
    User deploy
    IdentityFile ~/.ssh/prod_key
```

Now you can just type:
```bash
ssh myserver     # instead of: ssh -i ~/.ssh/id_ed25519 ankit@192.168.1.100
```

### SSH Server Configuration

The SSH server config is at `/etc/ssh/sshd_config`. Key settings:

```bash
Port 22   # Optional: change to reduce automated scan noise (not a real security measure)
PermitRootLogin no               # Never allow root login directly
# WARNING: Only set this after confirming key-based auth works.
# If you set this before your key is in authorized_keys, you will lock yourself out.
PasswordAuthentication no        # Disable passwords — keys only
PubkeyAuthentication yes         # Enable key auth
AllowUsers ankit deploy          # Whitelist specific users
```

After changing sshd_config:
```bash
sudo systemctl restart sshd      # Apply changes
```

### SSH Tunneling (Port Forwarding)

```bash
# Local forwarding — access remote port locally
ssh -L 8080:localhost:80 ankit@server.com
# Now http://localhost:8080 on your machine → port 80 on the server

# Remote forwarding — expose local port to remote server
ssh -R 9090:localhost:3000 ankit@server.com
```

### SCP — Copy Files Over SSH

```bash
scp file.txt ankit@server.com:/home/ankit/          # Upload
scp ankit@server.com:/home/ankit/file.txt ./         # Download
scp -r mydir/ ankit@server.com:/home/ankit/          # Upload directory
```

---

## Exercises

1. Check all running processes and find the one using the most CPU.
2. Start a web server (`python3 -m http.server 8080`), find its PID, then kill it gracefully.
3. Generate an SSH key pair using ed25519. Where are the files stored?
4. Find which process is listening on port 22 on your machine.
5. Use `curl` to make an HTTP request to `https://httpbin.org/get` and view the response.
6. Add an entry to `~/.ssh/config` for a server (real or fictional).

---

## Key Takeaways

- `ps aux | grep <name>` to find a running process
- `systemctl` manages services — know start/stop/status/enable
- Always use SSH keys, not passwords
- Never allow root login via SSH
- Know what ports your services are listening on (`ss -tulpn`)
