# Day 2 — Files, Permissions, and Users

## Why This Matters

On a server, the wrong permissions can mean:
- A private key is readable by anyone (security breach)
- A web server cannot read its config file (outage)
- A script cannot execute (broken deployment)

Understanding permissions is essential for security and reliability.

---

## File Permissions

Every file in Linux has three permission sets:
- **Owner (u)** — the user who owns the file
- **Group (g)** — users in the file's group
- **Others (o)** — everyone else

Each set has three permission types:
- **r** — read (4)
- **w** — write (2)
- **x** — execute (1)

### Reading Permissions

```bash
ls -la
```

Output:
```
-rwxr-xr-- 1 ankit devops 1234 Mar 1 10:00 deploy.sh
```

Breaking it down:
```
- rwx r-x r--
│ │   │   └── Others: read only
│ │   └────── Group: read + execute
│ └────────── Owner: read + write + execute
└──────────── File type (- = file, d = directory, l = symlink)
```

### Why Does a Directory Need Execute Permission?

Execute on a **directory** means the ability to **enter** it (cd into it).

```bash
chmod 644 mydir   # removes execute — you can no longer cd into it
chmod 755 mydir   # correct: owner can rwx, others can r-x
```

This is one of the most common mistakes beginners make.

---

## Changing Permissions

### Symbolic Method

```bash
chmod u+x script.sh       # Add execute for owner
chmod g-w file.txt        # Remove write from group
chmod o+r file.txt        # Add read for others
chmod a+x script.sh       # Add execute for all (a = all)
chmod u+x,g-w file.txt    # Multiple changes at once
```

### Numeric (Octal) Method

Each permission is a number: r=4, w=2, x=1. Add them up.

| Octal | Binary | Permissions |
|-------|--------|-------------|
| 7     | 111    | rwx         |
| 6     | 110    | rw-         |
| 5     | 101    | r-x         |
| 4     | 100    | r--         |
| 0     | 000    | ---         |

```bash
chmod 755 script.sh    # rwxr-xr-x  (common for scripts/directories)
chmod 644 file.txt     # rw-r--r--  (common for files)
chmod 600 private.key  # rw-------  (SSH keys — owner only)
chmod 400 private.key  # r--------  (read-only, even by owner)
```

**Rule of thumb:**
- Directories: `755`
- Regular files: `644`
- Scripts: `755`
- SSH private keys: `600` or `400`

---

## Ownership

```bash
chown ankit file.txt           # Change owner to ankit
chown ankit:devops file.txt    # Change owner AND group
chown -R ankit /var/www/       # Recursive (all files in directory)
chgrp devops file.txt          # Change group only
```

---

## Users and Groups

### Managing Users

```bash
# View current user
whoami

# View all users
cat /etc/passwd

# Create a user
sudo useradd -m -s /bin/bash ankit    # -m creates home dir, -s sets shell

# Set password
sudo passwd ankit

# Delete user
sudo userdel -r ankit    # -r removes home directory too

# Switch users
su - ankit               # Switch to ankit (- loads full environment)
```

### Managing Groups

```bash
# View current user's groups
groups
id

# View all groups
cat /etc/group

# Create a group
sudo groupadd devops

# Add user to group
sudo usermod -aG devops ankit    # -a = append, -G = groups

# Verify
groups ankit
```

### sudo — Running Commands as Root

```bash
# Run a single command as root
sudo apt install nginx

# Open a root shell (use sparingly)
sudo -i

# Edit the sudoers file safely
sudo visudo
```

The sudoers file (`/etc/sudoers`) controls who can use sudo and what commands they can run.

```
# Allow ankit to run all commands without a password
ankit ALL=(ALL) NOPASSWD:ALL

# Allow the devops group to restart nginx
%devops ALL=(ALL) /bin/systemctl restart nginx
```

---

## Special Permissions

### SUID (Set User ID) — 4xxx

A file with SUID runs as the file's owner, not the user executing it.

```bash
chmod 4755 script.sh    # Sets SUID bit
ls -la script.sh
# -rwsr-xr-x (notice the 's' instead of 'x' for owner)
```

Example: `/usr/bin/passwd` has SUID so any user can change their own password (it needs root access to write to /etc/shadow).

### Sticky Bit — 1xxx

On a directory, prevents users from deleting files they don't own.

```bash
chmod 1777 /tmp    # The 't' in the permission means sticky bit is set
ls -la /
# drwxrwxrwt  tmp
```

---

## Exercises

1. Create a file `secret.txt`. Set permissions so only you can read and write it.
2. Create a script `hello.sh` with `echo "Hello World"`. Give it execute permission and run it.
3. Create a directory `shared/`. Set it so everyone can read and enter it, but only the owner can write.
4. Create a new user `testuser` and add them to a group called `developers`.
5. Check what groups your current user belongs to.
6. Find all files in `/usr/bin` that have SUID set (`-perm /4000`).

---

## Key Takeaways

- Permissions have three sets: owner, group, others
- Numbers: r=4, w=2, x=1 — add them up (`755` = `rwxr-xr-x`)
- Directories always need execute (`x`) to be accessible
- SSH private keys must be `600` — always
- `sudo` gives temporary root access — use it carefully
