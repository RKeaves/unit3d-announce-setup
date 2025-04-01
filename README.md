# UNIT3D Announce Setup Tutorial

[![UNIT3D Announce](https://img.shields.io/badge/UNIT3D-Announce%20Setup-blueviolet)](https://github.com/HDInnovations/UNIT3D-Announce)

<p align="center">
  <img src="https://ptpimg.me/6o8x8j.png" alt="UNIT3D Logo" style="width: 12%;">
</p>

_Set up the UNIT3D Announce service on your existing UNIT3D installation using this step-by-step guide._

---

<div style="border: 2px solid #e74c3c; background-color: #f9e6e6; padding: 10px; border-radius: 5px; margin: 15px 0;">
  <strong>🚨 IMPORTANT:</strong> Before starting the update, always create a backup of your current installation! In this tutorial, we assume you’ve already created and securely stored your latest backup.
</div>


## Create a Backup

Regular backups protect your UNIT3D installation. Use PHP Artisan to securely back up your files and database.

### Step-by-Step Backup Process

1. **Navigate to Your Project Directory:**

   ```bash
   cd /var/www/html
   ```

2. **Run the Artisan Backup Command:**

   ```bash
   php artisan backup:run
   ```

3. **View Available Backups:**

   To list all generated backups, execute:

   ```bash
   php artisan backup:list
   ```

### unit3d-backup-restore

[![GitHub Backup & Restore Tutorial](https://img.shields.io/badge/UNIT3D%20Backup%20%26%20Restore-Tutorial-blue?style=flat-square)](https://github.com/RKeaves/unit3d-backup-restore)  

> For a more detailed guide on creating and managing backups, you may want to check out this other [tutorial](https://github.com/RKeaves/unit3d-backup-restore)  🚀

---

## Table of Contents

- [Prerequisites](#prerequisites)
- [Installation Steps](#installation-steps)
  - [1. Prepare the Environment](#1-prepare-the-environment)
  - [2. Clone & Configure UNIT3D-Announce](#2-clone--configure-unit3d-announce)
  - [3. Build the Tracker](#3-build-the-tracker)
  - [4. Update UNIT3D Environment](#4-update-unit3d-environment)
  - [5. Configure Nginx & Supervisor](#5-configure-nginx--supervisor)
  - [6. Finalize & Verify Setup](#6-finalize--verify-setup)
- [Troubleshooting](#troubleshooting)
- [Acknowledgements](#acknowledgements)

---

## Prerequisites

- An existing UNIT3D installation (typically located at `/var/www/html`).
- Sudo privileges for making system changes.
- Basic knowledge of terminal commands and text editing.
- Your database credentials (grab these from your current `/var/www/html/.env` file).
- Rust's package manager (`cargo`) and Rust compiler installed.

---

## Installation Steps

### 1. Prepare the Environment

Navigate to the directory where UNIT3D is installed:

```bash
cd /var/www/html
```

### 2. Clone & Configure UNIT3D-Announce

Clone the repository and enter its directory:

```bash
git clone -b v0.1 https://github.com/HDInnovations/UNIT3D-Announce unit3d-announce
cd unit3d-announce
```

Copy the example environment file to create your own configuration:

```bash
cp .env.example .env
```

Open the `.env` file to adjust configuration settings:

- Ensure your DB details are correctly copied from `/var/www/html/.env`.
- **Important:** Check the line with `ANNOUNCE_MIN_ENFORCED=1740,` for a trailing comma. Remove it if present.
- Uncomment the line `REVERSE_PROXY_CLIENT_IP_HEADER_NAME="X-Real-IP"` if you're using a reverse proxy on the same host.

```bash
sudo nano .env
```

### 3. Build the Tracker

Install Rust's package manager (`cargo`) and build the tracker:

```bash
sudo apt update
sudo apt -y install cargo
cargo build --release
```

### 4. Update UNIT3D Environment

Switch back to UNIT3D's base directory:

```bash
cd /var/www/html
```

Add the required environment variables to UNIT3D's `.env` file:

- **Variables to add:** `TRACKER_HOST`, `TRACKER_PORT`, and `TRACKER_KEY`.
- These should match the values in UNIT3D-Announce's `.env` for `LISTENING_IP_ADDRESS`, `LISTENING_PORT`, and `APIKEY`.
- **Note:** `APIKEY` is the site password for API requests to the tracker and must be at least 32 characters long.

```bash
sudo nano .env
```

### 5. Configure Nginx & Supervisor

#### Edit Nginx Configuration

Open the Nginx site configuration file:

```bash
sudo nano /etc/nginx/sites-enabled/default
```

Add the following location block inside the first server block (after the last existing location block):

```nginx
location /announce/ {
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Host $host;
    proxy_pass http://aaa.bbb.ccc.ddd:eeee$request_uri;
    real_ip_header X-Forwarded-For;
    real_ip_recursive on;
    set_real_ip_from fff.ggg.hhh.iii;
}
```

- Replace `aaa.bbb.ccc.ddd:eeee` with your local UNIT3D-Announce IP address and port.
- Replace `fff.ggg.hhh.iii` with your public proxy IP address.

#### Edit Supervisor Configuration

Open the supervisor configuration file:

```bash
sudo nano /etc/supervisor/conf.d/unit3d.conf
```

Append the following block:

```ini
[program:unit3d-announce]
process_name=%(program_name)s_%(process_num)02d
command=/var/www/html/unit3d-announce/target/release/unit3d-announce
directory=/var/www/html/unit3d-announce
autostart=true
autorestart=false
user=root
numprocs=1
redirect_stderr=true
stdout_logfile=/var/www/html/storage/logs/announce.log
```

#### Enable External Tracker in UNIT3D's Config

Edit the `announce.php` configuration file:

```bash
sudo nano config/announce.php
```

Ensure that the external tracker settings are enabled.

### 6. Finalize & Verify Setup

Reset the Nginx configuration and start the announcer, then restart UNIT3D:

```bash
cd /var/www/html && sudo systemctl restart nginx && sudo supervisorctl reread && sudo supervisorctl update && sudo supervisorctl reload && sudo php artisan set:all_cache && sudo systemctl restart php8.4-fpm && sudo php artisan queue:restart
```

Confirm the announcer is running:

```bash
sudo supervisorctl status
```

For more detailed process logs, check the tail of the process log:

```bash
supervisorctl tail -100 unit3d-announce:unit3d-announce_00
```

---

## Troubleshooting

- **Configuration Issues:** Double-check your `.env` files (both in UNIT3D and UNIT3D-Announce) for any typos or mismatches.
- **NGINX Errors:** Run `sudo nginx -t` after editing the configuration to ensure there are no syntax errors.
- **Process Failures:** Review the supervisor logs in `/var/www/html/storage/logs/announce.log` for detailed error messages.
- **Database Variables:** Make sure that `TRACKER_KEY` (APIKEY) is properly randomized and at least 32 characters long.

---

## Acknowledgements

This project was made possible thanks to airclay aka [ericlay](https://github.com/ericlay).

---

_If you have any questions or suggestions, please open an issue or submit a pull request._
