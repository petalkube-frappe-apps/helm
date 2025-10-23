# ERPNext Migration Guide - Complete

## Overview
This guide covers migrating ERPNext from a traditional server installation to an Oracle Kubernetes Engine (OKE) cluster.

**Source:** Ubuntu server with ERPNext at `/home/ubuntu/erpnext`  
**Destination:** OKE Kubernetes cluster  
**Method:** Backup, transfer, and restore

---

## STEP 1: BACKUP ON SOURCE SERVER ✅

### Find Your Site Name

```bash
# Go to your ERPNext installation directory
cd /home/ubuntu/erpnext

# List all sites
bench --site list
```

### Create Backup

Run from your bench directory:

```bash
cd /home/ubuntu/erpnext

bench --site erpnext.localhost backup \
  --with-files \
  --compress \
  --verbose \
  --backup-path-db /home/ubuntu/backup/db.sql.gz \
  --backup-path-files /home/ubuntu/backup/files.tar \
  --backup-path-private-files /home/ubuntu/backup/private-files.tar \
  --backup-path-conf /home/ubuntu/backup/config.json
```

**Options:**
- `--site erpnext.localhost` - Your site name
- `--with-files` - Include uploaded files
- `--compress` - Compress files to save space
- `--verbose` - Show detailed progress
- `--backup-path-*` - Backup file paths with proper extensions

**Output files:**
- Database: `db.sql.gz` - 2.0 MiB (compressed SQL dump)
- Public files: `files.tar` - 6.4 KiB (tar archive)
- Private files: `private-files.tar` - 81.4 KiB (tar archive)
- Config: `config.json` - 582 Bytes (JSON file)

### Turn Off Maintenance Mode (if enabled)

```bash
bench --site erpnext.localhost set-maintenance-mode off
sleep 60
# Hard refresh browser: Ctrl+Shift+R or Cmd+Shift+R
```

---

## STEP 2: TRANSFER TO LOCAL MACHINE ✅

### Download from Source Server

**Script:** `scp_oracle_bastion.sh` (available in Git repository)

**Purpose:** Connects to Oracle bastion host (source server) and downloads backup files using SCP.

**Usage:**
```bash
# Make script executable
chmod +x scp_oracle_bastion.sh

# Download backup from bastion host
./scp_oracle_bastion.sh /home/ubuntu/backup /home/harris/tmp/erpnext-backup
```

**Arguments:**
- First argument: Remote path on bastion host (`/home/ubuntu/backup`)
- Second argument: Local destination folder (`/home/harris/tmp/erpnext-backup`)

**What it does:**
1. Connects to Oracle bastion host using SSH key authentication
2. Downloads all backup files from `/home/ubuntu/backup`
3. Saves to `/home/harris/tmp/erpnext-backup` on local machine

**Files downloaded:**
- `/home/harris/tmp/erpnext-backup/db.sql.gz` - Database backup
- `/home/harris/tmp/erpnext-backup/files.tar` - Public files
- `/home/harris/tmp/erpnext-backup/private-files.tar` - Private files
- `/home/harris/tmp/erpnext-backup/config.json` - Site configuration

---

## STEP 3: UPLOAD TO KUBERNETES POD ✅

### Find the Pod Name

```bash
kubectl get pods -n erpnext | grep gunicorn
```

**Example output:**
```
frappe-bench-erpnext-gunicorn-79b78fdb5b-xm255   1/1   Running
```

### Upload Backup Files to Pod

```bash
# Upload entire backup directory
kubectl cp /home/harris/tmp/erpnext-backup erpnext/frappe-bench-erpnext-gunicorn-75f7cfcd9b-btsr7:/tmp/erpnext-backup/
```

**Command breakdown:**
- `kubectl cp` - Kubernetes copy command
- `/home/harris/tmp/erpnext-backup` - Source directory (local machine)
- `erpnext/` - Namespace
- `frappe-bench-erpnext-gunicorn-75f7cfcd9b-btsr7` - Pod name
- `:/tmp/erpnext-backup/` - Destination path in pod

**Why gunicorn pod?**
- The gunicorn pod runs the main ERPNext web application
- It has access to the bench directory and database connections
- It's the appropriate container to run restore commands

**Verify Upload:**
```bash
# Check files are in the pod
kubectl exec -n erpnext frappe-bench-erpnext-gunicorn-75f7cfcd9b-btsr7 -- ls -lh /tmp/erpnext-backup/

# Check directory structure
kubectl exec -n erpnext frappe-bench-erpnext-gunicorn-75f7cfcd9b-btsr7 -- du -sh /tmp/erpnext-backup/*
```

---

## STEP 4: RESTORE ON KUBERNETES POD ✅

### Prerequisites Check
- Backup uploaded to pod (Step 3) ✅
- Site name identified: `erp.localtest.me` ✅
- Decision made: Overwrite existing site ✅

### Version Compatibility Warning

**IMPORTANT:** During restore, you may see a version mismatch warning:
- Source backup: Frappe 16.0.0-dev
- Destination site: Frappe 15.84.0

This is a **downgrade** scenario which can cause:
- Database schema incompatibilities
- Missing features or fields
- Unexpected behavior

**Recommendation:** After restore, upgrade the destination to match source version using `bench update --upgrade`

### Restore Process

#### Step 1: Enter the Pod

```bash
kubectl exec -it -n erpnext frappe-bench-erpnext-gunicorn-75f7cfcd9b-btsr7 -- /bin/bash
```

#### Step 2: Navigate to Bench Directory

```bash
cd /home/frappe/frappe-bench
```

#### Step 3: Verify Backup Files

The backup files should already have proper extensions:

```bash
# Check files exist
ls -lh /tmp/erpnext-backup/backup/db.sql.gz
ls -lh /tmp/erpnext-backup/backup/files.tar
ls -lh /tmp/erpnext-backup/backup/private-files.tar
```

#### Step 4: Run Restore Command

```bash
bench --site erp.localtest.me --force restore \
  --with-public-files /tmp/erpnext-backup/backup/files.tar \
  --with-private-files /tmp/erpnext-backup/backup/private-files.tar \
  /tmp/erpnext-backup/backup/db.sql.gz
```

**Options:**
- `--site erp.localtest.me` - Target site name
- `--force` - Overwrite existing site without confirmation
- `--with-public-files` - Restore uploaded/public files
- `--with-private-files` - Restore private/attachment files
- Last parameter: Database backup file path

**Expected output:**
```
Restoring Database file...
Restoring public files...
Restoring private files...
```

**If you see version mismatch warning:**
```
Warning: Backup is from version 16.0.0-dev but site is on 15.84.0
```
This is expected. Continue to Step 5.

#### Step 5: Exit the Pod

```bash
exit
```

---

## STEP 5: UPGRADE VERSION (IF NEEDED) ✅

If you encountered version mismatch warning in Step 4, upgrade the site:

```bash
# Re-enter the pod
kubectl exec -it -n erpnext frappe-bench-erpnext-gunicorn-75f7cfcd9b-btsr7 -- /bin/bash

# Navigate to bench
cd /home/frappe/frappe-bench

# Run upgrade
bench update --upgrade

# Exit
exit
```

**What this does:**
- Pulls latest code for all apps
- Runs database migrations
- Updates Python dependencies
- Rebuilds assets

**Note:** In Kubernetes, this upgrade is temporary unless you update the container images in your deployment.

---

## STEP 6: VERIFY RESTORATION ✅

### Check Site Status

```bash
kubectl exec -n erpnext frappe-bench-erpnext-gunicorn-75f7cfcd9b-btsr7 -- bench --site erp.localtest.me migrate
```

### Access the Site

Open your browser and navigate to the site URL (e.g., `http://erp.localtest.me` or your ingress URL)

### Verify Data

- Check if users exist
- Verify DocTypes and customizations
- Test file uploads/downloads
- Check custom fields and forms

---

## Final Checklist

- [ ] All backup files created on source server
- [ ] Backups downloaded to local machine
- [ ] Backups uploaded to Kubernetes pod
- [ ] Database restored successfully
- [ ] Public and private files restored
- [ ] Version upgraded (if needed)
- [ ] Site accessible via browser
- [ ] Data and customizations verified
- [ ] All functionality tested

---

## Important Notes

1. **Version Compatibility:** Always test thoroughly in staging
2. **Consider upgrading to v16** for version parity
3. **Update Kubernetes deployment images** for persistence
4. **Monitor logs** for any issues
5. **Create backup schedule** for new installation

---

## Troubleshooting

**If pod restarts and loses changes:**
- Kubernetes pods are ephemeral
- Need to update deployment/Helm chart with v16 images
- Current changes only persist while pod is running

**If site doesn't load:**
- Check pod logs: `kubectl logs -n erpnext <pod-name>`
- Verify services: `kubectl get svc -n erpnext`
- Check ingress: `kubectl get ingress -n erpnext`

**If customizations missing:**
- Re-run migrate: `bench --site erp.localtest.me migrate`
- Rebuild: `bench build --force`
- Clear cache: `bench --site erp.localtest.me clear-cache`