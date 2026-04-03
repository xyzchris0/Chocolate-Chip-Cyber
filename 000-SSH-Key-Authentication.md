# Setting Up SSH Key Authentication on Linux

> A quick guide to replacing password-based SSH logins with key-based authentication.

---

## Table of Contents

1. [What is SSH Key Authentication?](#what-is-ssh-key-authentication)
2. [Prerequisites](#prerequisites)
3. [Generate an SSH Key Pair](#1-generate-an-ssh-key-pair)
4. [Copy Your Public Key to the Server](#2-copy-your-public-key-to-the-server)
5. [Test Key-Based Authentication](#3-test-key-based-authentication)
6. [Disable Password Authentication](#4-disable-password-authentication)
7. [Troubleshooting](#troubleshooting)

---

## What is SSH Key Authentication?

**SSH (Secure Shell)** is a cryptographic network protocol that lets you securely run commands on a remote machine over an untrusted network. All traffic is tunneled through an encrypted connection using the TCP/IP protocol stack.

By default, SSH allows you to authenticate with a password, but passwords are vulnerable to **brute-force attacks**, credential stuffing, and phishing. A stronger alternative is **SSH key authentication**, which uses a linked key pair:

| Key | Location | Purpose |
|-----|----------|---------|
| **Private Key** | Your local machine (secret) | Proves your identity for other device |
| **Public Key** | Remote server (`~/.ssh/authorized_keys`) | Validates your private key |

For example if an attacker knows your username & server IP, they cannot log in without your private key. Removes the ability for password-guessing attacks.

---

## Prerequisites

- A local Linux/macOS machine
- A remote Linux server you can currently reach via SSH (Confirm SSH is enabled)
- `sudo` or root access on the remote server 
- `openssh-client` installed locally (usually pre-installed on fresh machines)

---

## 1. Generate an SSH Key Pair

On your **local machine**, run `ssh-keygen` to generate a new key pair. Using the `-t ed25519` flag is recommended over the older RSA algorithm.

```bash
$ ssh-keygen -t ed25519
```
> ```bash
> $ ssh-keygen -t rsa -b 4096"
> ```

**Example output:**

```
Generating public/private ed25519 key pair.
Enter file in which to save the key (/home/client/.ssh/id_ed25519): ***/home/client/.ssh/target-server-key***
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /home/client/.ssh/target-server-key
Your public key has been saved in /home/client/.ssh/target-server-key.pub
The key fingerprint is:
SHA256:CAxxx/tt5xxxx+vsxxxx your_email@example.com
The key's randomart image is:
+--[ED25519 256]--+
|        +=+=== o+|
|       .  =+= =+o|
|      .. ..o.+.**|
|        .. .= orr|
|        S    oo=*|
|         .  . =o.|
|             . +.|
|              . o|
|                .|
+----[SHA256]-----+
```

When prompted for a **file path**, you can press `Enter` for default, or replace with a custom name like 'target-server-key'.

When prompted for a **passphrase**, you can either press `Enter` or add a passphrase.

---

## 2. Copy Your Public Key to the Server

Use `ssh-copy-id` to securely copy your **public key** to the remote server's `authorized_keys` file. This will prompt you for your password **one last time**.

```bash
$ ssh-copy-id -i ~/.ssh/target-server-key.pub target-server@192.168.1.100
```

**Expected output:**

```
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/home/client/.ssh/target-server-key.pub"
Number of key(s) added: 1

Now try logging into the machine, with the following command: " $ ssh 'target-server@192.168.1.100'"
and check to make sure that only the key(s) you wanted were added.
```

> **If you don't have the `ssh-copy-id`?** You manually do it:
> ```bash
> $ cat ~/.ssh/target-server-key.pub | ssh target-server@192.168.1.100 \
>   "$ mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
> ```

---

## 3. Test Key-Based Authentication

Before disabling password auth, **verify your key works**:

```bash
$ ssh -i ~/.ssh/target-server-key target-server@192.168.1.100
```

Now that you are logged into **without being prompted for a password** (or key passphrase).

>  **Update your Config File** by adding an entry to `~/.ssh/config` on your local machine:
> ```
> Host my-server
>     HostName 192.168.1.100
>     User target-server
>     IdentityFile ~/.ssh/target-server-key
> ```
> Now you can simply type: `ssh my-server`

---

## 4. Disable Password Authentication

> Make sure to only do this after confirming key-based login works as described above or you might be locked out.

On the **remote server**, open the SSH daemon config file:

```bash
$ sudo nano /etc/ssh/sshd_config
```

Find & update (add) these lines:

```
PasswordAuthentication no
ChallengeResponseAuthentication no
UsePAM no
```

Save the file, **restart the SSH service** to apply the changes:

```bash
# On systemd-based systems (Ubuntu, Debian, CentOS 7+)
$ sudo systemctl restart sshd

# On older init-based systems
$ sudo service ssh restart
```

**Verify the changes** Now open a *new* terminal and try logging in with a password:

```bash
$ ssh -o PasswordAuthentication=yes target-server@192.168.1.100
# Expected: Permission denied (publickey)
```

Password authentication is now disabled. 

---

## Troubleshooting

| Problem | Likely Cause | Fix |
|---------|-------------|-----|
| `Permission denied (publickey)` | Key not copied or wrong key used | Re-run `ssh-copy-id`; check `-i` flag points to correct key |
| `WARNING: UNPROTECTED PRIVATE KEY FILE!` | Permissions too open | `chmod 600 ~/.ssh/target-server-key` |
| `Connection refused` | SSH daemon not running | `sudo systemctl start sshd` |
| `Host key verification failed` | Server fingerprint changed | Remove old entry: `ssh-keygen -R 192.168.1.100` |
| Locked out after disabling passwords | Key not working before change | Access via console/VNC, re-enable `PasswordAuthentication yes`, then restart |

---
