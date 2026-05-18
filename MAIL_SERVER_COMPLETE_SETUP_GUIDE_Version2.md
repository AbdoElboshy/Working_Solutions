# Mail Server Complete Setup Guide
## Exim4 + DKIM + Fail2Ban + IPv6 + SSL Auto-Renewal

> Complete documentation for setting up and hardening a mail server  
> Tested on Ubuntu 22.04 LTS  
> Compatible with FastPanel

---

## Table of Contents

1. [Prerequisites & Questions to Ask Client](#1-prerequisites--questions-to-ask-client)
2. [Understanding Key Concepts](#2-understanding-key-concepts)
3. [Setting Up Test Server on AWS](#3-setting-up-test-server-on-aws)
4. [Task 1 - Configure IPv6](#4-task-1---configure-ipv6)
5. [Task 2 - Fix Webmail IPv6 Issues](#5-task-2---fix-webmail-ipv6-issues)
6. [Task 3 - Change Hostname](#6-task-3---change-hostname)
7. [Task 4 - Harden Empty Sender in Exim4](#7-task-4---harden-empty-sender-in-exim4)
8. [Task 5 - Implement DKIM](#8-task-5---implement-dkim)
9. [Task 6 - Fix Fail2Ban Jails](#9-task-6---fix-fail2ban-jails)
10. [SSL Certificate Auto-Renewal](#10-ssl-certificate-auto-renewal)
11. [Testing & Verification](#11-testing--verification)
12. [Troubleshooting](#12-troubleshooting)
13. [Quick Reference](#quick-reference)
14. [How to Upload to GitHub](#how-to-upload-to-github)

---

## 1. Prerequisites & Questions to Ask Client

### Send this message to the client before starting:

Hello!

Before I start working, I need some information to complete the tasks correctly. Could you please provide:

- What is your domain name? (example: `yourdomain.com`)
- What is your VPS provider? (Hetzner, OVH, Vultr, etc.)
- I need this to get the IPv6 address details:
  - Can you share the IPv6 address from your VPS provider panel? (if you already have one assigned)
- What should the new hostname be?  
  (usually `mail.yourdomain.com` or `server.yourdomain.com`)
- Who manages your DNS records?  
  (Cloudflare, cPanel, FastPanel, domain registrar?)
- What is the webmail software you use?  
  (Roundcube, Rainloop, or something else?)
- Can you give me SSH root access to the server?
  - Server IP address
  - Root password OR SSH key
- What SSL certificate are you currently using?  
  (Let's Encrypt, paid certificate, or none?)
- What is the exact error when webmail is not accessible?  
  (screenshot would be helpful)
- Is FastPanel installed? What is the FastPanel URL and login?

Thank you!

### What "IPv6 from ISP" means

When client says "I got IPv6 from ISP" it means:

- VPS provider already **assigned** an IPv6 address
- The address exists but is **not configured on the server**
- We need to activate it in the network config
- No need to request a new one

---

## 2. Understanding Key Concepts

### What is a VPS?

VPS = Virtual Private Server

- A computer in a datacenter you control remotely
- You connect to it using SSH
- Think of it as a remote computer

### What is IPv4 vs IPv6?

**IPv4:** `192.168.1.100`

- Old format
- Only ~4 billion addresses
- Running out globally

**IPv6:** `2001:db8:85a3::8a2e:370:7334`

- New format
- 340 trillion trillion trillion addresses
- Internet is moving to this

### What is Exim4?

Exim4 = Email server software

- Handles sending and receiving emails
- Like a post office on your server
- Config files in: `/etc/exim4/`

### What is DKIM?

DKIM = DomainKeys Identified Mail

- Digital signature on your emails
- Proves email really came from your domain
- Without it = emails go to spam

How it works:

- Private Key (on server) → signs every email
- Public Key (in DNS) → receivers verify signature
- If match = trusted; if no match = spam

### What is Fail2Ban?

Fail2Ban = Automatic IP ban system

- Watches server log files
- Counts failed attempts per IP
- After X failures = bans that IP in firewall
- Like a bouncer at a club

Terms:

- Jail = rules for one specific service
- `exim-spam` jail = watches for spam attempts
- `exim-auth` jail = watches for failed logins

### What is FastPanel?

FastPanel = Russian web hosting control panel

- Like cPanel but different
- Manages websites, email, databases, SSL
- Has its own Exim4 configuration
- Always use FastPanel interface when possible, otherwise FastPanel might overwrite your changes

### What is SSL?

SSL = Secure Socket Layer

- The padlock in your browser
- Encrypts connection between user and server
- Certificate expires (Let's Encrypt = every 90 days)
- We set up auto-renewal so it never expires

---

## 3. Setting Up Test Server on AWS

### Why test first?

- Never practice on client's server
- Always test on your own server first
- AWS has free tier = no cost for testing

### Create AWS Account

- Go to aws.amazon.com
- Create free account
- Free tier = 12 months free
- `t2.micro` instance = free tier eligible (verify in your AWS region)

### Launch EC2 Instance

In AWS Console:

1. Search **EC2**
2. Click **Launch Instance**

Settings:

- Name: `mail-test-server`
- OS: Ubuntu 22.04 LTS
- Instance Type: `t2.micro`
- Key pair: Create new
  - Name: `my-test-key`
  - Type: RSA
  - Format: `.pem`
- Storage: 8GB default
- Click **Launch Instance**

Save the `.pem` key file (you need it).

### Connect to AWS Server

**Windows (PuTTY):**

1. Download PuTTY from putty.org
2. Open PuTTYgen
3. Load `.pem`
4. Save as `.ppk`
5. Open PuTTY
   - Host: `ubuntu@YOUR_AWS_IP`
   - Connection > SSH > Auth > Browse > select `.ppk`
6. Click Open

**Mac/Linux:**

```bash
# Fix key permissions (required by AWS)
chmod 400 my-test-key.pem

# Connect
ssh -i my-test-key.pem ubuntu@YOUR_AWS_IP

# Switch to root
sudo su -
```

### Install Required Software

```bash
# Update system
apt update && apt upgrade -y

# Install everything needed
apt install -y \
  exim4 \
  fail2ban \
  certbot \
  python3-certbot-apache \
  dnsutils \
  net-tools \
  curl \
  nano \
  openssl \
  mailutils \
  iptables-persistent
```

---

## 4. Task 1 - Configure IPv6

### Understanding the issue

ISP gave IPv6 address = address exists but not configured  
Like having a phone number but no phone connected to it.

We need to configure the server to use that address.

### Step 1: Check current status

```bash
# Show all IP addresses
ip addr show

# Show only IPv6
ip -6 addr show

# Check IPv6 routing
ip -6 route show
```

Output explained:

- `inet6 ::1/128 scope host` = localhost only (no real IPv6)
- `inet6 2001:db8::1/64 scope global` = real IPv6 configured

### Step 2: Check OS and network manager

```bash
# Check OS version
cat /etc/os-release | grep -E "NAME|VERSION"

# Check for netplan (Ubuntu 18.04+)
ls /etc/netplan/

# Check current netplan config
cat /etc/netplan/*.yaml
```

### Step 3: AWS Specific - Enable IPv6 in VPC First

> AWS users must do this first.

In AWS Console:

1. **VPC Settings**
   - VPC service → Your VPCs
   - Select your VPC
   - Actions → Edit VPC settings
   - Enable IPv6 CIDR
   - Save

2. **Subnet Settings**
   - Subnets → select your subnet
   - Actions → Edit subnet settings
   - Enable IPv6
   - Save

3. **Route Table**
   - Route Tables → select your route table
   - Routes tab → Edit routes
   - Add: Destination `::/0` → Target: Internet Gateway (igw-...)
   - Save

4. **Security Group**
   - EC2 → Security Groups → select your SG
   - Edit inbound rules
   - Add rule: All ICMP IPv6, Source `::/0`
   - Add rule: All traffic, Source `::/0` (testing only)
   - Save rules

5. **Assign IPv6 to Instance**
   - EC2 → Instances → select instance
   - Actions → Networking → Manage IP addresses
   - Under IPv6, click Assign new IP
   - Auto-assign
   - Save

### Step 4: Disable cloud-init (AWS specific)

```bash
# AWS cloud-init overwrites netplan on reboot
# This stops that from happening
echo "network: {config: disabled}" > \
  /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg

# Verify
cat /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
```

### Step 5: Configure Netplan

```bash
# Backup current config first
cp /etc/netplan/50-cloud-init.yaml \
   /etc/netplan/50-cloud-init.yaml.backup

# Edit config
nano /etc/netplan/50-cloud-init.yaml
```

Replace entire content with:

```yaml
network:
  version: 2
  ethernets:
    ens5:                    # Interface name (check yours with: ip addr show)
      dhcp4: true            # Keep IPv4 automatic
      dhcp6: false           # Manual IPv6 config
      accept-ra: false       # Don't auto-configure from router

      addresses:
        - "YOUR_IPv6_ADDRESS/64"   # Replace with your actual IPv6

      routes:
        - to: ::/0               # Default IPv6 route (all IPv6 traffic)
          via: fe80::1           # Gateway (check with: ip -6 route show)
          on-link: true          # Gateway is directly connected

      nameservers:
        addresses:
          - 8.8.8.8              # Google DNS IPv4
          - 1.1.1.1              # Cloudflare DNS IPv4
          - 2001:4860:4860::8888 # Google DNS IPv6
          - 2606:4700:4700::1111 # Cloudflare DNS IPv6
```

Apply:

```bash
# Validate config before applying
netplan generate

# Apply config
netplan apply

# If something breaks, restore backup:
cp /etc/netplan/50-cloud-init.yaml.backup \
   /etc/netplan/50-cloud-init.yaml
netplan apply
```

### Step 6: Verify IPv6 works

```bash
# Test IPv6 connectivity
ping6 google.com -c 4
```

Common IPv6 issues:

| Problem | Cause | Fix |
|---|---|---|
| Address unreachable | AWS Security Group blocking | Add IPv6 rules to Security Group |
| Network unreachable | No default IPv6 route | Add `::/0` route in Route Table |
| `ping6: connect: Permission denied` | IPv6 disabled | `sysctl -w net.ipv6.conf.all.disable_ipv6=0` |
| Address shows but no internet | Wrong gateway | Check `ip -6 route show` for correct gateway |

---

## 5. Task 2 - Fix Webmail IPv6 Issues

### Why IPv6 breaks webmail

Before IPv6:

- Browser → `webmail.domain.com` → DNS gives IPv4 → works

After IPv6 (misconfigured):

- Browser → `webmail.domain.com` → DNS gives IPv6 → server not listening → error

### Diagnose the issue

```bash
# Check what is listening on which ports
ss -tlnp
```

Port meanings:

- 80 = HTTP
- 443 = HTTPS
- 2096 = FastPanel/cPanel webmail HTTPS
- 2095 = FastPanel/cPanel webmail HTTP

Output explained:

- `0.0.0.0:443` = listening on IPv4 only
- `:::443` = listening on both IPv4 and IPv6

```bash
# Check DNS records
# A record = IPv4
# AAAA record = IPv6
dig A webmail.yourdomain.com
dig AAAA webmail.yourdomain.com

# Test webmail over IPv4
curl -4 -I https://webmail.yourdomain.com

# Test webmail over IPv6
curl -6 -I https://webmail.yourdomain.com
```

```bash
# Check IPv6 firewall
ip6tables -L -n -v
```

### Fix options

**Option 1 - Allow IPv6 in firewall:**

```bash
# Allow web traffic on IPv6
ip6tables -A INPUT -p tcp --dport 80 -j ACCEPT
ip6tables -A INPUT -p tcp --dport 443 -j ACCEPT
ip6tables -A INPUT -p tcp --dport 2096 -j ACCEPT
ip6tables -A INPUT -p tcp --dport 2095 -j ACCEPT

# Save rules (survive reboot)
netfilter-persistent save
```

**Option 2 - Remove IPv6 DNS record for webmail:**

If IPv6 is causing problems:

1. Go to DNS provider
2. Delete the AAAA record for `webmail.yourdomain.com`
3. This forces webmail to use IPv4 only

---

## 6. Task 3 - Change Hostname

### Why hostname matters for email

Hostname = server's name (e.g. `mail.yourdomain.com`)

Email servers check:

1. Your server says: "Hello I am `mail.yourdomain.com`"
2. Receiving server looks up `mail.yourdomain.com` → gets your IP
3. Also checks reverse: your IP → should give `mail.yourdomain.com`
4. If these don't match = emails go to spam

PTR Record = Reverse DNS

- Normal: `mail.yourdomain.com` → `1.2.3.4` (set in DNS)
- Reverse: `1.2.3.4` → `mail.yourdomain.com` (set in VPS provider panel)

### Change hostname

```bash
# Check current hostname
hostname
hostname -f
hostnamectl

# Change hostname
hostnamectl set-hostname mail.yourdomain.com

# Verify
hostname -f
```

```bash
# Find your IPs
curl -4 ifconfig.me  # Your IPv4
curl -6 ifconfig.me  # Your IPv6

# Update hosts file
nano /etc/hosts
```

Example `/etc/hosts`:

```text
127.0.0.1     localhost
127.0.1.1     mail.yourdomain.com mail
YOUR_IPv4     mail.yourdomain.com mail
YOUR_IPv6     mail.yourdomain.com mail
```

```bash
# Update Exim config
nano /etc/exim4/update-exim4.conf.conf
```

Change:

```text
dc_other_hostnames='mail.yourdomain.com'
dc_readhost='mail.yourdomain.com'
```

Apply:

```bash
update-exim4.conf
systemctl restart exim4

# Verify
hostname -f
exim4 -bV | grep "Exim version"
```

### Set PTR Record (Reverse DNS)

This is done in your VPS provider's control panel, not in your domain DNS.

Examples:

- Hetzner: Servers > Server > Networking > Reverse DNS
- OVH: Bare Metal Cloud > IP > Gear icon > Modify reverse
- Vultr: Products > Instances > IPv4 > Reverse DNS
- AWS: Contact AWS support OR use Elastic IP

Set PTR for:

- IPv4: `1.2.3.4` → `mail.yourdomain.com`
- IPv6: `2001:db8::1` → `mail.yourdomain.com`

---

## 7. Task 4 - Harden Empty Sender in Exim4

### What is empty sender?

Normal email:

- `MAIL FROM: john@example.com`

Bounce message:

- `MAIL FROM: <>` (empty)

Empty sender is legitimate for bounce messages, but spammers abuse it to:

- Send spam anonymously
- Send to thousands of recipients at once
- Create email loops

### Understanding the ACL rules

ACL = Access Control List  
Rules that Exim checks for every incoming email.

### Backup before editing

```bash
# Always backup first!
cp /etc/exim4/exim4.conf.template \
   /etc/exim4/exim4.conf.template.backup.$(date +%Y%m%d)
```

### Add hardening rules

```bash
nano /etc/exim4/exim4.conf.template
```

Find `acl_check_rcpt:` and add these rules:

```text
# Rule 1: Empty sender can only have ONE recipient
# Real bounces always go to exactly one address
# Spammers send to many recipients at once
deny
  message   = Bounce messages cannot have multiple recipients
  senders   = :
  condition = ${if > {$recipients_count}{0}{yes}{no}}

# Rule 2: Rate limit - max 5 bounces per hour per IP
# Legitimate servers don't send floods of bounces
deny
  message   = Too many bounce messages from your IP
  senders   = :
  ratelimit = 5 / 1h / strict / $sender_host_address

# Rule 3: Require proper HELO/EHLO greeting
# Spammers often skip this required step
deny
  message   = Proper HELO/EHLO identification required
  senders   = :
  condition = ${if eq{$sender_helo_name}{}{yes}{no}}

# Rule 4: Require reverse DNS
# Legitimate servers always have reverse DNS (PTR record)
# Spam servers usually don't bother
deny
  message   = No reverse DNS found for your IP address
  senders   = :
  !verify   = reverse_host_lookup

# ================================================
# BLOCK ALL EMPTY SENDER COMPLETELY
# ================================================
deny
message = Rejected: Empty sender not accepted on this server
  senders = :

        # (your existing rules below this)
```

### Apply and verify

```bash
# Test config for syntax errors
exim4 -bV

# Apply
update-exim4.conf

# Restart
systemctl restart exim4

# Check running properly
systemctl status exim4
```

---

## 8. Task 5 - Implement DKIM

### How DKIM works (simple)

Think of it like a wax seal on a letter:

1. You have a unique private seal (private key) - secret
2. You stamp every letter before sending
3. Recipients have a way to verify your seal (public key in DNS)
4. If seal matches = email is genuine
5. If seal is broken/fake = email goes to spam

- Private Key: stays on your server forever - never share it
- Public Key: goes in your DNS as a TXT record - everyone can see it

### Step 1: Generate keys

```bash
# Create directory for DKIM keys
mkdir -p /etc/exim4/dkim

# Generate private key (2048 bits)
openssl genrsa \
  -out /etc/exim4/dkim/yourdomain.com.key \
  2048

# Extract public key from private key
openssl rsa \
  -in /etc/exim4/dkim/yourdomain.com.key \
  -pubout \
  -out /etc/exim4/dkim/yourdomain.com.pub

# Set correct permissions
chown root:Debian-exim /etc/exim4/dkim/yourdomain.com.key
chmod 640 /etc/exim4/dkim/yourdomain.com.key

# Verify permissions
ls -la /etc/exim4/dkim/
```

### Step 2: Get public key for DNS

```bash
# Extract key in DNS-ready format
# Removes header/footer lines and joins into one line
cat /etc/exim4/dkim/yourdomain.com.pub \
  | grep -v "^-" \
  | tr -d '\n' \
  && echo ""
```

Copy the entire output for DNS.

### Step 3: Add DNS records

**DKIM Record:**

- Type: TXT
- Name: `dkim._domainkey`
- Value: `v=DKIM1; k=rsa; p=YOUR_PUBLIC_KEY_HERE`

Full name: `dkim._domainkey.yourdomain.com`

**SPF Record (add this too):**

- Type: TXT
- Name: `@`
- Value: `v=spf1 ip4:YOUR_IPv4 ip6:YOUR_IPv6 -all`

**DMARC Record (add this too):**

- Type: TXT
- Name: `_dmarc`
- Value: `v=DMARC1; p=none; rua=mailto:postmaster@yourdomain.com`

Start with `p=none` (monitoring only), then:

- `p=quarantine` after confirming emails work
- `p=reject` for maximum security

### Step 4: Configure Exim to sign emails

```bash
nano /etc/exim4/exim4.conf.template
```

Find `remote_smtp:` transport section and add:

```text
remote_smtp:
  driver = smtp

  # DKIM signing settings:

  # Which domain to sign as
  # Automatically gets domain from From: email header
  dkim_domain = ${lc:${domain:$h_from:}}

  # Selector = prefix before ._domainkey in DNS
  # Our DNS record: dkim._domainkey.domain.com
  # So selector = dkim
  dkim_selector = dkim

  # Path to private key
  # Automatically loads the right key file for each domain
  dkim_private_key = /etc/exim4/dkim/${lc:${domain:$h_from:}}.key

  # How to prepare email for signing
  # relaxed = allow minor changes (recommended)
  dkim_canon = relaxed

  # 0 = don't fail if signing doesn't work
  # 1 = require signing (strict)
  dkim_strict = 0
```

### Step 5: Apply and test

```bash
# Check config
exim4 -bV

# Apply and restart
update-exim4.conf
systemctl restart exim4

# Check DNS propagation (wait 5-10 min after adding record)
dig TXT dkim._domainkey.yourdomain.com

# Send test email
# 1. Go to https://www.mail-tester.com
# 2. Copy the test email address
# 3. Send email to it:
echo "DKIM Test Email" | mail -s "DKIM Test" test-xxxxx@srv1.mail-tester.com
```

Then check your score on mail-tester.com (DKIM should pass).

---

## 9. Task 6 - Fix Fail2Ban Jails

### How Fail2Ban works

1. Someone tries to login to email server
2. Wrong password
3. Exim writes to log: `"Authentication failed from [1.2.3.4]"`
4. Fail2Ban reads log
5. Counts failures
6. Adds firewall rule: block IP
7. Traffic from that IP is dropped
8. After ban time expires, rule is removed

### Why jails fail

- Log file missing or wrong path
- Filter pattern doesn't match log format
- Backend mismatch
- Jail not enabled (`enabled = false`)

### Step 1: Diagnose

```bash
# Check Fail2Ban is running
systemctl status fail2ban

# List all jails
fail2ban-client status

# Check specific jails
fail2ban-client status exim-spam
fail2ban-client status exim-auth

# Check Fail2Ban log for errors
tail -50 /var/log/fail2ban.log
grep "ERROR" /var/log/fail2ban.log | tail -20
```

```bash
# Check if log file exists
ls -la /var/log/exim4/
tail -20 /var/log/exim4/mainlog

# Look at actual log format
grep -i "auth" /var/log/exim4/mainlog | tail -10
grep -i "fail" /var/log/exim4/mainlog | tail -10
grep -i "reject" /var/log/exim4/mainlog | tail -10
grep -i "spam" /var/log/exim4/mainlog | tail -10
```

```bash
# Check filter files exist
ls /etc/fail2ban/filter.d/ | grep exim

# Test if filter matches logs
fail2ban-regex /var/log/exim4/mainlog /etc/fail2ban/filter.d/exim.conf
```

### Step 2: Configure jails

```bash
# Backup first
cp /etc/fail2ban/jail.local \
   /etc/fail2ban/jail.local.backup.$(date +%Y%m%d) 2>/dev/null

# Edit jail config
nano /etc/fail2ban/jail.local
```

Example `jail.local`:

```ini
[DEFAULT]
bantime  = 3600
findtime = 600
maxretry = 5
backend = auto
ignoreip = 127.0.0.1/8 ::1

[exim]
enabled  = true
port     = smtp,465,587
filter   = exim
logpath  = /var/log/exim4/mainlog
maxretry = 5
bantime  = 3600

[exim-spam]
enabled  = true
port     = smtp,465,587
filter   = exim-spam
logpath  = /var/log/exim4/mainlog
maxretry = 3
bantime  = 86400
```

### Step 3: Create/fix filter files

Create `exim-spam` filter:

```bash
nano /etc/fail2ban/filter.d/exim-spam.conf
```

```ini
[Definition]
failregex = ^.*rejected after DATA:.*\[<HOST>\].*$
            ^.*spam score.*\[<HOST>\].*$
            ^.*Message rejected.*spam.*\[<HOST>\].*$
            ^.*\[<HOST>\].*spam detected.*$

ignoreregex =
```

Check/fix Exim auth filter:

```bash
nano /etc/fail2ban/filter.d/exim.conf
```

```ini
[Definition]
failregex = ^.*authenticator failed for .* \[<HOST>\].*$
            ^.*\[<HOST>\] F=.*rejected.*$
            ^.*login auth.*failed.*\[<HOST>\].*$
            ^.*535.*\[<HOST>\].*$

ignoreregex =
```

Test filters:

```bash
fail2ban-regex /var/log/exim4/mainlog \
  /etc/fail2ban/filter.d/exim.conf

fail2ban-regex /var/log/exim4/mainlog \
  /etc/fail2ban/filter.d/exim-spam.conf
```

### Step 4: Apply and verify

```bash
systemctl restart fail2ban
sleep 5

fail2ban-client status
fail2ban-client status exim
fail2ban-client status exim-spam

tail -f /var/log/fail2ban.log
```

Healthy jail output example:

```text
Status for the jail: exim-spam
|- Filter
|  |- Currently failed: 2
|  |- Total failed: 45
|  `- File list: /var/log/exim4/mainlog
`- Actions
   |- Currently banned: 1
   |- Total banned: 12
   `- Banned IP list: 192.168.1.100
```

---

## 10. SSL Certificate Auto-Renewal

### What is SSL and why auto-renew?

SSL = secure padlock in browser.

Let's Encrypt SSL:

- Free
- Expires every 90 days
- Must be renewed before expiry
- If expired = browser shows scary warning
- Visitors leave = bad for business

Auto-renewal = automatically renews before expiry  
= no warnings ever.

### Install Certbot

```bash
apt install -y certbot python3-certbot-apache python3-certbot-nginx
```

### Get SSL certificate

**For Apache:**

```bash
certbot --apache \
  -d yourdomain.com \
  -d www.yourdomain.com \
  -d mail.yourdomain.com
```

Certbot asks:

- Email: your@email.com
- Terms: A
- Share email: N
- Redirect HTTP to HTTPS: 2 (yes)

**For Nginx:**

```bash
certbot --nginx \
  -d yourdomain.com \
  -d www.yourdomain.com \
  -d mail.yourdomain.com
```

**For FastPanel:**

```bash
# Method 1: Through FastPanel interface (recommended)
# Login to FastPanel
# Sites > Click domain > SSL > Let's Encrypt > Enable Auto-renewal

# Method 2: Webroot method (command line)
certbot certonly --webroot \
  -w /var/www/yourdomain.com \
  -d yourdomain.com \
  -d mail.yourdomain.com \
  --email your@email.com \
  --agree-tos \
  --no-eff-email
```

### Verify SSL certificate

```bash
# List all certificates
certbot certificates

# Check certificate expiry
echo | openssl s_client -connect yourdomain.com:443 2>/dev/null \
  | openssl x509 -noout -dates

# Check certificate details
openssl x509 -in /etc/letsencrypt/live/yourdomain.com/fullchain.pem \
  -text -noout | grep -E "Subject:|Not After"
```

### Set up auto-renewal

```bash
# Check if certbot timer is already set up
systemctl status certbot.timer

# Enable if not active
systemctl enable certbot.timer
systemctl start certbot.timer

# Test renewal
certbot renew --dry-run
```

Backup method (cron):

```bash
crontab -e

# Run renewal check twice per day
0 3 * * * /usr/bin/certbot renew --quiet
0 15 * * * /usr/bin/certbot renew --quiet
```

### Create post-renewal hook

```bash
mkdir -p /etc/letsencrypt/renewal-hooks/post/

nano /etc/letsencrypt/renewal-hooks/post/restart-services.sh
```

```bash
#!/bin/bash
# Runs after SSL certificate is renewed
# Restarts services so they use the new certificate

echo "$(date): SSL renewed - restarting services"

# Restart mail server
systemctl restart exim4
echo "Exim4 restarted"

# Restart web server (uncomment what applies)
# systemctl restart apache2
# systemctl restart nginx

# Log completion
echo "$(date): All services restarted after SSL renewal" \
  >> /var/log/ssl-renewal.log
```

```bash
chmod +x /etc/letsencrypt/renewal-hooks/post/restart-services.sh
bash /etc/letsencrypt/renewal-hooks/post/restart-services.sh
```

### Configure SSL for Exim4

```bash
nano /etc/exim4/exim4.conf.template
```

Add in main configuration:

```text
# SSL certificate paths
tls_certificate = /etc/letsencrypt/live/yourdomain.com/fullchain.pem
tls_privatekey  = /etc/letsencrypt/live/yourdomain.com/privkey.pem

# Offer TLS to all connecting servers
tls_advertise_hosts = *

# Minimum TLS version (TLS 1.2 minimum)
tls_minimum_version = tls1_2
```

Apply:

```bash
update-exim4.conf
systemctl restart exim4
```

---

## 11. Testing & Verification

### Complete verification script

Save as `/usr/local/bin/verify-server.sh`:

```bash
#!/bin/bash

echo "============================================"
echo "      MAIL SERVER VERIFICATION REPORT"
echo "      $(date)"
echo "============================================"

DOMAIN="yourdomain.com"  # Change this!
PASS=0
FAIL=0

check() {
  if [ $1 -eq 0 ]; then
    echo "  ✓ $2"
    PASS=$((PASS + 1))
  else
    echo "  ✗ $2"
    FAIL=$((FAIL + 1))
  fi
}

echo ""
echo "[ HOSTNAME ]"
HOSTNAME=$(hostname -f)
echo "  Hostname: $HOSTNAME"

echo ""
echo "[ IPv6 ]"
ip -6 addr show | grep "scope global" > /dev/null 2>&1
check $? "IPv6 address configured"
ping6 google.com -c 1 -W 5 > /dev/null 2>&1
check $? "IPv6 internet connectivity"

echo ""
echo "[ EXIM4 ]"
systemctl is-active exim4 > /dev/null 2>&1
check $? "Exim4 is running"
exim4 -bV > /dev/null 2>&1
check $? "Exim4 config is valid"

echo ""
echo "[ FAIL2BAN ]"
systemctl is-active fail2ban > /dev/null 2>&1
check $? "Fail2Ban is running"
fail2ban-client status exim > /dev/null 2>&1
check $? "Exim jail is active"
fail2ban-client status exim-spam > /dev/null 2>&1
check $? "Exim-spam jail is active"

echo ""
echo "[ SSL CERTIFICATE ]"
certbot certificates 2>&1 | grep -q "VALID"
check $? "SSL certificate is valid"

echo ""
echo "[ DKIM ]"
dig TXT dkim._domainkey.$DOMAIN +short | grep -q "DKIM1"
check $? "DKIM DNS record exists"
ls /etc/exim4/dkim/$DOMAIN.key > /dev/null 2>&1
check $? "DKIM private key exists"

echo ""
echo "[ DNS RECORDS ]"
dig A $DOMAIN +short > /dev/null 2>&1
check $? "A record exists"
dig MX $DOMAIN +short > /dev/null 2>&1
check $? "MX record exists"
dig TXT $DOMAIN +short | grep -q "spf"
check $? "SPF record exists"
dig TXT _dmarc.$DOMAIN +short | grep -q "DMARC"
check $? "DMARC record exists"

echo ""
echo "============================================"
echo "  Results: $PASS passed, $FAIL failed"
echo "============================================"
```

Make executable and run:

```bash
chmod +x /usr/local/bin/verify-server.sh
verify-server.sh
```

### Online testing tools

Email deliverability:

- https://www.mail-tester.com (send test email, get score)
- https://mxtoolbox.com (check MX, SPF, DKIM, DMARC)
- https://dmarcian.com/dmarc-inspector (check DMARC)
- https://toolbox.googleapps.com (Google email tools)

SSL checker:

- https://www.ssllabs.com/ssltest (full SSL analysis)
- https://whatsmychaincert.com (certificate chain check)

Blacklist checker:

- https://mxtoolbox.com/blacklists.aspx (check if IP is blacklisted)

---

## 12. Troubleshooting

### Exim4 issues

```bash
systemctl status exim4

tail -50 /var/log/exim4/mainlog
tail -50 /var/log/exim4/paniclog

exim4 -bV

tail -f /var/log/exim4/mainlog

echo "Test" | mail -s "Test Subject" test@gmail.com

exim4 -bp
exim4 -qf
```

### Fail2Ban issues

```bash
tail -50 /var/log/fail2ban.log

fail2ban-regex /var/log/exim4/mainlog \
  /etc/fail2ban/filter.d/exim.conf \
  --print-all-matched

fail2ban-client set exim unbanip 1.2.3.4
fail2ban-client set exim banip 1.2.3.4
fail2ban-client reload
```

### IPv6 issues

```bash
ip -6 route show

ip -6 route add default via fe80::1 dev ens5

sysctl net.ipv6.conf.all.disable_ipv6

sysctl -w net.ipv6.conf.all.disable_ipv6=0
echo "net.ipv6.conf.all.disable_ipv6 = 0" >> /etc/sysctl.conf

ip -6 route flush all
netplan apply
```

### SSL issues

```bash
certbot certificates

certbot renew --force-renewal

certbot certificates | grep "Domains"

openssl verify \
  -CAfile /etc/letsencrypt/live/yourdomain.com/chain.pem \
  /etc/letsencrypt/live/yourdomain.com/cert.pem
```

---

## Quick Reference

### Important file locations

| File | Purpose |
|---|---|
| `/etc/exim4/exim4.conf.template` | Main Exim config |
| `/etc/exim4/update-exim4.conf.conf` | Exim settings |
| `/etc/exim4/dkim/` | DKIM key files |
| `/var/log/exim4/mainlog` | Exim main log |
| `/etc/fail2ban/jail.local` | Fail2Ban jail config |
| `/etc/fail2ban/filter.d/` | Fail2Ban filter files |
| `/var/log/fail2ban.log` | Fail2Ban log |
| `/etc/netplan/` | Network config |
| `/etc/letsencrypt/live/` | SSL certificates |
| `/etc/hosts` | Local hostname mapping |
| `/etc/hostname` | Server hostname file |

### Important commands

```bash
# Exim
update-exim4.conf
systemctl restart exim4
exim4 -bV
exim4 -bp

# Fail2Ban
fail2ban-client status
fail2ban-client reload
fail2ban-client set exim unbanip IP

# SSL
certbot certificates
certbot renew --dry-run
certbot renew

# Network
netplan apply
ip -6 addr show
ip -6 route show

# DNS
dig A domain.com
dig AAAA domain.com
dig MX domain.com
dig TXT domain.com
```

### DNS records checklist

| Record | Type | Example Value |
|---|---|---|
| `yourdomain.com` | A | `1.2.3.4` |
| `yourdomain.com` | AAAA | `2001:db8::1` |
| `mail.yourdomain.com` | A | `1.2.3.4` |
| `mail.yourdomain.com` | AAAA | `2001:db8::1` |
| `yourdomain.com` | MX | `mail.yourdomain.com` |
| `yourdomain.com` | TXT | `v=spf1 ip4:1.2.3.4 -all` |
| `dkim._domainkey` | TXT | `v=DKIM1; k=rsa; p=...` |
| `_dmarc` | TXT | `v=DMARC1; p=none; rua=mailto:...` |

Documentation maintained by: Your Name  
Last updated: 2024  
Version: 1.0

---

## How to Upload to GitHub

```bash
# On your computer:

# 1. Create new file called README.md
# 2. Paste the entire markdown content above
# 3. Save the file

# Then on GitHub:
# ](#)*
