---
date: 2025-11-14T00:00:00+00:00
draft: false
title: "SSH Guide: From Basics to Secure Setup"
description: "A practical guide to SSH - understanding the basics, setting up key authentication, and securing your remote connections"
authors: ["Simeon Ivanov"]
tags: ["ssh", "security", "linux"]
---

## What is SSH?

SSH (Secure Shell) creates an encrypted connection between your computer and a remote server.

Everything you type and all output is encrypted.

**Components:**

- **SSH client** - On your computer (you type `ssh`)
- **SSH server** - On the remote machine (`sshd` daemon)
- **Encryption** - Your traffic is unreadable to anyone in between

## How SSH Works

1. You run `ssh user@server`
2. Client connects to server port 22
3. They negotiate encryption (key exchange)
4. You authenticate (password or key)
5. Encrypted channel established
6. Your commands run on the server

## Password vs Key Authentication

**Password authentication:**
- Simple but less secure
- You type password every time
- Vulnerable to brute force attacks

**Key authentication:**
- More secure
- No password needed once set up
- Based on cryptographic key pairs
- Required by many cloud providers

## Generate SSH Keys

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
```

**Output:**

```
Generating public/private ed25519 key pair.
Enter file in which to save the key (/home/user/.ssh/id_ed25519):
Enter passphrase (empty for no passphrase):
```

Press Enter to accept the default location. You can add a passphrase for extra security.

**This creates:**
- `~/.ssh/id_ed25519` - Private key (NEVER share this)
- `~/.ssh/id_ed25519.pub` - Public key (safe to share)

## Copy Key to Server

```bash
ssh-add ~/.ssh/id_ed25519
ssh-copy-id user@192.168.1.100
```

You'll enter your password once. After this, you can log in without a password.

**Manual method** (if `ssh-copy-id` isn't available):

```bash
cat ~/.ssh/id_ed25519.pub | ssh user@192.168.1.100 "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

## Verify Key Authentication is Working

After copying your key, verify it's actually being used. Use verbose mode:

```bash
ssh -v user@192.168.1.100
```

Look for these lines in the output:

```
debug1: Authentications that can continue: publickey,password
debug1: Offering public key: /home/user/.ssh/id_ed25519 ED25519
debug1: Server accepts key: /home/user/.ssh/id_ed25519 ED25519
debug1: Authentication succeeded (publickey).
```

**The key lines are:**
- `Offering public key` - SSH is trying your key
- `Server accepts key` - The server recognized your public key
- `Authentication succeeded (publickey)` - You're in with your key, not password

If you see `Authentication succeeded (password)` instead, your key isn't set up correctly.

## Disable Password Authentication

Once key authentication works, disable password login for better security. This prevents brute force attacks.

**On the server** (the machine you SSH into):

```bash
sudo vim /etc/ssh/sshd_config
```

Find and change these lines:

```
PasswordAuthentication no
PubkeyAuthentication yes
```

If the lines have `#` at the start, remove the `#` to uncomment them.

**Restart the SSH service:**

```bash
sudo systemctl restart sshd
```

**Test before disconnecting:** Open a new terminal and verify you can still connect:

```bash
ssh user@192.168.1.100
```

If key auth fails now, you'll be locked out. That's why you test with a separate connection while keeping your current session open.

## SSH Config File

Create `~/.ssh/config`:

```bash
vim ~/.ssh/config
```

Add:

```
Host homelab
    HostName 192.168.1.100
    User yourusername
    IdentityFile ~/.ssh/id_ed25519
```

Now instead of:

```bash
ssh yourusername@192.168.1.100
```

You just type:

```bash
ssh homelab
```

**Set proper permissions:**

```bash
chmod 600 ~/.ssh/config
```

## Copy Files: scp

**Copy local file to remote server:**

```bash
scp file.txt user@server:/home/user/
```

**Copy remote file to local directory:**

```bash
scp user@server:/var/log/syslog ./
```

**Copy a directory recursively:**

```bash
scp -r directory/ user@server:/home/user/
```

## SSH Best Practices

- **Use keys, not passwords** - Disable password auth if possible
- **Use a passphrase** - Protects your key if someone gets the file
- **Use the config file** - Simpler commands, fewer mistakes
- **Keep private keys private** - Never share, never commit to git
- **Keep your system updated** - Regular security patches matter
- **Use specific keys per service** - Don't reuse the same key everywhere

## Troubleshooting

**Permission errors:**

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_ed25519
chmod 644 ~/.ssh/id_ed25519.pub
chmod 600 ~/.ssh/authorized_keys
```

**Connection refused:**
- Check if SSH server is running: `sudo systemctl status sshd`
- Check firewall rules: `sudo ufw status`
- Verify port 22 is open: `sudo ss -tulpn | grep :22`

**Key not working:**
- Use `ssh -v` to see what's happening
- Check server logs: `sudo journalctl -u sshd -n 50`
- Verify your public key is in `~/.ssh/authorized_keys` on the server

## Conclusion

SSH is the backbone of remote Linux administration. Key-based authentication is more secure and more convenient than passwords. Take the time to set it up properly, and your future self will thank you.

Start with key authentication, use the config file for convenience, and disable password login once everything works. These three steps will dramatically improve both your security and your workflow.
