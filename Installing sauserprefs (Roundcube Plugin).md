# Quick Install (3 Steps)

## Step 1: Download and Place Plugin

```bash
cd /path/to/roundcube/plugins
wget https://github.com/johndoh/roundcube-sauserprefs/archive/refs/tags/1.20.2.tar.gz
tar xzf 1.20.2.tar.gz
mv roundcube-sauserprefs-1.20.2 sauserprefs
rm 1.20.2.tar.gz
```

**Important:** The folder name must be exactly `sauserprefs`.

---

## Step 2: Configure Plugin

```bash
cd sauserprefs
cp config.inc.php.dist config.inc.php
nano config.inc.php
```

Edit only these **3 settings**:

```php
// Your SpamAssassin database connection
$config['sauserprefs_db_dsnw'] = 'mysqli://username:password@localhost/spamassassin';

// If using SpamAssassin v4 (recommended)
$config['sauserprefs_sav4'] = true;

// How usernames are stored (email format)
$config['sauserprefs_userid'] = '%u@%d';
```

---

## Step 3: Enable Plugin

Edit your Roundcube config:

```bash
nano /path/to/roundcube/config/config.inc.php
```

Add to plugins array:

```php
$config['plugins'] = ['sauserprefs'];  // Add to your existing plugins
```

**Done!** Users will see a **"Spam"** tab in **Settings** where they can manage whitelists, blacklists, and spam score.

---

# What You Need Before Installing

- SpamAssassin configured to read user prefs from **MySQL/MariaDB**
- Database with a `userpref` table (columns: `username`, `preference`, `value`)

If you don't have the database setup yet, create it with:

```sql
CREATE TABLE userpref (
  username varchar(100) NOT NULL,
  preference varchar(50) NOT NULL,
  value varchar(255) NOT NULL,
  PRIMARY KEY (username, preference)
);
```

That’s the minimal setup—no complicated configurations needed.