# Complete Guide: Enable User-Friendly Email Filters in Roundcube (managesieve plugin)

## What This Does

Users get a graphical interface in Roundcube where they can:

- Create filter rules using dropdowns (subject, sender, body, etc.)
- Choose actions (move to folder, delete, mark as read, forward, etc.)
- Set conditions (contains, does not contain, is, begins with, etc.)

Example:

> If subject contains `product bla` → Move to Junk

All filters run **server-side** (even when user is not logged in).

---

## 1. Install Dovecot Sieve and ManageSieve

```bash
sudo apt update
sudo apt install dovecot-sieve dovecot-managesieved
```

---

## 2. Configure Dovecot for Sieve

### Edit `/etc/dovecot/conf.d/20-managesieve.conf`

```bash
sudo nano /etc/dovecot/conf.d/20-managesieve.conf
```

Make sure these lines are present and uncommented:

```text
service managesieve-login {
  inet_listener sieve {
    port = 4190
  }
}

protocol sieve {
}
```

### Edit `/etc/dovecot/conf.d/90-sieve.conf`

```bash
sudo nano /etc/dovecot/conf.d/90-sieve.conf
```

Ensure:

```text
plugin {
  sieve = file:~/sieve;active=~/.dovecot.sieve
}
```

Restart Dovecot:

```bash
sudo systemctl restart dovecot
```

---

## 3. Enable managesieve Plugin in Roundcube

Edit:

- `/usr/share/fastpanel2-roundcube/config/config.inc.php`

Add/ensure the plugin is enabled (example plugins array):

```php
$config['plugins'] = array(
    'jqueryui',
    'markasjunk',
    'sauserprefs',
    'managesieve'
);
```

---

## 4. Configure managesieve Plugin

The plugin should already be in Roundcube (it's a core plugin). If config doesn't exist, create it:

```bash
cd /usr/share/fastpanel2-roundcube/plugins/managesieve
cp config.inc.php.dist config.inc.php 2>/dev/null || echo "Config already exists or not needed"
nano config.inc.php
```

Set (if needed):

```php
$config['managesieve_host'] = 'localhost';
$config['managesieve_port'] = 4190;
$config['managesieve_usetls'] = false;
```

Usually defaults work fine.

---

## 5. Set Permissions

```bash
chown -R www-data:www-data /usr/share/fastpanel2-roundcube/plugins/managesieve
```

---

## 6. Restart PHP-FPM

```bash
sudo systemctl restart php8.3-fpm
```

---

## 7. Test The Filter UI

1. Log in to Roundcube.
2. Go to **Settings → Filters** (or **Settings → Message Filters**).

You should see an interface with:

- Filter name field
- Rules dropdown: Subject, From, To, Body, Size, etc.
- Condition dropdown: Contains, Does not contain, Is, Begins with, Ends with, etc.
- Text input for keywords
- Action dropdown: Move to folder, Delete, Mark as read, Forward, etc.

---

## 8. Example: Block Emails with "product bla" in Subject

User creates filter:

- **Filter name:** Block Product Bla  
- **If:** Subject → Contains → `product bla`  
- **Then:** Move message to → Junk  
- **Save**

Now any email with `product bla` in the subject is automatically moved to Junk folder (server-side, even if user is offline).

---

## 9. Advanced: Multiple Conditions

Users can add multiple rules, for example:

- If subject contains `spam` **OR** from domain `badspammer.com` → Delete
- If sender is `boss@company.com` **AND** subject contains `urgent` → Mark as important and move to Inbox

---

## 10. Verify Sieve Scripts Are Created

Filters are saved as `.sieve` files in user mail directories:

```bash
ls -lah /var/www/fastuser/data/email/USERNAME/USERDOMAIN/sieve/
cat /var/www/fastuser/data/email/USERNAME/USERDOMAIN/.dovecot.sieve
```

You’ll see Sieve script syntax like:

```text
require ["fileinto"];
if header :contains "subject" "product bla" {
  fileinto "Junk";
  stop;
}
```

---

## 11. Troubleshooting

### No "Filters" tab in Settings?

- Check plugin is enabled in `config.inc.php`.
- Check Roundcube error logs:
  - `/usr/share/fastpanel2-roundcube/logs/errors`

### "Connection to managesieve server failed"

- Ensure `dovecot-managesieved` is installed and running.
- Check port 4190 is listening:

```bash
ss -plnt | grep 4190
```

- Verify Dovecot config enables managesieve protocol.

### Filters not working (messages not moving)

- Check user's sieve script was created.
- Verify Dovecot sieve plugin is loaded:

```bash
doveconf -n | grep sieve
```

- Test with `sieve-test` command (if available).

---

## Summary

With `managesieve` plugin enabled, users get a full graphical filter editor where they can:

- Create rules based on subject, sender, body, size, headers, etc.
- Set actions: move to folder, delete, mark, forward, etc.
- Combine multiple conditions (AND/OR logic)

No coding or manual Sieve scripting required—all done via dropdown menus and text boxes.

This is exactly the user-friendly filtering system you described!