---
title: "The Complete Guide to SSH Hardening"
date: 2025-11-14
lastmod: 2025-11-14
topics: ["ssh", "security"]
---

## Overview

SSH (Secure Shell) is the foundation of remote server administration, yet its default configuration leaves significant security gaps. A properly hardened SSH setup is not about security through obscurity—it's about implementing defense-in-depth with modern cryptographic standards, strict authentication policies, and comprehensive monitoring.

------

## Understanding the Threat Model

Before implementing security measures, it's essential to understand what we're protecting against:

- **Brute-force attacks**: Automated attempts to guess credentials
- **Credential stuffing**: Using leaked credentials from other breaches
- **Man-in-the-middle attacks**: Intercepting SSH connections
- **Cryptographic weaknesses**: Exploiting outdated algorithms
- **Privilege escalation**: Gaining unauthorized root access
- **Session hijacking**: Taking over active SSH sessions

------

## Prerequisites

This guide assumes you have:

- Root or sudo access to your server
- Basic understanding of Linux command line
- A backup access method (console access or alternative SSH configuration)
- OpenSSH 7.4 or newer (check with `ssh -V`)

------

> **⚠️ Critical Warning**
>
>  Always maintain a backup session when modifying SSH configuration. A misconfiguration can prevent you from accessing your server.

------

------

## Step 1: Preparation and Backup

Before making any changes, establish safety measures:

```bash
# Verify OpenSSH is installed and running
systemctl status sshd

# Check current OpenSSH version
ssh -V

# Create a timestamped backup of the original configuration
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.backup.$(date +%F)

# Keep a root session open throughout the configuration process
# This ensures you can revert changes if something breaks
```

------

## Step 2: Core SSH Configuration

The `/etc/ssh/sshd_config` file controls all SSH server behavior. Here is a hardened configuration with detailed explanations:

```bash
# Edit the SSH daemon configuration
sudo nano /etc/ssh/sshd_config
```

### Protocol and Listening Configuration

```bash
# Protocol 2 only - SSH v1 is deprecated and insecure
Protocol 2

# Specify which host keys to use (prefer modern Ed25519)
HostKey /etc/ssh/ssh_host_ed25519_key
HostKey /etc/ssh/ssh_host_rsa_key

# Disable compression or delay until after authentication
# Compression can be exploited in CRIME-style attacks
Compression delayed

# Listen on specific interface if possible (more secure than 0.0.0.0)
# ListenAddress 10.0.1.10

# Default port 22 is fine - changing ports is security through obscurity
# Focus on proper authentication and rate limiting instead
Port 22

# Use privilege separation for additional security
UsePrivilegeSeparation sandbox
```

------

> **📝 Note **
>
>  While some guides recommend changing the default port, this provides minimal security benefit. Attackers can easily port-scan your server. Instead, focus on proper authentication, rate limiting, and firewall rules.

------

### Authentication Configuration

```bash
# Disable root login completely
PermitRootLogin no

# Disable password-based authentication
PasswordAuthentication no
PermitEmptyPasswords no
ChallengeResponseAuthentication no

# Enable public key authentication
PubkeyAuthentication yes

# Limit authentication attempts
MaxAuthTries 3
MaxSessions 10

# Time allowed for authentication before disconnect
LoginGraceTime 60

# Disable unused authentication methods
HostbasedAuthentication no
IgnoreRhosts yes
IgnoreUserKnownHosts yes
KerberosAuthentication no
GSSAPIAuthentication no

# Prevent users from setting environment variables
PermitUserEnvironment no
```

### Access Control

```bash
# Use group-based access control (more maintainable than AllowUsers)
AllowGroups sshusers

# Alternative: Restrict by specific users
# AllowUsers alice bob

# Deny specific users if needed
# DenyUsers baduser

# Strict mode checks file permissions
StrictModes yes
```

### Cryptographic Hardening

Use modern, secure cryptographic algorithms:

```bash
# Key Exchange Algorithms - only secure, modern options
KexAlgorithms curve25519-sha256,curve25519-sha256@libssh.org,diffie-hellman-group16-sha512,diffie-hellman-group18-sha512,diffie-hellman-group-exchange-sha256

# Ciphers - authenticated encryption modes only
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com,aes256-ctr,aes192-ctr,aes128-ctr

# Message Authentication Codes - ETM (Encrypt-then-MAC) modes preferred
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com,hmac-sha2-512,hmac-sha2-256

# Host key algorithms - prefer Ed25519 and modern options
HostKeyAlgorithms ssh-ed25519,ssh-ed25519-cert-v01@openssh.com,sk-ssh-ed25519@openssh.com,sk-ssh-ed25519-cert-v01@openssh.com,rsa-sha2-512,rsa-sha2-256
```

### Session Management

```bash
# Disconnect inactive sessions
ClientAliveInterval 300
ClientAliveCountMax 2

# This means: send keepalive every 5 minutes (300 seconds)
# Disconnect after 2 failed keepalive responses (10 minutes total)

# TCP keepalive messages (less reliable than ClientAlive)
TCPKeepAlive yes

# Maximum time to keep connection open
# MaxStartups 10:30:60  # Uncomment if needed
```

### Feature Restrictions

```bash
# Disable X11 forwarding unless specifically needed
X11Forwarding no

# Disable agent forwarding by default (security risk)
AllowAgentForwarding no

# Disable TCP forwarding if not needed
AllowTcpForwarding no

# Disable tunnel device forwarding
PermitTunnel no

# Disable gateway ports
GatewayPorts no

# Disable SSH banner (information disclosure)
DebianBanner no

# Print last login information
PrintLastLog yes

# Use PAM for account management
UsePAM yes
```

### Logging and Monitoring

```bash
# Verbose logging for security auditing
LogLevel VERBOSE

# Log authentication attempts to auth facility
SyslogFacility AUTH
```

### Legal Banner

```bash
# Display a legal warning banner
Banner /etc/ssh/banner.txt
```

Create the banner file:

```bash
sudo nano /etc/ssh/banner.txt
```

**Banner Content Requirements:**

An effective SSH banner should include:

1. **Authorized Access Statement**: Clear declaration that system is for authorized users only
2. **Consent to Monitoring**: Notice that activity may be monitored and recorded
3. **No Expectation of Privacy**: Statement that users have no privacy expectation
4. **Legal Consequences**: Warning about prosecution for unauthorized access
5. **Jurisdictional Information**: Applicable laws and location (if required)

**Example banner content:**

```
****************************************************************************
                            AUTHORIZED ACCESS ONLY

Unauthorized access to this system is forbidden and will be prosecuted by law.
By accessing this system, you agree that your actions may be monitored and
recorded if unauthorized usage is suspected.

You have no expectation of privacy on this system. All activities are logged
and monitored for security and compliance purposes.

Disconnect immediately if you are not an authorized user.

****************************************************************************
```

**Important Notes:**

- Keep banner concise (most terminals display ~24 lines)
- Avoid ASCII art that may not render correctly
- Don't disclose OpenSSH version or system details
- Review banner text with legal counsel for your jurisdiction
- Update banner if legal requirements change

```bash
# Set appropriate permissions
sudo chmod 644 /etc/ssh/banner.txt
sudo chown root:root /etc/ssh/banner.txt
```

------

## Step 3: Validate Configuration

Before restarting SSH, always test the configuration:

```bash
# Test configuration for syntax errors
sudo sshd -t

# If you get no output, the configuration is valid
# If there are errors, they will be displayed
```

Common validation errors:

- **Unknown parameter**: Check spelling and OpenSSH version compatibility
- **Bad configuration option**: Usually a typo or deprecated option
- **Missing required parameter**: Add the required configuration

------

> **✅ Important**
>
>  Only proceed if validation passes without errors.

------

------

## Step 4: SSH Key-Based Authentication

Password authentication is inherently vulnerable. SSH keys provide cryptographic authentication that's resistant to brute-force attacks.

### Generate SSH Keys (On Your Local Machine)

```bash
# Generate Ed25519 key (modern, fast, secure)
ssh-keygen -t ed25519 -a 100 -C "your_email@example.com" -f ~/.ssh/id_ed25519_server

# Parameters explained:
# -t ed25519: Use Ed25519 algorithm (preferred over RSA)
# -a 100: 100 rounds of key derivation function (strengthens passphrase)
# -C "...": Comment for identifying the key
# -f ~/.ssh/id_ed25519_server: Custom filename

# Set a strong passphrase when prompted
```

### Secure Your Private Key

```bash
# Set correct permissions on SSH directory and keys
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_ed25519_server
chmod 644 ~/.ssh/id_ed25519_server.pub

# On your local machine, use ssh-agent to avoid repeated passphrase entry
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519_server
```

### Copy Public Key to Server

```bash
# Method 1: Using ssh-copy-id (recommended)
ssh-copy-id -i ~/.ssh/id_ed25519_server.pub user@server_ip

# Method 2: Manual copy (if ssh-copy-id is unavailable)
cat ~/.ssh/id_ed25519_server.pub | ssh user@server_ip "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

### Verify on Server

```bash
# SSH into the server and verify permissions
ssh user@server_ip

# Check authorized_keys file permissions
ls -la ~/.ssh/authorized_keys
# Should show: -rw------- (600)

# Check SSH directory permissions
ls -ld ~/.ssh
# Should show: drwx------ (700)
```

### Test Key-Based Login

```bash
# Test the key-based login BEFORE disabling password authentication
ssh -i ~/.ssh/id_ed25519_server user@server_ip

# If successful, you should log in without a password prompt
```

------

> **⚠️ Critical**
>
>  Only disable password authentication after confirming key-based login works.

------

### SSH Key Management Best Practices

Proper key management is essential for maintaining security over time:

```bash
# Regular key rotation (recommended every 90-180 days for high-security environments)
# Generate new key
ssh-keygen -t ed25519 -a 100 -C "your_email@example.com" -f ~/.ssh/id_ed25519_new

# Deploy new key to servers
ssh-copy-id -i ~/.ssh/id_ed25519_new.pub user@server_ip

# Test new key works
ssh -i ~/.ssh/id_ed25519_new user@server_ip

# Remove old key from authorized_keys
# On server: edit ~/.ssh/authorized_keys and remove the old key line
```

**Key Lifecycle Management:**

- **Generate**: Always use Ed25519 with strong passphrases
- **Deploy**: Use ssh-copy-id or secure methods
- **Rotate**: Replace keys periodically (90-180 day cycles)
- **Revoke**: Immediately remove compromised keys from all servers
- **Audit**: Regularly review authorized_keys files for unknown keys

```bash
# Audit all authorized_keys across your infrastructure
sudo find /home -name authorized_keys -exec grep -H . {} \;

# Check for keys without comments (harder to track)
grep -v "^#" ~/.ssh/authorized_keys | grep -v "@"
```

**Passphrase Requirements:**

- Minimum 20 characters for SSH key passphrases
- Use a password manager to store passphrases securely
- Never use the same passphrase across multiple keys
- Consider hardware security keys (YubiKey) for critical systems

**Revoking Compromised Keys:**

```bash
# On each server where the key is authorized:
# 1. Edit authorized_keys
sudo nano /home/username/.ssh/authorized_keys

# 2. Remove the compromised key line

# 3. Verify removal
grep "compromised_key_comment" /home/username/.ssh/authorized_keys

# 4. Log the revocation for audit purposes
logger "SSH key revoked for user username - $(date)"
```

------

## Step 5: Create SSH Access Group

Using group-based access control is more maintainable than listing individual users:

```bash
# Create a group for SSH access
sudo groupadd sshusers

# Add your user to the group
sudo usermod -aG sshusers $USER

# Verify group membership
groups $USER

# Verify in /etc/group
grep sshusers /etc/group
```

------

## Step 6: Apply Configuration Changes

```bash
# Restart SSH service to apply changes
sudo systemctl restart sshd

# Verify SSH is running
sudo systemctl status sshd

# Check which port SSH is listening on
sudo ss -tlnp | grep sshd
```

------

> **⚠️ Important**
>
>  Do not close your current SSH session yet. Open a new terminal and test the connection first.

------

------

## Step 7: Firewall Configuration

A properly configured firewall adds an essential defense layer.

### UFW (Ubuntu/Debian)

```bash
# Install UFW if not already installed
sudo apt update && sudo apt install ufw -y

# Default policies: deny incoming, allow outgoing
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow SSH with rate limiting (prevents brute-force)
sudo ufw limit 22/tcp comment 'SSH with rate limiting'

# Rate limiting allows 6 connections from the same IP within 30 seconds
# Additional attempts are blocked

# If you need to allow specific IPs only:
# sudo ufw allow from 203.0.113.10 to any port 22 proto tcp

# Enable firewall
sudo ufw enable

# Check status
sudo ufw status verbose
```

### Firewalld (RHEL/CentOS/Fedora)

```bash
# Install firewalld if not already installed
sudo dnf install firewalld -y
sudo systemctl enable --now firewalld

# Add SSH service
sudo firewall-cmd --permanent --add-service=ssh

# Add rate limiting with rich rules
sudo firewall-cmd --permanent --add-rich-rule='rule service name="ssh" limit value="6/m" accept'

# Reload firewall
sudo firewall-cmd --reload

# Verify configuration
sudo firewall-cmd --list-all
```

------

## Step 8: Fail2Ban - Automated Intrusion Prevention

Fail2Ban monitors logs and automatically blocks IPs after repeated failed authentication attempts.

### Installation

```bash
# Ubuntu/Debian
sudo apt update && sudo apt install fail2ban -y

# RHEL/CentOS/Fedora
sudo dnf install fail2ban -y

# Enable and start the service
sudo systemctl enable --now fail2ban
```

### Configuration

```bash
# Create a local configuration file (overrides defaults)
sudo nano /etc/fail2ban/jail.local
```

Add the following configuration:

```ini
[DEFAULT]
# Ban duration: 24 hours
bantime = 86400

# Time window for counting failures: 10 minutes
findtime = 600

# Number of failures before ban
maxretry = 3

# Email notifications (optional)
# destemail = your_email@example.com
# sender = fail2ban@yourdomain.com
# action = %(action_mwl)s

[sshd]
enabled = true
port = 22
logpath = /var/log/auth.log  # Ubuntu/Debian
# logpath = /var/log/secure  # RHEL/CentOS
maxretry = 3
findtime = 600
bantime = 86400

# Optional: Increase ban time for repeat offenders
[recidive]
enabled = true
logpath = /var/log/fail2ban.log
bantime = 604800  # 1 week
findtime = 86400  # 1 day
maxretry = 3
```

### Manage Fail2Ban

```bash
# Restart Fail2Ban to apply changes
sudo systemctl restart fail2ban

# Check status
sudo systemctl status fail2ban

# View all jails
sudo fail2ban-client status

# View specific jail (e.g., sshd)
sudo fail2ban-client status sshd

# Unban an IP address
sudo fail2ban-client set sshd unbanip 203.0.113.10

# View banned IPs
sudo fail2ban-client get sshd banip
```

------

## Step 9: Two-Factor Authentication (Optional but Recommended)

Adding 2FA provides an additional security layer even if SSH keys are compromised. This section covers the most practical approach for implementing TOTP-based authentication.

### Understanding 2FA for SSH

Two-factor authentication requires two separate methods to verify identity:

1. **Something you have**: Your SSH private key
2. **Something you know**: A time-based code from your authenticator app

This means that even if someone steals your SSH key, they cannot access your server without the second factor.

### Installing the PAM Module

```bash
# Ubuntu/Debian
sudo apt install libpam-google-authenticator -y

# RHEL/CentOS/Fedora
sudo dnf install google-authenticator -y

# Arch Linux
sudo pacman -S libpam-google-authenticator
```

------

> **📝 Note**
>
>  Despite the package name "google-authenticator", this implements the open **TOTP standard (RFC 6238)** and works with any compatible authenticator app.

------

### Choosing an Authenticator App

Select any TOTP-compatible app for your phone or desktop:

**Recommended open source options:**

- **FreeOTP** (Red Hat) - No vendor lock-in, available on iOS and Android
- **Aegis Authenticator** (Android) - Encrypted backups, import/export functionality
- **KeePassXC** (Desktop) - Integrates with password database

**Popular alternatives:**

- **Bitwarden Authenticator** - Cloud backup, cross-platform
- **Authy** - Multi-device sync, cloud backup
- **Microsoft Authenticator** - Enterprise features
- **Google Authenticator** - Simple but lacks backup features

------

> **💡 Important**
>
>  Choose an app with backup/export functionality to avoid lockout if you lose your device.

------

### Configuring 2FA for Your User

```bash
# Run as the user who will log in (NOT as root)
google-authenticator

# Answer the prompts:
# - Do you want tokens to be time-based? YES
# - Update ~/.google_authenticator file? YES
# - Disallow multiple uses of the same token? YES
# - Increase time skew window? YES (allows ±30 seconds for time drift)
# - Enable rate-limiting? NO (Fail2Ban already handles this)
```

After running the command:

1. **Scan the QR code** with your chosen authenticator app
2. **Save the emergency scratch codes** in a secure location (each works once)
3. **Verify** by entering a generated code when prompted

### Configuring SSH for 2FA

#### Step 1: Configure PAM

```bash
# Edit PAM configuration for SSH
sudo nano /etc/pam.d/sshd
```

Add this line **at the top** of the file:

```
auth required pam_google_authenticator.so nullok
```

The `nullok` option allows users without 2FA configured to still log in. Remove it once all users have set up 2FA.

#### Step 2: Update SSH Configuration

```bash
sudo nano /etc/ssh/sshd_config
```

Add or modify these settings:

```bash
# Require both SSH key AND 2FA code
AuthenticationMethods publickey,keyboard-interactive

# Enable keyboard-interactive for 2FA prompts
KbdInteractiveAuthentication yes
ChallengeResponseAuthentication yes

# Ensure PAM is enabled
UsePAM yes
```

#### Step 3: Test and Restart

```bash
# Test SSH configuration for errors
sudo sshd -t

# If no errors, restart SSH
sudo systemctl restart sshd
```

### Testing Your Configuration

------

> **⚠️ Critical**
>
>  Test in a new session before closing your current one!

------

1. Keep your current SSH session **open** (safety measure)
2. Open a **new terminal window**
3. SSH to the server: `ssh user@server`
4. You'll be prompted for:
   - SSH key passphrase (if your key is encrypted)
   - Verification code from your authenticator app
5. Enter the 6-digit code from your app
6. If successful, you're now using 2FA!

Only close your original session after confirming the new authentication flow works.

### Important Security Notes

**Backup and Recovery:**

- Save emergency scratch codes in a secure location (password manager, encrypted file)
- Each scratch code works only once
- Without backup codes or console access, losing your 2FA device means lockout

**Time Synchronization:**

- TOTP codes depend on accurate time
- Ensure your server's time is synchronized: `sudo timedatectl set-ntp true`
- If codes don't work, check time difference between server and device

**Gradual Rollout:**

- Use `nullok` initially to allow gradual user enrollment
- Remove `nullok` after all users have configured 2FA
- Document the process for your team

### Troubleshooting

```bash
# Check if PAM module is configured
grep google-authenticator /etc/pam.d/sshd

# Verify user's 2FA file exists and has correct permissions
ls -la ~/.google_authenticator
# Should show: -rw------- (600)

# Check server time synchronization
timedatectl status

# View authentication logs for errors
sudo tail -f /var/log/auth.log  # Ubuntu/Debian
sudo tail -f /var/log/secure    # RHEL/CentOS
```

### Alternative: Hardware Security Keys

For maximum security, consider hardware tokens like **YubiKey** (requires OpenSSH 8.2+):

```bash
# Generate SSH key on hardware token
ssh-keygen -t ed25519-sk -O resident -f ~/.ssh/id_yubikey

# Deploy to server
ssh-copy-id -i ~/.ssh/id_yubikey.pub user@server
```

Hardware tokens provide phishing-resistant authentication and are recommended for high-security environments. Consider purchasing two tokens (primary and backup).

------

> **⚠️ Warning**
>
>  Always maintain an alternative access method (console access, IPMI, or backup SSH configuration) before enforcing 2FA. Losing your 2FA device without backup codes will lock you out of your server.

------

------

## Step 10: Monitoring and Maintenance

Security is not a one-time configuration—it requires ongoing monitoring and maintenance.

### Log Monitoring

```bash
# Monitor authentication attempts in real-time
sudo tail -f /var/log/auth.log  # Ubuntu/Debian
sudo tail -f /var/log/secure    # RHEL/CentOS

# View recent failed SSH attempts
sudo grep "Failed password" /var/log/auth.log | tail -n 20

# View recent successful logins
sudo grep "Accepted publickey" /var/log/auth.log | tail -n 20

# View current SSH sessions
who
w

# View detailed login history
last | head -n 20
lastlog
```

### Automated Security Auditing

Install and run ssh-audit to verify your SSH configuration:

```bash
# Install ssh-audit
git clone https://github.com/jtesta/ssh-audit.git
cd ssh-audit

# Run audit against your server
./ssh-audit.py localhost

# Or install via package manager
# sudo apt install ssh-audit      # Ubuntu/Debian 22.04+
# ssh-audit localhost
```

The tool will identify:

- Weak key exchange algorithms
- Outdated ciphers
- Vulnerable MAC algorithms
- Configuration recommendations

### Regular Maintenance Tasks

```bash
# Update OpenSSH regularly
sudo apt update && sudo apt upgrade openssh-server -y  # Ubuntu/Debian
sudo dnf upgrade openssh-server -y                     # RHEL/CentOS

# Review Fail2Ban logs weekly
sudo cat /var/log/fail2ban.log | grep "Ban"

# Check for unauthorized SSH keys
sudo find /home -name authorized_keys -ls

# Review user accounts with SSH access
getent group sshusers

# Audit active SSH connections
sudo netstat -tnpa | grep 'ESTABLISHED.*sshd'
```

### Centralized Logging (Production Environments)

For production environments, send SSH logs to a centralized logging system:

```bash
# Configure rsyslog to forward auth logs
sudo nano /etc/rsyslog.d/50-ssh.conf
```

Add:

```
# Forward auth logs to remote syslog server
auth,authpriv.* @@log-server.example.com:514
```

```bash
# Restart rsyslog
sudo systemctl restart rsyslog
```

------

## Step 11: SSH Client Hardening

While server hardening is critical, securing SSH clients is equally important for complete security.

### Client Configuration (~/.ssh/config)

Create or edit your SSH client configuration:

```bash
# Create SSH config directory if it doesn't exist
mkdir -p ~/.ssh
chmod 700 ~/.ssh

# Create/edit client configuration
nano ~/.ssh/config
```

**Recommended client configuration:**

```bash
# Global defaults for all hosts
Host *
    # Hash known_hosts for privacy
    HashKnownHosts yes
    
    # Verify host keys strictly
    StrictHostKeyChecking ask
    
    # Verify host keys via DNS (if DNSSEC available)
    VerifyHostKeyDNS yes
    
    # Disable agent forwarding by default
    ForwardAgent no
    
    # Disable X11 forwarding by default
    ForwardX11 no
    
    # Use only key-based authentication
    PasswordAuthentication no
    PubkeyAuthentication yes
    
    # Prefer modern key types
    HostKeyAlgorithms ssh-ed25519,ssh-ed25519-cert-v01@openssh.com,rsa-sha2-512,rsa-sha2-256
    
    # Use secure ciphers
    Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com
    
    # Use secure MACs
    MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com
    
    # Use secure key exchange
    KexAlgorithms curve25519-sha256,curve25519-sha256@libssh.org,diffie-hellman-group16-sha512
    
    # Connection multiplexing for performance (optional)
    ControlMaster auto
    ControlPath ~/.ssh/sockets/%r@%h-%p
    ControlPersist 10m
    
    # Server alive interval (prevent timeout)
    ServerAliveInterval 60
    ServerAliveCountMax 3

# Example: specific host with agent forwarding enabled
Host trusted-jumphost
    HostName jumphost.example.com
    User admin
    ForwardAgent yes
    IdentityFile ~/.ssh/id_ed25519_jumphost

# Example: host through bastion
Host internal-server
    HostName 10.0.1.50
    User sysadmin
    ProxyJump jumphost.example.com
    IdentityFile ~/.ssh/id_ed25519_internal
```

**Create socket directory for connection multiplexing:**

```bash
mkdir -p ~/.ssh/sockets
chmod 700 ~/.ssh/sockets
```

### Client Security Best Practices

```bash
# Verify server fingerprints before first connection
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub

# Check known_hosts for unknown entries
cat ~/.ssh/known_hosts

# Remove compromised host keys
ssh-keygen -R hostname

# Use specific identity files per connection
ssh -i ~/.ssh/specific_key user@host

# Disable agent forwarding for untrusted hosts
ssh -o ForwardAgent=no user@untrusted-host
```

------

## Step 12: Advanced Access Control with Match Blocks

Match blocks enable fine-grained access control based on user, group, host, or address.

### Conditional Configuration Examples

Add these to `/etc/ssh/sshd_config`:

```bash
# Example 1: Different rules for administrators
Match Group admins
    PermitRootLogin yes
    AllowTcpForwarding yes
    X11Forwarding yes
    MaxSessions 20

# Example 2: Restricted access for developers
Match Group developers
    PermitRootLogin no
    AllowTcpForwarding local
    X11Forwarding yes
    MaxSessions 10
    ForceCommand /usr/local/bin/dev-shell

# Example 3: Internal network gets relaxed security
Match Address 10.0.0.0/8,172.16.0.0/12,192.168.0.0/16
    PasswordAuthentication yes
    ChallengeResponseAuthentication yes

# Example 4: Internet access requires 2FA and key
Match Address !10.0.0.0/8,!172.16.0.0/12,!192.168.0.0/16
    AuthenticationMethods publickey,keyboard-interactive
    PasswordAuthentication no

# Example 5: SFTP-only users
Match Group sftponly
    ForceCommand internal-sftp
    ChrootDirectory /sftp/%u
    AllowTcpForwarding no
    X11Forwarding no

# Example 6: Automated backup user
Match User backup
    PermitRootLogin no
    AllowTcpForwarding no
    X11Forwarding no
    ForceCommand /usr/local/bin/backup-receive
    AuthenticationMethods publickey
```

### Match Block Best Practices

- **Order matters**: More specific Match blocks should come before general ones
- **Testing**: Always test with `sshd -t` after adding Match blocks
- **Documentation**: Comment each Match block explaining its purpose
- **Defaults**: Rules outside Match blocks apply to all connections not matching
- **Negation**: Use `!` to exclude patterns (e.g., `Match Address !10.0.0.0/8`)

------

## Step 13: Automated Compliance Checking

Regular security audits ensure your SSH configuration remains compliant over time.

### SSH-Audit Tool

```bash
# Install ssh-audit
git clone https://github.com/jtesta/ssh-audit.git
cd ssh-audit

# Run audit against your server
./ssh-audit.py localhost

# Or scan a remote server
./ssh-audit.py example.com

# JSON output for automation
./ssh-audit.py -j localhost > ssh-audit-report.json
```

**ssh-audit checks:**

- Weak key exchange algorithms
- Outdated ciphers and MACs
- Deprecated host key types
- CVE vulnerabilities
- OpenSSH version issues

### Lynis Security Auditing

```bash
# Install Lynis
sudo apt install lynis -y  # Ubuntu/Debian
sudo dnf install lynis -y  # RHEL/CentOS

# Run SSH-specific tests
sudo lynis audit system --tests SSH

# Full system audit including SSH
sudo lynis audit system

# View suggestions
sudo cat /var/log/lynis.log
```

### OpenSCAP Compliance Scanning

```bash
# Install OpenSCAP (Ubuntu/Debian)
sudo apt install libopenscap8 -y

# Install SCAP Security Guide
sudo apt install ssg-debian -y  # Debian
sudo apt install scap-security-guide -y  # RHEL/CentOS

# Scan for CIS compliance
sudo oscap xccdf eval \
    --profile xccdf_org.ssgproject.content_profile_cis \
    --results-arf /tmp/arf.xml \
    --report /tmp/report.html \
    /usr/share/xml/scap/ssg/content/ssg-debian11-ds.xml

# View the HTML report
firefox /tmp/report.html
```

### Automated Monitoring Script

Create a daily SSH security check:

```bash
# Create monitoring script
sudo nano /usr/local/bin/ssh-security-check.sh
```

```bash
#!/bin/bash
# SSH Security Daily Check

REPORT_FILE="/var/log/ssh-security-$(date +%F).log"

{
    echo "=== SSH Security Check - $(date) ==="
    echo
    
    # Check for weak permissions
    echo "Checking SSH directory permissions..."
    find /home -name .ssh -exec ls -ld {} \; | awk '$1 !~ /^drwx------/ {print "WARNING: Weak permissions on " $NF}'
    
    # Check authorized_keys
    echo "Checking authorized_keys..."
    find /home -name authorized_keys -exec ls -l {} \; | awk '$1 !~ /^-rw-------/ {print "WARNING: Weak permissions on " $NF}'
    
    # Count SSH keys
    echo "SSH key count per user:"
    for dir in /home/*/.ssh; do
        user=$(echo "$dir" | cut -d'/' -f3)
        count=$(grep -c "^ssh-" "$dir/authorized_keys" 2>/dev/null || echo "0")
        echo "  $user: $count keys"
    done
    
    # Check for root login attempts
    echo "Recent failed root login attempts:"
    grep "Failed password for root" /var/log/auth.log | tail -n 5
    
    # Check for successful logins
    echo "Recent successful SSH logins:"
    grep "Accepted publickey" /var/log/auth.log | tail -n 10
    
    # Test SSH configuration
    echo "Testing SSH configuration..."
    sshd -t && echo "SSH configuration is valid" || echo "ERROR: SSH configuration has errors"
    
    # Check SSH service status
    echo "SSH service status:"
    systemctl is-active sshd
    
    echo "=== End of Report ==="
} > "$REPORT_FILE"

# Send report via email if mailx is configured
# mail -s "SSH Security Report - $(hostname)" admin@example.com < "$REPORT_FILE"

# Keep only last 30 days of reports
find /var/log -name "ssh-security-*.log" -mtime +30 -delete
```

```bash
# Make executable
sudo chmod +x /usr/local/bin/ssh-security-check.sh

# Add to crontab (run daily at 6 AM)
sudo crontab -e
# Add line:
# 0 6 * * * /usr/local/bin/ssh-security-check.sh
```

------

## Step 14: Advanced Security Measures

### SSH Certificates vs. Public Keys

For larger infrastructures, SSH certificates provide centralized key management:

**Benefits:**

- Centralized revocation
- Time-limited access
- Simplified key distribution
- Principal-based access control

**Implementation (overview):**

```bash
# Generate a CA key (do this on a secure, offline machine)
ssh-keygen -t ed25519 -f ssh_ca -C "SSH CA"

# Sign a user's public key
ssh-keygen -s ssh_ca -I user@example.com -n admin,developer -V +1w id_ed25519.pub

# On the server, trust the CA
echo "@cert-authority * $(cat ssh_ca.pub)" | sudo tee -a /etc/ssh/ca.pub

# Update sshd_config
TrustedUserCAKeys /etc/ssh/ca.pub
```

### Bastion Host Architecture

For production environments, never expose SSH directly to the internet:

```
Internet → Bastion Host → Private Servers
```

- Bastion host has extreme SSH hardening
- Private servers only accept connections from bastion
- All access is logged and audited

### Port Knocking (Optional)

Port knocking hides SSH behind a sequence of connection attempts:

```bash
# Install knockd
sudo apt install knockd -y

# Configure knocking sequence
sudo nano /etc/knockd.conf
```

Example configuration:

```
[openSSH]
sequence = 7000,8000,9000
seq_timeout = 5
command = /sbin/iptables -A INPUT -s %IP% -p tcp --dport 22 -j ACCEPT
tcpflags = syn

[closeSSH]
sequence = 9000,8000,7000
seq_timeout = 5
command = /sbin/iptables -D INPUT -s %IP% -p tcp --dport 22 -j ACCEPT
tcpflags = syn
```

### VPN-Based SSH Access

The most secure approach for production:

- Set up WireGuard or OpenVPN
- Only allow SSH over VPN interface
- Public SSH port completely closed

```bash
# In sshd_config, listen only on VPN interface
ListenAddress 10.8.0.1
```

------

## Troubleshooting Common Issues

### Issue: Locked Out After Configuration Changes

**Solution:**

```bash
# If you have console access:
# 1. Boot into single-user mode or rescue mode
# 2. Restore the backup configuration:
sudo cp /etc/ssh/sshd_config.backup.YYYY-MM-DD /etc/ssh/sshd_config
sudo systemctl restart sshd

# Always keep a session open when testing changes
```

### Issue: SSH Keys Not Working

**Checklist:**

```bash
# On server:
ls -ld ~/.ssh              # Should be 700
ls -l ~/.ssh/authorized_keys  # Should be 600

# Check SELinux context (RHEL/CentOS)
restorecon -R -v ~/.ssh

# Verify key in authorized_keys
cat ~/.ssh/authorized_keys

# Check SSH logs
sudo tail -f /var/log/auth.log
# Look for "Permission denied" or "Authentication refused"
```

### Issue: 2FA Not Prompting

**Solution:**

```bash
# Verify PAM is enabled
grep "UsePAM yes" /etc/ssh/sshd_config

# Check PAM configuration
cat /etc/pam.d/sshd | grep google-authenticator

# Verify AuthenticationMethods
grep "AuthenticationMethods" /etc/ssh/sshd_config

# Check Google Authenticator config exists
ls -la ~/.google_authenticator
```

### Issue: Fail2Ban Not Blocking

**Solution:**

```bash
# Check Fail2Ban is running
sudo systemctl status fail2ban

# Verify jail is enabled
sudo fail2ban-client status

# Check log path is correct
sudo fail2ban-client get sshd logpath

# Test regex pattern
sudo fail2ban-regex /var/log/auth.log /etc/fail2ban/filter.d/sshd.conf
```

------

## Performance Considerations

SSH hardening typically has minimal performance impact, but be aware:

- **Strong KDF rounds** (`-a 100`): Slight delay when generating keys (one-time cost)
- **Modern ciphers**: ChaCha20-Poly1305 is faster than AES on systems without AES-NI
- **Verbose logging**: Increases disk I/O slightly
- **Fail2Ban**: Minimal CPU overhead; uses minimal memory

For high-traffic SSH servers:

```bash
# Increase MaxStartups if you have many concurrent connections
MaxStartups 10:30:100

# Consider connection multiplexing on client side
# Add to ~/.ssh/config:
ControlMaster auto
ControlPath ~/.ssh/sockets/%r@%h-%p
ControlPersist 600
```

------

## Compliance and Standards

This guide aligns with:

- **CIS Benchmarks**: Level 1 & Level 2 SSH hardening guidelines
- **NIST SP 800-53**: Access Control (AC) and Identification & Authentication (IA) requirements
- **PCI DSS**: Requirement 2.3 - Encrypt all non-console administrative access
- **HIPAA**: Technical safeguards for access control
- **Mozilla SSH Guidelines**: Modern cryptographic standards

------

## Security Checklist

### Core Configuration

-  Backed up original `sshd_config`
-  Set `Protocol 2` explicitly
-  Configured modern `HostKey` algorithms (Ed25519 preferred)
-  Set `Compression delayed` or disabled
-  Disabled root login (`PermitRootLogin no`)
-  Disabled password authentication
-  Configured modern cryptographic algorithms (KexAlgorithms, Ciphers, MACs)
-  Set session timeouts (ClientAliveInterval/CountMax)
-  Disabled unnecessary features (X11, agent forwarding, TCP forwarding)

### Access Control

-  Created SSH access group
-  Implemented group-based access control (`AllowGroups`)
-  Added authorized users to SSH group
-  Set `StrictModes yes`

### Authentication

-  Generated Ed25519 SSH keys with strong passphrases
-  Set correct permissions on keys and directories
-  Deployed public keys to servers
-  Tested key-based authentication
-  (Optional) Configured 2FA/MFA

### Defense Layer

-  Configured firewall with rate limiting
-  Installed and configured Fail2Ban
-  Set appropriate ban times and retry limits

### Monitoring

-  Set `LogLevel VERBOSE`
-  Configured centralized logging (production)
-  Created legal banner
-  Set up automated security checks

### Client Security

-  Created hardened `~/.ssh/config`
-  Configured connection multiplexing
-  Set secure client-side cryptography

### Testing and Validation

-  Tested configuration with `sshd -t`
-  Ran ssh-audit scan
-  Tested SSH login with new configuration
-  Verified logging works
-  Tested Fail2Ban blocking

### Documentation

-  Documented all configuration changes
-  Created runbook for common issues
-  Scheduled regular security audits
-  Configured backup access method

------

## Conclusion

SSH hardening is a critical component of server security. By implementing these practices, you've established:

✓ **Strong authentication** through key-based access and optional 2FA
 ✓ **Modern cryptography** resistant to current attacks
 ✓ **Defense-in-depth** with firewalls and intrusion prevention
 ✓ **Comprehensive monitoring** for detecting unauthorized access
 ✓ **Maintainable security** through group-based access control

Remember that security is an ongoing process. Regularly review logs, update software, audit your configuration with tools like ssh-audit, and stay informed about emerging threats.