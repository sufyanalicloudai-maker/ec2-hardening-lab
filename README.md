# EC2 Hardening Lab

Hardening an AWS EC2 server (Amazon Linux 2023) that runs Nginx. "Hardening" means making the server harder to hack: lock the doors, remove unnecessary entrances, and give keys only to people who need them.

This repository documents four security fixes I applied, with the exact commands, the reason behind each one, and screenshots proving the result.

## Screenshots

### Before hardening
![Settings before hardening](images/before-hardening.png)

### After hardening
![Settings after hardening](images/after-hardening.png)

## How the server is protected

```
Internet  -->  Security Group (the gate)  -->  EC2 Server (the computer)
                                                  |-- SSH   (remote control)
                                                  |-- Nginx (shows the website)
```

| Term | Meaning |
|---|---|
| EC2 | The computer rented from AWS |
| Linux | The operating system |
| Security group | The gate that controls who can connect to the server |
| SSH | Lets me control the computer remotely |
| Nginx | The web server that shows the website |
| `ec2-user` | My normal login account |
| `sudo` | Lets me perform admin tasks |

SSH settings live in `/etc/ssh/sshd_config` (`/etc` is the Linux settings folder, `ssh` is the SSH folder, `sshd_config` is the SSH server settings file).

## Summary of fixes

| # | Problem | Setting | Before | After |
|---|---|---|---|---|
| 1 | Direct root login | `PermitRootLogin` | Not set to `no` | `no` |
| 2 | Too many login attempts | `MaxAuthTries` | 6 (default) | `3` |
| 3 | Password login | `PasswordAuthentication` / `PubkeyAuthentication` | Not explicitly set | `no` / `yes` (keys only) |
| 4 | Nginx version leak | `server_tokens` | `Server: nginx/1.x.x` | `Server: nginx` |

## Before you start

Connect to the server and take a "before" snapshot:

```bash
sudo sshd -T | grep -Ei "permitrootlogin|maxauthtries|passwordauthentication|pubkeyauthentication"
curl -I http://localhost 2>/dev/null | grep -i server
```

`sshd -T` (capital T) prints the active SSH settings. Then back up both config files so every change can be undone:

```bash
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
sudo cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.bak
```

## Problem 1: Direct root login

**Issue:** Root is the most powerful Linux user and everyone knows its name, so an attacker only has to guess the password. One correct guess means the whole server is lost.

**Analogy:** Everyone knows the house owner's name, and the owner's door is the front door.

**Fix:** Turn off direct root login.

```bash
sudo nano /etc/ssh/sshd_config
```

Set the line to:

```
PermitRootLogin no
```

Apply it with three commands:

```bash
sudo sshd -t
sudo systemctl reload sshd
sudo sshd -T | grep -i permitrootlogin
```

- `sshd` is the SSH server and `-t` tests the config for mistakes.
- `systemctl` is the service manager and `reload` applies the change without dropping connections.
- `grep -i` shows only the matching line, ignoring upper or lower case.

Expected output: `permitrootlogin no`

![Root login disabled](images/problem1-root-login.png)

## Problem 2: Too many login attempts

**Issue:** By default one connection gets 6 attempts, which helps brute-force bots try many passwords.

**Analogy:** An ATM blocks the card after 3 wrong PINs.

**Fix:** Allow only 3 attempts per connection.

```bash
sudo nano /etc/ssh/sshd_config
```

Set the line to:

```
MaxAuthTries 3
```

```bash
sudo sshd -t
sudo systemctl reload sshd
sudo sshd -T | grep -i maxauthtries
```

Expected output: `maxauthtries 3`

![MaxAuthTries set to 3](images/problem2-maxauthtries.png)

## Problem 3: Key-only login

**Issue:** A password can be guessed, leaked, or weak. One weak password can mean a lost server.

**Analogy:** A password is a lock that opens with any matching key. A key pair is your own unique key that nobody else has.

**Fix:** Disable password login and allow only key login.

```bash
sudo nano /etc/ssh/sshd_config
```

Set these lines:

```
PasswordAuthentication no
PubkeyAuthentication yes
```

```bash
sudo sshd -t
sudo systemctl reload sshd
sudo sshd -T | grep -Ei "passwordauthentication|pubkeyauthentication"
```

Expected output:

```
passwordauthentication no
pubkeyauthentication yes
```

![Key-only login settings](images/problem3-key-only.png)

**Safety test:** I kept my first SSH session open and tested in a second terminal. Key login still worked, and a login attempt without the key was refused:

```bash
ssh -o PubkeyAuthentication=no ec2-user@PUBLIC_IP
```

Result: `Permission denied (publickey...)`

![Password login refused](images/problem3-password-refused.png)

## Problem 4: Nginx version leak

**Issue:** Every response said `Server: nginx/1.x.x`. Attackers search for known weaknesses of that exact version.

**Analogy:** The brand and model of the lock is written on the door.

**Fix:** Hide the version.

```bash
sudo nano /etc/nginx/nginx.conf
```

Add this line inside the `http { }` block:

```
server_tokens off;
```

```bash
sudo nginx -t
sudo systemctl reload nginx
curl -I http://localhost 2>/dev/null | grep -i server
```

Expected output: `Server: nginx` (no version numbers)

![Nginx version hidden](images/problem4-nginx-version.png)

## Final verification

```bash
sudo sshd -T | grep -Ei "permitrootlogin|maxauthtries|passwordauthentication|pubkeyauthentication"
curl -I http://localhost 2>/dev/null | grep -i server
sudo nginx -t
sudo systemctl status nginx --no-pager | head -5
```

Expected values: `permitrootlogin no`, `maxauthtries 3`, `passwordauthentication no`, `pubkeyauthentication yes`, and `Server: nginx` without a version.

## Troubleshooting and undo

| Problem | Fix |
|---|---|
| A setting does not change after reload | Another file overrides it. Run `sudo grep -ri "settingname" /etc/ssh/sshd_config.d/` and edit that file |
| `sshd -t` prints an error | There is a typo in `sshd_config`. Reopen the file and fix that line |
| `nginx -t` fails | Check that `server_tokens off;` has a semicolon and sits inside the `http { }` block |
| Locked out of SSH | Use **EC2 Instance Connect** in the AWS console to get in and restore the backup |
| Undo SSH changes | `sudo cp /etc/ssh/sshd_config.bak /etc/ssh/sshd_config` then `sudo systemctl reload sshd` |
| Undo Nginx changes | `sudo cp /etc/nginx/nginx.conf.bak /etc/nginx/nginx.conf` then `sudo systemctl reload nginx` |

## What I learned

- Hardening is a set of small changes that together make a server much harder to attack.
- Always test a config (`sshd -t`, `nginx -t`) before reloading it, and keep a working session open while changing SSH settings.
- Keys are safer than passwords, and limiting attempts and hiding version information reduce what attackers can learn or try.
- Back up config files before editing so every change can be reversed.

## Security note

Never upload your `.pem` key file, access keys, or screenshots that show them to GitHub.

## Cleanup

To avoid charges, stop or terminate the EC2 instance from the console when the lab is finished.

## Author

Sufyan
