# Full Guide: Install Roundcube Password Plugin for Dovecot Virtual Users

This guide uses a custom **exec** driver and a **sudo-enabled** bash script to securely change passwords from the Roundcube interface.

---

## Step 1: Create the Password Change Script (Bash)

This script contains the logic to verify the old password and write the new one.

Create and open the script file:

```bash
sudo nano /usr/local/bin/change-virtual-pass.sh
```

Paste the entire script below into the file:

```bash
#!/bin/bash

# Log file for recording every step executed
LOG_FILE="/var/log/change-password-debug.log"

# Path to the Dovecot password file
PASSWD_FILE="/etc/dovecot/dovecot.passwd"

# Variables passed from Roundcube
EMAIL="$1"
OLD_PASSWORD="$2"
NEW_PASSWORD="$3"

# Start logging
echo "==========================" >> "$LOG_FILE"
echo "$(date): Starting password change for user $EMAIL" >> "$LOG_FILE"

# Check if the password file exists
if [ ! -f "$PASSWD_FILE" ]; then
    echo "$(date): Error: Password file $PASSWD_FILE not found!" >> "$LOG_FILE"
    exit 1
fi

# Check if the user exists in the password file
USER_ENTRY=$(grep "^$EMAIL:" "$PASSWD_FILE")
if [ -z "$USER_ENTRY" ]; then
    echo "$(date): Error: User $EMAIL not found in $PASSWD_FILE" >> "$LOG_FILE"
    exit 1
fi

# Extract the current hashed password
CURRENT_HASH=$(echo "$USER_ENTRY" | cut -d':' -f2)

# Validate the old password
echo "$OLD_PASSWORD" | doveadm pw -t "$CURRENT_HASH" > /dev/null 2>&1
if [ $? -ne 0 ]; then
    echo "$(date): Error: Old password is incorrect for user $EMAIL" >> "$LOG_FILE"
    exit 1
fi

# Generate the new hashed password
NEW_HASH=$(doveadm pw -s SHA256-CRYPT -p "$NEW_PASSWORD")
if [ -z "$NEW_HASH" ]; then
    echo "$(date): Error: Failed to generate the new hashed password!" >> "$LOG_FILE"
    exit 1
fi

# Escape the hash for sed
CURRENT_HASH_ESCAPED=$(sed 's/[&/\]/\\&/g' <<< "$CURRENT_HASH")

# Update ONLY the password hash field
sed -i "s|^\($EMAIL:\)$CURRENT_HASH_ESCAPED\(.*\)|\1$NEW_HASH\2|" "$PASSWD_FILE"

if [ $? -eq 0 ]; then
    echo "$(date): Password updated successfully for user $EMAIL" >> "$LOG_FILE"
    echo "Password updated successfully!"
    exit 0
else
    echo "$(date): Error: Failed to update password!" >> "$LOG_FILE"
    exit 1
fi
```

Save and exit (`Ctrl+O`, `Enter`, `Ctrl+X`).

Make the script executable and owned by root:

```bash
sudo chmod +x /usr/local/bin/change-virtual-pass.sh
sudo chown root:root /usr/local/bin/change-virtual-pass.sh
```

---

## Step 2: Create the Custom `exec.php` Driver

This PHP file adds the **exec** driver to your Roundcube plugin.

Create and open the driver file:

```bash
sudo nano /usr/share/fastpanel2-roundcube/plugins/password/drivers/exec.php
```

Paste the entire PHP code below into the file:

```php
<?php

/**
 * exec Driver for Roundcube Password Plugin
 */

class rcube_exec_password
{
    public function save($currpass, $newpass, $username)
    {
        $rcmail = rcmail::get_instance();
        $cmd = $rcmail->config->get('password_exec_cmd');

        if (empty($cmd)) {
            rcmail::write_log('password', 'ERROR: password_exec_cmd not configured');
            return PASSWORD_ERROR;
        }

        // Replace placeholders
        $cmd = str_replace('%l', escapeshellarg($username), $cmd);
        $cmd = str_replace('%u', escapeshellarg($username), $cmd);
        $cmd = str_replace('%o', escapeshellarg($currpass), $cmd);
        $cmd = str_replace('%c', escapeshellarg($currpass), $cmd);
        $cmd = str_replace('%p', escapeshellarg($newpass), $cmd);
        $cmd = str_replace('%n', escapeshellarg($newpass), $cmd);

        // Execute
        exec($cmd, $output, $return_code);

        // Log
        rcmail::write_log('password', 'Command: ' . preg_replace('/\'[^\']+\'/', '***', $cmd));
        rcmail::write_log('password', 'Return code: ' . $return_code);

        if (!empty($output)) {
            rcmail::write_log('password', 'Output: ' . implode("\n", $output));
        }

        if ($return_code == 0) {
            return PASSWORD_SUCCESS;
        }

        return PASSWORD_ERROR;
    }
}
?>
```

Save and exit.

---

## Step 3: Configure sudo Permissions

This securely allows the web user (`www-data`) to run **only your script** as root.

Create a new sudoers file:

```bash
sudo nano /etc/sudoers.d/roundcube-virtual
```

Paste this single line into the file (change `www-data` if your web server uses a different user):

```text
www-data ALL=(ALL) NOPASSWD: /usr/local/bin/change-virtual-pass.sh
```

Save and exit.

Set the correct, secure permissions for this file:

```bash
sudo chmod 440 /etc/sudoers.d/roundcube-virtual
```

---

## Step 4: Configure Roundcube

Now, tell Roundcube to use your new driver and script.

Open your Roundcube configuration file:

```bash
sudo nano /usr/share/fastpanel2-roundcube/config/config.inc.php
```

Go to the bottom of the file and add the following lines.  
(If you already have a `$config['plugins']` line, just add `'password'` to it.)

```php
// --- Password Plugin Settings ---

// Make sure the plugin is enabled
$config['plugins'] = array('password');

// Use the custom 'exec' driver we created
$config['password_driver'] = 'exec';

// Command to execute
$config['password_exec_cmd'] = 'sudo /usr/local/bin/change-virtual-pass.sh %u %o %n';

// THIS IS IMPORTANT:
// Set to false because our script already verifies the old password.
// This prevents the Dovecot "Temporary authentication failure" IMAP error.
$config['password_confirm_current'] = false;

// Enable logging in Roundcube
$config['password_log'] = true;
```

Save and exit.

---

## Step 5: Fix Dovecot File Permissions

This was the final error we fixed. This ensures Dovecot can read its own password file to log you in.

Set the owner to `root` and the group to `dovecot`:

```bash
sudo chown root:dovecot /etc/dovecot/dovecot.passwd
```

Set the permissions so root can read/write and the dovecot group can only read:

```bash
sudo chmod 640 /etc/dovecot/dovecot.passwd
```

---

## Step 6: Restart Services and Test

Create the empty log file so the script can write to it:

```bash
sudo touch /var/log/change-password-debug.log
sudo chmod 644 /var/log/change-password-debug.log
```

Restart all related services to apply every change:

```bash
sudo systemctl restart dovecot
sudo systemctl restart nginx
sudo systemctl restart php8.1-fpm  # <-- Use your server's PHP-FPM version
```

Go to your Roundcube login page, log in, and navigate to:

**Settings → Password**

Try to change it. It should now work perfectly.

---

## If It Fails: How to Debug

Check these two log files immediately after a failed attempt:

### 1) Your script's log

```bash
tail -f /var/log/change-password-debug.log
```

This will tell you if the script ran and what error it had (e.g., `Old password is incorrect`).

### 2) Roundcube's error log

```bash
tail -f /usr/share/fastpanel2-roundcube/logs/errors.log
```

This will tell you if Roundcube had a problem (e.g., `Driver file does not exist` or a sudo error).