# Day 1 — Linux Basics

## Why Linux?

Almost every server in the world runs Linux. As a DevOps engineer, you will spend most of your day in a Linux terminal. Learning Linux is not optional.

The most common distributions you will see at work:
- **Ubuntu** — most common for development and servers
- **Amazon Linux** — used on AWS EC2
- **CentOS / RHEL** — used in enterprise environments

---

## The Filesystem

Linux organizes everything in a single tree starting from `/` (root).

```
/
├── etc/        Configuration files
├── var/        Logs, variable data
├── home/       User home directories (/home/ankit)
├── usr/        User programs and libraries
├── bin/        Essential binaries (ls, cp, mv)
├── tmp/        Temporary files (cleared on reboot)
├── proc/       Virtual filesystem for process info
└── dev/        Device files
```

**Key rule:** Everything in Linux is a file — including devices, processes, and network sockets.

---

## Essential Commands

### Navigation

```bash
pwd                    # Print current directory
ls                     # List files
ls -la                 # List all files with details (permissions, size, owner)
cd /etc                # Go to /etc
cd ~                   # Go to your home directory
cd ..                  # Go up one level
cd -                   # Go back to previous directory
```

### File Operations

```bash
touch file.txt         # Create an empty file
mkdir mydir            # Create a directory
mkdir -p a/b/c         # Create nested directories
cp file.txt backup.txt # Copy a file
mv file.txt newname.txt# Rename or move a file
rm file.txt            # Delete a file
rm -rf mydir           # Delete a directory and all its contents (careful!)
```

### Viewing Files

```bash
cat file.txt           # Print entire file
less file.txt          # Scroll through a file (q to quit)
head -20 file.txt      # First 20 lines
tail -20 file.txt      # Last 20 lines
tail -f /var/log/syslog# Follow a log file in real time
```

### Searching

```bash
grep "error" file.txt          # Search for "error" in a file
grep -r "error" /var/log/      # Search recursively in a directory
grep -i "error" file.txt       # Case-insensitive search
find /etc -name "*.conf"       # Find files by name
find / -type f -size +100M     # Find files larger than 100MB
```

### Text Processing

```bash
# awk — process structured text (fields separated by spaces or delimiter)
awk '{print $1}' file.txt           # Print first column of each line
awk -F: '{print $1}' /etc/passwd    # Print usernames (delimiter is :)

# sed — find and replace in text
sed 's/old/new/g' file.txt          # Replace "old" with "new" (all occurrences)
sed -i 's/old/new/g' file.txt       # Replace in-place (modifies the file)

# cut — extract fields
cut -d: -f1 /etc/passwd             # Print first field, delimiter is :
```

### Getting Help

```bash
man ls                 # Full manual for the ls command
ls --help              # Quick help
which python3          # Find where a command is installed
type ls                # Is it a binary, alias, or shell builtin?
```

---

## Package Management

### Ubuntu / Debian

```bash
sudo apt update              # Refresh package list
sudo apt install nginx       # Install nginx
sudo apt remove nginx        # Remove nginx
sudo apt upgrade             # Upgrade all installed packages
```

### Amazon Linux / CentOS / RHEL

```bash
sudo yum update
sudo yum install nginx
sudo dnf install nginx       # newer systems use dnf
```

---

## Output Redirection and Pipes

```bash
# Redirect output to a file
ls -la > output.txt         # Overwrite
ls -la >> output.txt        # Append

# Redirect errors
command 2> error.txt        # Redirect stderr
# Every process has three standard streams: stdin (0), stdout (1), and stderr (2). `2>&1` means redirect the error stream (2) to wherever stdout (1) is currently going.
command > output.txt 2>&1   # Redirect both stdout and stderr

# Pipes — send output of one command as input to another
ls -la | grep ".txt"        # List only .txt files
cat /var/log/syslog | grep "error" | tail -20
```

---

## Exercises

1. Open a terminal and print your current directory.
2. Create the following directory structure in one command: `projects/devops/week1/`
3. Create a file called `notes.txt` inside `week1/` and write "Day 1 complete" into it without opening an editor. Example: `echo "Day 1 complete" > week1/notes.txt`
4. Find all `.log` files in `/var/log/` (you may not have access to read them, but you should be able to list them).
5. Use `grep` to find all lines containing "root" in `/etc/passwd`.
6. Use `awk` to print only the usernames (first field) from `/etc/passwd`.

---

## Key Takeaways

- Linux filesystem starts at `/` — know the main directories
- `man <command>` is always available when you're stuck
- Pipes (`|`) let you chain commands together — this is very powerful
- `grep`, `awk`, `sed` are tools you will use every single day
