# ghostbsd-znapzend-backup
Scripts and SOP for ZFS backups using znapzend on GhostBSD

# SOP for Remote Backup and Restore using `znapzend` on GhostBSD
**Anonymous-Cupholder**  
*Copyright 2024*

## Introduction
This SOP outlines the steps to configure, use, validate, and restore ZFS backups using `znapzend` on GhostBSD, storing backups to a remote location.

## Prerequisites
- GhostBSD with ZFS configured.
- `znapzend` installed.
- Sufficient disk space on both source and remote backup pools.
- SSH keys set up for passwordless access to the remote server.
- Remote server with ZFS configured for receiving backups.

## Installation of `znapzend`

1. **Install `znapzend`**:
    ```
    pkg install znapzend
    ```

## Configuration of `znapzend`

### Set Up SSH Keys
Generate SSH keys and copy them to the remote server for passwordless login.

1. **Generate SSH keys**:
    ```
    ssh-keygen -t rsa -b 4096 -C "${USER}@${HOSTNAME}" -f ~/.ssh/id_rsa -N ""
    ```

2. **Copy the public key to the remote server**:
    ```
    ssh-copy-id user@backupserver
    ```

### Create a Snapshot Schedule
Create a snapshot schedule to define backup frequency and retention.

1. **Create a Snapshot Schedule**:
    Replace `pool/dataset` with your actual pool and dataset names, and `backup_pool/dataset` with your backup pool and dataset names on the remote server.
    ```
    znapzendzetup create --recursive --tsformat='%Y-%m-%d-%H%M%S' \
      SRC '1d=>7d,1w=>4w,1m=>6m' \
      pool/dataset \
      DST 'user@backupserver:backup_pool/dataset'
    ```
    This setup:
    - Keeps daily snapshots for 7 days.
    - Keeps weekly snapshots for 4 weeks.
    - Keeps monthly snapshots for 6 months.
    - Backs up from `pool/dataset` to `backup_pool/dataset` on the remote server.

2. **Verify Configuration**:
    ```
    znapzendzetup list
    ```

### Start and Enable `znapzend`

1. **Start `znapzend`**:
    ```
    service znapzend start
    ```

2. **Enable `znapzend` at Boot**:
    ```
    sysrc znapzend_enable="YES"
    ```

## Scripts

### 1. `setup_znapzend.sh`
This script sets up `znapzend`, configures SSH keys, and creates a snapshot schedule.

```
#!/bin/sh

# Variables
LOCAL_POOL="pool"
LOCAL_DATASET="dataset"
REMOTE_USER="user"
REMOTE_SERVER="backupserver"
REMOTE_POOL="backup_pool"
REMOTE_DATASET="dataset"
SNAPSHOT_SCHEDULE="1d=>7d,1w=>4w,1m=>6m"

# Install znapzend
pkg install -y znapzend

# Generate SSH keys if not already present
if [ ! -f ~/.ssh/id_rsa ]; then
    ssh-keygen -t rsa -b 4096 -C "${USER}@${HOSTNAME}" -f ~/.ssh/id_rsa -N ""
fi

# Copy public key to remote server
ssh-copy-id ${REMOTE_USER}@${REMOTE_SERVER}

# Create znapzend schedule
znapzendzetup create --recursive --tsformat='%Y-%m-%d-%H%M%S' \
  SRC "${SNAPSHOT_SCHEDULE}" \
  ${LOCAL_POOL}/${LOCAL_DATASET} \
  DST "${REMOTE_USER}@${REMOTE_SERVER}:${REMOTE_POOL}/${REMOTE_DATASET}"

# Verify configuration
znapzendzetup list

# Start znapzend service
service znapzend start

# Enable znapzend at boot
sysrc znapzend_enable="YES"

echo "znapzend setup and configuration completed successfully."
```

### 2. `backup_now.sh`
This script triggers a manual backup using `znapzend`.

```
#!/bin/sh

# Variables
LOCAL_POOL="pool"
LOCAL_DATASET="dataset"

# Run manual backup
znapzend --runonce ${LOCAL_POOL}/${LOCAL_DATASET}

echo "Manual backup triggered successfully."
```

### 3. `restore_snapshot.sh`
This script restores a specified snapshot from the remote server to a new location.

```
#!/bin/sh

# Variables
LOCAL_POOL="pool"
RESTORE_DATASET="restore_dataset"
REMOTE_USER="user"
REMOTE_SERVER="backupserver"
REMOTE_POOL="backup_pool"
REMOTE_DATASET="dataset"
SNAPSHOT_NAME="snapshot_name"

# Create local restore dataset
zfs create ${LOCAL_POOL}/${RESTORE_DATASET}

# Receive the snapshot from remote server
zfs receive ${LOCAL_POOL}/${RESTORE_DATASET} < <(ssh ${REMOTE_USER}@${REMOTE_SERVER} zfs send ${REMOTE_POOL}/${REMOTE_DATASET}@${SNAPSHOT_NAME})

# Verify restored data
zfs mount ${LOCAL_POOL}/${RESTORE_DATASET}
ls /${LOCAL_POOL}/${RESTORE_DATASET}

echo "Snapshot restored to ${LOCAL_POOL}/${RESTORE_DATASET} successfully."
```

### 4. `validate_backups.sh`
This script validates the backups by running a scrub and comparing datasets using `rsync`.

```
#!/bin/sh

# Variables
SOURCE_POOL="pool"
SOURCE_DATASET="dataset"
BACKUP_POOL="backup_pool"
BACKUP_DATASET="dataset"
REMOTE_USER="user"
REMOTE_SERVER="backupserver"

# Run scrub on source and backup pools
echo "Running scrub on source dataset..."
zfs scrub ${SOURCE_POOL}/${SOURCE_DATASET}
echo "Running scrub on backup dataset..."
ssh ${REMOTE_USER}@${REMOTE_SERVER} zfs scrub ${BACKUP_POOL}/${BACKUP_DATASET}

# Wait for scrub to complete
echo "Waiting for scrub to complete..."
sleep 60  # Adjust based on the size of your datasets

# Check scrub status
SOURCE_STATUS=$(zpool status ${SOURCE_POOL} | grep state)
BACKUP_STATUS=$(ssh ${REMOTE_USER}@${REMOTE_SERVER} zpool status ${BACKUP_POOL} | grep state)

echo "Source Pool Status: ${SOURCE_STATUS}"
echo "Backup Pool Status: ${BACKUP_STATUS}"

# List snapshots
echo "Listing snapshots on source dataset..."
zfs list -t snapshot -r ${SOURCE_POOL}/${SOURCE_DATASET}
echo "Listing snapshots on backup dataset..."
ssh ${REMOTE_USER}@${REMOTE_SERVER} zfs list -t snapshot -r ${BACKUP_POOL}/${BACKUP_DATASET}

# Compare datasets using rsync
echo "Comparing source and backup datasets using rsync..."
rsync -avnc /${SOURCE_POOL}/${SOURCE_DATASET}/ ${REMOTE_USER}@${REMOTE_SERVER}:/${BACKUP_POOL}/${BACKUP_DATASET}/

echo "Backup validation completed successfully."
```

### 5. `schedule_validation.sh`
This script sets up a cron job to regularly perform validation.

```
#!/bin/sh

# Path to validation script
VALIDATION_SCRIPT_PATH="/path/to/validate_backups.sh"

# Add cron job to run validation script every Sunday at 2 AM
(crontab -l ; echo "0 2 * * 0 ${VALIDATION_SCRIPT_PATH}") | crontab -

echo "Scheduled validation script to run every Sunday at 2 AM."
```

## Usage Instructions

1. **Set up `znapzend` and SSH keys**:
    ```
    ./setup_znapzend.sh
    ```

2. **Trigger a manual backup**:
    ```
    ./backup_now.sh
    ```

3. **Restore a snapshot**:
    ```
    ./restore_snapshot.sh
    ```

4. **Validate backups**:
    ```
    ./validate_backups.sh
    ```

5. **Schedule regular validation**:
    ```
    ./schedule_validation.sh
    ```

## Best Practices

- Regularly test backups and restores.
- Monitor `znapzend` logs for errors.
- Ensure adequate storage space on both source and backup pools.
- Use SSH keys for secure and automated remote backups.

---

This SOP provides a comprehensive guide to setting up, using, validating, and restoring backups with `znapzend` on GhostBSD, storing backups to a remote location. Regular testing and monitoring are key to ensuring data integrity and availability.


