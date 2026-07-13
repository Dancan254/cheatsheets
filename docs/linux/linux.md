# Linux

Reach for this for permissions, processes, systemd, networking, and disk - the stuff
that saves you when a box misbehaves.

## Permissions

```bash
ls -l                       # rwx rwx rwx = user group other
chmod 644 file              # rw-r--r--  (typical file)
chmod 755 script.sh         # rwxr-xr-x  (executable)
chmod +x script.sh          # add execute
chmod -R 750 dir/           # recursive
chown user:group file       # change owner + group
chown -R app:app /app       # recursive
```

Octal: read=4, write=2, execute=1. `7`=rwx, `6`=rw-, `5`=r-x, `4`=r--.

```bash
umask                       # default-permission mask (022 → new files 644, dirs 755)
sudo -u app command         # run as another user
id                          # your uid/gid/groups
```

## Processes

```bash
ps aux                      # all processes (a=all users, u=detail, x=no-tty)
ps aux | grep java          # find a process
pgrep -f my-app             # PIDs matching a pattern
top                         # live;  htop is nicer if installed
kill 1234                   # SIGTERM (graceful)
kill -9 1234                # SIGKILL (force - last resort, no cleanup)
pkill -f my-app             # kill by name pattern
kill -HUP 1234              # signal reload (many daemons re-read config)

jobs                        # background jobs in this shell
command &                   # run in background
nohup command &             # survive terminal close
disown %1                   # detach job from shell
fg %1  /  bg %1             # foreground / background a job
```

## systemd (services)

```bash
systemctl status nginx              # is it running? recent logs
systemctl start|stop|restart nginx
systemctl reload nginx              # reload config without dropping connections
systemctl enable nginx              # start on boot
systemctl disable nginx
systemctl is-enabled nginx
systemctl list-units --type=service --state=running
systemctl daemon-reload             # after editing a .service file

journalctl -u nginx                 # logs for a unit
journalctl -u nginx -f              # follow (like tail -f)
journalctl -u nginx --since "1 hour ago"
journalctl -p err -b                # errors since last boot
journalctl --disk-usage             # how much space logs use
```

## Networking

```bash
ss -tulpn                   # listening sockets (t=tcp u=udp l=listen p=pid n=numeric)
ss -tulpn | grep 8080       # what's on port 8080?
ip a                        # interfaces + IPs  (replaces ifconfig)
ip r                        # routing table
ping host
curl -I https://site.com    # headers only
curl -v https://site.com    # verbose (TLS, redirects)
dig example.com             # DNS lookup;  dig +short example.com
nslookup example.com
traceroute host
nc -zv host 5432            # test if a port is open (connection test)
```

"Can't connect" flow: `ss -tulpn` (is it listening?) → `curl -v` / `nc -zv` (can I reach it?) → check firewall (`ufw status` / SG).

## Disk & files

```bash
df -h                       # disk free per mount (human-readable)
du -sh *                    # size of each item in cwd
du -sh * | sort -rh | head  # biggest items first
ncdu                        # interactive disk usage (if installed)
find / -size +500M 2>/dev/null    # big files
lsof -p 1234                # files a process has open
lsof -i :8080               # what's using port 8080
lsblk                       # block devices / partitions
```

Disk full but `df` looks fine? A deleted-but-open file. `lsof | grep deleted` - restart the holder.

## Finding & tailing

```bash
find . -name "*.log" -mtime -1        # modified in last 24h
find . -type f -size +100M
tail -f app.log                       # follow
tail -n 100 app.log
grep -rn "NullPointer" logs/          # recursive, line numbers
watch -n 2 'df -h'                    # rerun a command every 2s
```

## Env & shell

```bash
env                         # all env vars
export VAR=value            # set for session + children
echo $PATH
which java                  # first match in PATH
type -a java                # all matches + aliases/functions
history | grep docker       # past commands
source ~/.bashrc            # reload shell config
```

## Users & sudo

```bash
whoami
sudo command
sudo -i                     # root shell
usermod -aG docker $USER    # add self to docker group (re-login to apply)
cat /etc/passwd             # users;  groups <user> for a user's groups
```

## Gotchas / things I always forget

- `kill -9` skips cleanup (no flushing, no shutdown hooks). Try plain `kill` (SIGTERM) first - `-9` only when hung.
- After editing a `.service` file you **must** `systemctl daemon-reload` before restart, or it uses the old definition.
- `ss -tulpn` needs `sudo` to show the PID/process for ports owned by others.
- Adding yourself to a group (`usermod -aG`) doesn't apply until you log out/in (or `newgrp`).
- `chmod -R 777` is never the fix. It's a security hole and hides the real permission problem.
- `df` shows free space; if it says full but `du` disagrees, it's a deleted-but-still-open file held by a process (`lsof | grep deleted`).
- `nohup cmd &` or `disown` to survive logout - a plain `&` job dies when the shell (SSH session) closes.
- `>` truncates, `>>` appends. One wrong keystroke nukes a file. `2>&1` redirects stderr to wherever stdout points.

## Quick reference

| Task | Command |
|---|---|
| What's on a port | `ss -tulpn \| grep PORT` |
| Port reachable? | `nc -zv host PORT` |
| Service logs | `journalctl -u NAME -f` |
| Biggest dirs | `du -sh * \| sort -rh \| head` |
| Kill by name | `pkill -f NAME` |
| Follow a log | `tail -f FILE` |
| Recursive grep | `grep -rn "TEXT" DIR/` |
| After editing .service | `systemctl daemon-reload` |
