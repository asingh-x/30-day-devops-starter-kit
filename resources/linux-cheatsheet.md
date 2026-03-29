# Linux Cheatsheet

A quick-reference guide for common Linux commands used in DevOps workflows.

---

## Navigation

```bash
pwd                   # Print current working directory
ls                    # List files
ls -la                # List all files with permissions and hidden files
cd /path/to/dir       # Change directory
cd ~                  # Go to home directory
cd -                  # Go to previous directory
tree                  # Display directory tree (install with apt/yum)
```

---

## File Operations

```bash
# Create
touch file.txt                    # Create empty file
mkdir -p /path/to/nested/dir      # Create directory and parents

# Copy / Move / Delete
cp file.txt /tmp/                 # Copy file
cp -r dir/ /tmp/                  # Copy directory recursively
mv file.txt newname.txt           # Move or rename
rm file.txt                       # Delete file
rm -rf dir/                       # Delete directory recursively (use with care)

# View
cat file.txt                      # Print entire file
less file.txt                     # Page through file (q to quit)
head -n 20 file.txt               # First 20 lines
tail -n 20 file.txt               # Last 20 lines
tail -f /var/log/syslog           # Follow a log file in real time

# Find
find / -name "*.log" 2>/dev/null  # Find files by name
find /var -mtime -1               # Files modified in the last 1 day
locate filename                   # Fast search using index (updatedb first)
which python3                     # Find binary location in PATH
```

---

## Permissions

### chmod — Change File Permissions

```bash
chmod 755 script.sh               # Owner: rwx, Group: r-x, Others: r-x
chmod 644 file.txt                # Owner: rw-, Group: r--, Others: r--
chmod 600 ~/.ssh/id_rsa           # Owner: rw-, no access for others (private key)
chmod +x script.sh                # Add execute for all
chmod -R 755 /var/www/html        # Recursive chmod

chown user:group file.txt         # Change owner and group
chown -R www-data:www-data /var/www
```

### Permission Values Reference

| Value | Binary | Meaning     |
|-------|--------|-------------|
| 7     | 111    | rwx (read, write, execute) |
| 6     | 110    | rw- (read, write)          |
| 5     | 101    | r-x (read, execute)        |
| 4     | 100    | r-- (read only)            |
| 0     | 000    | --- (no permissions)       |

### Common Permission Combinations

| Octal | Who       | Typical Use                        |
|-------|-----------|------------------------------------|
| 755   | rwxr-xr-x | Executables, public directories    |
| 644   | rw-r--r-- | Config files, web assets           |
| 600   | rw------- | Private SSH keys, sensitive files  |
| 700   | rwx------ | Private scripts, user home dirs    |
| 777   | rwxrwxrwx | Avoid — insecure                   |

---

## Users and Groups

```bash
# User management
whoami                            # Current user
id                                # UID, GID, and groups for current user
adduser username                  # Create a new user (interactive)
useradd -m -s /bin/bash username  # Create user with home dir and shell
passwd username                   # Set password
userdel -r username               # Delete user and home directory
usermod -aG sudo username         # Add user to sudo group

# Group management
groups                            # List groups for current user
groupadd devops                   # Create a group
groupdel devops                   # Delete a group
gpasswd -a username devops        # Add user to group

# Switch users
su - username                     # Switch to user (new login shell)
sudo command                      # Run command as root
sudo -i                           # Open root shell
```

---

## Processes

```bash
# Viewing processes
ps aux                            # All running processes
ps aux | grep nginx               # Filter for a specific process
top                               # Interactive process viewer
htop                              # Improved top (install separately)
pgrep nginx                       # Get PID by process name

# Killing processes
kill PID                          # Send SIGTERM (graceful stop)
kill -9 PID                       # Send SIGKILL (force stop)
killall nginx                     # Kill all processes named nginx
pkill -f "python app.py"          # Kill by matching command string

# Background / Foreground
command &                         # Run in background
jobs                              # List background jobs
fg %1                             # Bring job 1 to foreground
bg %1                             # Resume job 1 in background
nohup command &                   # Run command immune to hangup

# systemctl (systemd)
systemctl start nginx             # Start a service
systemctl stop nginx              # Stop a service
systemctl restart nginx           # Restart a service
systemctl reload nginx            # Reload config without restart
systemctl enable nginx            # Enable service at boot
systemctl disable nginx           # Disable service at boot
systemctl status nginx            # Check service status
systemctl list-units --type=service  # List all services
journalctl -u nginx -f            # Follow logs for a service
journalctl -u nginx --since "1 hour ago"
```

---

## Networking

```bash
# IP and interfaces
ip addr show                      # Show IP addresses
ip addr show eth0                 # Show specific interface
ip route show                     # Show routing table
ip link set eth0 up               # Bring interface up

# Connections and sockets
ss -tulnp                         # Show listening TCP/UDP ports with PID
ss -tnp                           # Active TCP connections
netstat -tulnp                    # Alternative (older systems)

# Connectivity
ping -c 4 google.com              # Send 4 ICMP packets
traceroute google.com             # Trace route to host
dig example.com                   # DNS lookup
nslookup example.com              # DNS lookup (alternative)
host example.com                  # Simple DNS lookup

# curl — HTTP requests
curl https://example.com                          # GET request
curl -o file.html https://example.com             # Save output to file
curl -I https://example.com                       # Headers only
curl -X POST -d '{"key":"val"}' \
  -H "Content-Type: application/json" \
  https://api.example.com/endpoint                # POST with JSON
curl -u user:pass https://api.example.com         # Basic auth
curl -sk https://example.com                      # Skip TLS verification (testing only)

# wget
wget https://example.com/file.zip                 # Download file
wget -q --spider https://example.com              # Check URL without downloading

# Firewall (ufw / iptables)
ufw status                        # Check firewall status
ufw allow 22/tcp                  # Allow SSH
ufw allow 80/tcp                  # Allow HTTP
ufw enable                        # Enable firewall
iptables -L -n -v                 # List iptables rules
```

---

## SSH

```bash
# Connect
ssh user@hostname                 # Basic SSH connection
ssh -p 2222 user@hostname         # Connect on non-standard port
ssh -i ~/.ssh/mykey user@hostname # Use specific private key

# Key generation
ssh-keygen -t ed25519 -C "your_email@example.com"      # Recommended (ed25519)
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"  # RSA 4096-bit

# Copy public key to remote server
ssh-copy-id user@hostname
# Or manually:
cat ~/.ssh/id_ed25519.pub | ssh user@hostname "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"

# SSH agent
eval "$(ssh-agent -s)"           # Start agent
ssh-add ~/.ssh/id_ed25519         # Add key to agent
ssh-add -l                        # List loaded keys

# SCP — Secure Copy
scp file.txt user@host:/remote/path/           # Local to remote
scp user@host:/remote/file.txt /local/path/   # Remote to local
scp -r dir/ user@host:/remote/path/           # Copy directory

# rsync (preferred over scp for directories)
rsync -avz ./local/ user@host:/remote/        # Sync local to remote
rsync -avz --delete ./local/ user@host:/remote/  # Mirror (delete extra files)

# SSH config file (~/.ssh/config)
# Example entry — allows: ssh myserver
Host myserver
    HostName 203.0.113.10
    User ubuntu
    IdentityFile ~/.ssh/id_ed25519
    Port 22

Host bastion
    HostName bastion.example.com
    User ec2-user
    IdentityFile ~/.ssh/bastion_key

Host private-host
    HostName 10.0.1.50
    User ubuntu
    ProxyJump bastion         # Jump through bastion host
```

---

## Text Processing

### grep — Search Text

```bash
grep "error" app.log                       # Search for pattern
grep -i "error" app.log                    # Case-insensitive
grep -r "TODO" ./src/                      # Recursive search
grep -n "error" app.log                    # Show line numbers
grep -v "debug" app.log                    # Invert match (exclude)
grep -c "error" app.log                    # Count matches
grep -E "error|warn|critical" app.log      # Extended regex (OR)
grep -A 3 "FATAL" app.log                  # 3 lines after match
grep -B 3 "FATAL" app.log                  # 3 lines before match
grep -C 3 "FATAL" app.log                  # 3 lines before and after
grep "^ERROR" app.log                      # Lines starting with ERROR
grep "\.log$" filelist.txt                 # Lines ending with .log
```

### awk — Column Processing

```bash
awk '{print $1}' file.txt                  # Print first column
awk '{print $1, $3}' file.txt              # Print columns 1 and 3
awk -F: '{print $1}' /etc/passwd           # Custom delimiter (colon)
awk '{sum += $1} END {print sum}' nums.txt # Sum a column
awk 'NR==5' file.txt                       # Print line 5
awk 'NR>=5 && NR<=10' file.txt             # Print lines 5 to 10
awk '$3 > 100' file.txt                    # Print rows where col 3 > 100
awk '{print NR, $0}' file.txt              # Add line numbers
ps aux | awk '{print $1, $2, $11}'         # Show user, PID, command
df -h | awk 'NR>1 {print $5, $6}'          # Show disk usage %
```

### sed — Stream Editor

```bash
sed 's/foo/bar/' file.txt                  # Replace first occurrence per line
sed 's/foo/bar/g' file.txt                 # Replace all occurrences
sed -i 's/foo/bar/g' file.txt              # Edit file in place
sed -i.bak 's/foo/bar/g' file.txt          # Edit in place, keep backup
sed -n '5,10p' file.txt                    # Print lines 5 to 10
sed '5,10d' file.txt                       # Delete lines 5 to 10
sed '/^#/d' config.conf                    # Delete comment lines
sed '/^$/d' file.txt                       # Delete empty lines
sed 's/  */ /g' file.txt                   # Collapse multiple spaces
sed 's/^/  /' file.txt                     # Indent every line by 2 spaces
```

---

## Package Management

### apt (Debian / Ubuntu)

```bash
apt update                        # Refresh package index
apt upgrade                       # Upgrade all packages
apt install nginx                 # Install a package
apt remove nginx                  # Remove package (keep config)
apt purge nginx                   # Remove package and config
apt autoremove                    # Remove unused dependencies
apt search nginx                  # Search packages
apt show nginx                    # Package details
dpkg -l | grep nginx              # List installed packages matching pattern
```

### yum / dnf (RHEL / CentOS / Amazon Linux)

```bash
yum update                        # Update all packages
yum install httpd                 # Install a package
yum remove httpd                  # Remove a package
yum search httpd                  # Search packages
yum info httpd                    # Package details
yum list installed                # List all installed packages
dnf install nginx                 # dnf is the modern replacement for yum
rpm -qa | grep nginx              # List installed RPM packages matching pattern
```

---

## Disk and Memory

```bash
# Disk usage
df -h                             # Disk space for all filesystems (human readable)
df -h /var                        # Disk space for specific path
du -sh /var/log                   # Total size of a directory
du -sh *                          # Size of all items in current directory
du -sh * | sort -rh | head -10    # Top 10 largest items

# Memory
free -h                           # Memory usage (human readable)
free -m                           # Memory usage in MB
cat /proc/meminfo                 # Detailed memory info

# top / htop
top                               # Interactive process viewer
# Keys inside top: q=quit, k=kill PID, M=sort by memory, P=sort by CPU

# vmstat — system activity
vmstat 2 5                        # Report every 2 seconds, 5 times

# iostat — disk I/O
iostat -x 2                       # Extended disk stats every 2 seconds

# lsblk — block devices
lsblk                             # List all block devices
lsblk -f                          # With filesystem info
```

---

## Cron

Cron syntax: `minute hour day-of-month month day-of-week command`

```
*  *  *  *  *  command
|  |  |  |  |
|  |  |  |  +--- Day of week (0-7, Sunday = 0 or 7)
|  |  |  +------ Month (1-12)
|  |  +--------- Day of month (1-31)
|  +------------ Hour (0-23)
+--------------- Minute (0-59)
```

### Examples

```bash
# Edit crontab
crontab -e                        # Edit cron jobs for current user
crontab -l                        # List current cron jobs
crontab -r                        # Remove all cron jobs

# Examples
0 * * * *       /usr/bin/python3 /app/hourly.py          # Every hour at :00
0 2 * * *       /usr/bin/backup.sh                       # Daily at 02:00
*/15 * * * *    /usr/bin/check-health.sh                 # Every 15 minutes
0 0 * * 0       /usr/bin/weekly-report.sh                # Every Sunday at midnight
0 9 1 * *       /usr/bin/monthly-invoice.py              # 1st of month at 09:00
30 6 * * 1-5    /usr/bin/workday-task.sh                 # Weekdays at 06:30
0 0 * * *       find /tmp -mtime +7 -delete              # Daily cleanup of old tmp files

# Redirect output to log
0 2 * * * /usr/bin/backup.sh >> /var/log/backup.log 2>&1
```

### Special Cron Strings

```
@reboot    Run once at startup
@hourly    0 * * * *
@daily     0 0 * * *
@weekly    0 0 * * 0
@monthly   0 0 1 * *
@yearly    0 0 1 1 *
```
