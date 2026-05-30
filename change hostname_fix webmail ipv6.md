# Task 2 & 3 - Complete Quick Guide

## Webmail IPv6 Fix + Hostname Change

---

# Before You Start - Gather Information

```bash
# Run these first and save the output

# 1. Current hostname
hostname -f

# 2. Server IPs
curl -4 -s ifconfig.me && echo " = IPv4"
curl -6 -s ifconfig.me && echo " = IPv6"

# 3. What is listening
ss -tlnp

# 4. Check Nginx configs location
find /etc/nginx -name "*.conf" 2>/dev/null
find /usr/local/fastpanel2-nginx -name "*.conf" 2>/dev/null

# 5. Check Exim config
cat /etc/exim4/update-exim4.conf.conf | grep "dc_"
```

---

# Fill This Before Starting

```text
SERVER INFO
===========
Current Hostname : (from hostname -f)
New Hostname     : mail.yourdomain.com
IPv4 Address     : (from curl -4 ifconfig.me)
IPv6 Address     : (from curl -6 ifconfig.me)

WEBMAIL INFO
============
Webmail URL      : https://IP:8888/webmail/
Webmail Software : Roundcube (FastPanel)
Panel URL        : https://IP:8888

NGINX INFO
==========
Regular Nginx    : /etc/nginx/conf.d/
FastPanel Nginx  : /usr/local/fastpanel2-nginx/settings/
```

---

# TASK 3 - Change Hostname

## Step 1: Change Hostname

```bash
# Replace mail.yourdomain.com with what customer wants
hostnamectl set-hostname mail.yourdomain.com

# Verify immediately
hostname -f
```

---

## Step 2: Update Hosts File

```bash
# Get your IPs
IPv4=$(curl -4 -s ifconfig.me)
IPv6=$(curl -6 -s ifconfig.me)

echo "IPv4: $IPv4"
echo "IPv6: $IPv6"

# Edit hosts file
nano /etc/hosts
```

Make it look exactly like this:

```text
127.0.0.1       localhost
127.0.1.1       mail.yourdomain.com mail
YOUR_IPv4       mail.yourdomain.com mail
YOUR_IPv6       mail.yourdomain.com mail
```

---

## Step 3: Update Exim Config

```bash
nano /etc/exim4/update-exim4.conf.conf
```

Find and change these two lines:

```bash
dc_other_hostnames='mail.yourdomain.com'
dc_readhost='mail.yourdomain.com'
```

```bash
# Apply and restart
update-exim4.conf
systemctl restart exim4
```

---

## Step 4: Set DNS Records

```text
Go to your DNS provider and add/update:

Type    Name     Value                 TTL
A       mail     YOUR_IPv4             3600
AAAA    mail     YOUR_IPv6             3600
MX      @        mail.yourdomain.com   10
```

---

## Step 5: Set PTR Records (Reverse DNS)

```text
Go to VPS provider control panel (NOT DNS provider):

Set PTR for IPv4:
YOUR_IPv4 → mail.yourdomain.com

Set PTR for IPv6:
YOUR_IPv6 → mail.yourdomain.com

Where to find PTR settings:

Hetzner  → Servers > Networking > Reverse DNS
OVH      → Bare Metal > IP > Modify Reverse
Vultr    → Products > Instances > IPv4 > Reverse DNS
AWS      → Request via support OR use Elastic IP
```

---

## Step 6: Verify Hostname

```bash
echo "=== HOSTNAME CHECK ==="
hostname -f
exim4 -be '$primary_hostname'

echo "=== DNS CHECK ==="
dig A mail.yourdomain.com +short
dig AAAA mail.yourdomain.com +short

echo "=== PTR CHECK ==="
dig -x YOUR_IPv4 +short
dig -x YOUR_IPv6 +short
```

Expected output:

```text
mail.yourdomain.com    ← hostname ✓
mail.yourdomain.com    ← exim hostname ✓
YOUR_IPv4              ← A record ✓
YOUR_IPv6              ← AAAA record ✓
mail.yourdomain.com.   ← PTR IPv4 ✓
mail.yourdomain.com.   ← PTR IPv6 ✓
```

---

# TASK 2 - Fix Webmail IPv6 Issues

## Understand The Problem First

```bash
# Check what is listening and on which IP version
ss -tlnp

# 0.0.0.0:8888  = IPv4 only   ← problem
# [::]:8888     = IPv6 added  ← what we want

# [SPECIFIC_IP]:80  = specific IPv6 only  ← problem
# :::80             = all IPv6            ← what we want
```

```bash
# Test if webmail works on each protocol
curl -4 -k -s -o /dev/null -w "IPv4: %{http_code}\n" \
  https://YOUR_IPv4:8888/webmail/

curl -6 -k -s -o /dev/null -w "IPv6: %{http_code}\n" \
  https://[YOUR_IPv6]:8888/webmail/

# 200 = working
# 000 = failing = needs fix
```

---

## FastPanel Has TWO Nginx Instances

```text
1. Regular Nginx (nginx.service)
   Handles: websites, domains
   Ports: 80, 443
   Configs: /etc/nginx/conf.d/

2. FastPanel Nginx (fastpanel2-nginx.service)
   Handles: FastPanel panel, webmail
   Port: 8888
   Configs: /usr/local/fastpanel2-nginx/settings/

BOTH need to be fixed for full IPv6 support
```

---

## Fix 1 - Regular Nginx Port 80/443

```bash
# Check current config
cat /etc/nginx/conf.d/parking.conf
cat /etc/nginx/conf.d/reuseport.conf
```

```bash
# Edit parking.conf
nano /etc/nginx/conf.d/parking.conf
```

Change listen lines from:

```nginx
listen [YOUR_SPECIFIC_IPv6]:80 default_server;
listen [YOUR_SPECIFIC_IPv6]:443 ssl default_server;
```

To:

```nginx
listen 80 default_server;
listen [::]:80 default_server;
listen 443 ssl default_server;
listen [::]:443 ssl default_server;
```

```bash
# Edit reuseport.conf
nano /etc/nginx/conf.d/reuseport.conf
```

Change from:

```nginx
listen [YOUR_SPECIFIC_IPv6]:443 quic reuseport;
```

To:

```nginx
listen [::]:443 quic reuseport;
```

```bash
# Test and reload
nginx -t
systemctl reload nginx
```

---

## Fix 2 - FastPanel Nginx Port 8888

```bash
# Check current config
cat /usr/local/fastpanel2-nginx/settings/vhost.conf
```

```bash
# Edit the config
nano /usr/local/fastpanel2-nginx/settings/vhost.conf
```

Change from:

```nginx
listen 8888 ssl http2;
```

To:

```nginx
listen 0.0.0.0:8888 ssl;
listen [::]:8888 ssl;
```

```bash
# Restart FastPanel Nginx
systemctl restart fastpanel2-nginx

# Verify port 8888 now has IPv6
ss -tlnp | grep 8888

# Should show BOTH:
# 0.0.0.0:8888   ← IPv4
# [::]:8888      ← IPv6
```

---

## Fix 3 - IPv6 Firewall Rules

```bash
# Check current IPv6 firewall
ip6tables -L INPUT -n

# Add rules if missing
ip6tables -A INPUT -p tcp --dport 80 -j ACCEPT
ip6tables -A INPUT -p tcp --dport 443 -j ACCEPT
ip6tables -A INPUT -p tcp --dport 8888 -j ACCEPT
ip6tables -A INPUT -p tcp --dport 2096 -j ACCEPT
ip6tables -A INPUT -p tcp --dport 2095 -j ACCEPT

# Save rules so they survive reboot
apt install -y iptables-persistent
netfilter-persistent save

# Verify rules saved
ip6tables -L INPUT -n | grep -E "80|443|8888"
```

---

# Final Verification - Both Tasks

```bash
#!/bin/bash

echo "========================================"
echo "   TASK 2 & 3 VERIFICATION"
echo "========================================"

echo ""
echo "[ HOSTNAME ]"
hostname -f
exim4 -be '$primary_hostname'

echo ""
echo "[ PTR RECORDS ]"
IPv4=$(curl -4 -s ifconfig.me)
IPv6=$(curl -6 -s ifconfig.me)

dig -x $IPv4 +short
dig -x $IPv6 +short

echo ""
echo "[ PORTS LISTENING ]"
ss -tlnp | grep -E "8888|:80 |:443 "

echo ""
echo "[ WEBMAIL ACCESS ]"

curl -4 -k -s -o /dev/null -w "IPv4 HTTP: %{http_code}\n" \
  https://$IPv4:8888/webmail/

curl -6 -k -s -o /dev/null -w "IPv6 HTTP: %{http_code}\n" \
  https://[$IPv6]:8888/webmail/

echo ""
echo "[ IPv6 INTERNET ]"
ping6 google.com -c 2 | tail -1

echo ""
echo "========================================"
```

---

# Troubleshooting

## Problem: IPv6 webmail still fails after fix

```bash
# Check FastPanel nginx actually restarted
systemctl status fastpanel2-nginx | grep "Active:"
ss -tlnp | grep 8888

# If [::]:8888 still missing = restart again
systemctl restart fastpanel2-nginx
```

---

## Problem: Nginx config test fails

```bash
# Check what error nginx shows
nginx -t

# Common issue: duplicate default_server
# Only ONE server block can have default_server
# Remove default_server from one of the listen lines
```

---

## Problem: PTR record not resolving

```bash
# PTR records take time to propagate
# Wait 30 minutes then check again

dig -x YOUR_IPv4 +short

# If still wrong = check VPS provider panel
# Make sure PTR is set to FULL hostname including domain

# CORRECT:   mail.yourdomain.com
# WRONG:     mail
```

---

## Problem: Hostname reverts after reboot

```bash
# Check hostname file
cat /etc/hostname

# Should show:
# mail.yourdomain.com

# If wrong fix it
echo "mail.yourdomain.com" > /etc/hostname
hostnamectl set-hostname mail.yourdomain.com
```

---

## Problem: Exim still uses old hostname

```bash
# Check what Exim thinks hostname is
exim4 -be '$primary_hostname'

# If wrong check the config
cat /etc/exim4/update-exim4.conf.conf | grep "dc_"

# Fix and reapply
nano /etc/exim4/update-exim4.conf.conf
update-exim4.conf
systemctl restart exim4
```

---

# Important Notes

```text
1. FastPanel warns "Do not edit this file"
   Changes MAY be overwritten on FastPanel update
   Check configs after every FastPanel update

2. Two separate Nginx services exist in FastPanel
   Always restart BOTH after network changes:
   systemctl reload nginx
   systemctl restart fastpanel2-nginx

3. PTR records are set in VPS provider panel
   NOT in your domain's DNS settings
   Different location for every provider

4. After hostname change always verify:
   - hostname -f shows correct name
   - Exim uses correct name
   - PTR record matches hostname

   All three must match for email deliverability
```

---

# Quick Command Reference

```bash
# Hostname
hostnamectl set-hostname mail.yourdomain.com
hostname -f

# Exim
update-exim4.conf
systemctl restart exim4
exim4 -be '$primary_hostname'

# Regular Nginx
nginx -t
systemctl reload nginx

# FastPanel Nginx
systemctl restart fastpanel2-nginx
ss -tlnp | grep 8888

# Test webmail
curl -4 -k -I https://IP:8888/webmail/
curl -6 -k -I https://[IPv6]:8888/webmail/

# Check PTR
dig -x YOUR_IPv4 +short
dig -x YOUR_IPv6 +short

# Check firewall
ip6tables -L INPUT -n
```