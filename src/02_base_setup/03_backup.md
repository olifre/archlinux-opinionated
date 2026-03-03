# Set up Backup

We use a two-fold backup, one is `btrbk` for local snapshotting and versioning (self-protection) and one is an external backup with `restic`.

## `btrbk`
Create `/usr/local/bin/run-btrbk.sh` with content:
```
#!/bin/bash
UMOUNTAFTER=1
if grep -qs '/mnt/btrfs_pool' /proc/mounts; then
    # Already mounted by user, do not umount after!
    echo "/mnt/btrfs_pool already mounted."
    UMOUNTAFTER=0
else
    echo "Mounting /mnt/btrfs_pool."
    mount /mnt/btrfs_pool
    UMOUNTAFTER=1
fi

if [ $# -eq 0 ]; then
    btrbk --progress -v run
else
    btrbk $@
fi

if [ $UMOUNTAFTER -eq 1 ]; then
    echo "Unmounting /mnt/btrfs_pool."
    umount -l /mnt/btrfs_pool
else
    echo "NOT unmounting /mnt/btrfs_pool."
fi
```
and make it executable:
```
chmod +x /usr/local/bin/run-btrbk.sh
```
Then, install `btrbk` and `mbuffer` and copy over the example config:
```
cp /etc/btrbk/btrbk.conf.example /etc/btrbk/btrbk.conf
```
and adapt it, uncomment all the "complex examples" and "retention policy" at the end, then add:
```
snapshot_preserve_min   2d
snapshot_preserve       12h 7d
snapshot_create         always

timestamp_format        long-iso

volume /mnt/btrfs_pool
  subvolume  rootfs
  subvolume  home
```
Then, create `/etc/systemd/system/btrbk.service` with content:
```
[Unit]
Description=btrbk backup

[Service]
Type=oneshot
ExecStart=/usr/local/bin/run-btrbk.sh
```
Then, create `/etc/systemd/system/btrbk.timer` with content:
```
[Unit]
Description=btrbk hourly backup

[Timer]
OnCalendar=hourly
AccuracySec=5min
Persistent=true

[Install]
WantedBy=multi-user.target
```
Enable all that:
```
systemctl daemon-reload
systemctl enable --now btrbk.timer
```

## Restic
Install and setup `restic` for backup.
```
pacman -S restic
```
Create dir for configs:
```
mkdir -p /etc/restic
```
Within, create `/etc/restic/restic_root.conf` and `/etc/restic/restic_home.conf`, follow this scheme:
```
RESTIC_PASSWORD="secret"
RESTIC_COMPRESSION="max"
AWS_ACCESS_KEY_ID="secret"
AWS_SECRET_ACCESS_KEY="secret"
RESTIC_REPOSITORY="s3:rgw.example.com:7480/my-machine-home"
PRE_BACKUP_COMMAND=""
POST_BACKUP_COMMAND=""
KEEP_WITHIN="2d"
KEEP_LAST=""
KEEP_HOURLY=""
KEEP_DAILY="28"
KEEP_WEEKLY="26"
KEEP_MONTHLY="12"
KEEP_YEARLY="2"
VERBOSITY=1
ONE_FILE_SYSTEM=1
EXCLUDE_CACHES=1
PATH_TO_BACKUP="/home"
EXCLUDE_PATTERNS="'/home/olifre/.cache'"
IEXCLUDE_PATTERNS=""
EXCLUDE_IF_PRESENT_LIST=""
```
You will of course want to change the secrets and backup server address. 
For the `EXCLUDE_PATTERNS` you may want to set for the `home` backup:
```
EXCLUDE_PATTERNS="'/home/olifre/.cache' '/home/olifre/some_cloud_sync'"
```
and for the `root` backup:
```
EXCLUDE_PATTERNS="'/home' '/var/cache/pacman' '/root/.cache' '/var/lib/cvmfs' '/mnt/btrfs_pool'"
```
Make sure the repository names contain different bucket names, e.g. `myhostname-home` and `myhostname-root`!

Finally, make sure the files have permissions `0640` for security, and also protect the directory:
```
chmod 0750 /etc/restic
chmod 0640 /etc/restic/*
```

Then, create the log directory:
```
mkdir -p /var/log/restic/
```
Create two service files, first is `/etc/systemd/system/restic-backup@.service` with content:
```
[Unit]
Description=restic backup

[Service]
Type=oneshot
ExecStart=/bin/bash -c "/usr/local/bin/restic_backup.sh backup /etc/restic/restic_%i.conf 2>&1 | cat -v | tee -a /var/log/restic/restic_%i.log > /dev/null"
```
Second is `/etc/systemd/system/restic-check-and-prune@.service`:
```
[Unit]
Description=restic check-and-prune      

[Service]
Type=oneshot
ExecStart=/bin/bash -c "/usr/local/bin/restic_backup.sh check-and-prune /etc/restic/restic_%i.conf 2>&1 | cat -v | tee -a /var/log/restic/restic_%i.log > /dev/null"
```
The actual script in `/usr/local/bin/restic_backup.sh` should be created with the following content:
```
#!/bin/bash

# Check provided parameters
if [ ${#*} -ne 2 ]; then
  echo "Usage: $(basename $0) <mode> <config_file>"
  exit 1
fi

MODE=$1
CONFIG_FILE=$2

if [ "$MODE" != "backup" ] && [ "$MODE" != "check-and-prune" ]; then
  echo "Error, you passed mode = ${MODE}, but be one of: backup, check-and-prune!"
  exit 1
fi

echo "########## START - $(date) ##########"

if [ ! -r ${CONFIG_FILE} ]; then
  echo "Config file ${CONFIG_FILE} can not be accessed / does not exist!"
  exit 1
fi

set -o allexport
. ${CONFIG_FILE}
set +o allexport

# Ensure HOME is set (needed for cache).
export HOME=/root

# Init repo if absent.
restic snapshots > /dev/null 2> /dev/null
if [ ! $? -eq 0 ]; then
  restic init
  echo "Restic repository created at \"${RESTIC_REPOSITORY}\"."
else
  echo "Using existing restic repository at \"${RESTIC_REPOSITORY}\"."
fi

if [ "$MODE" = "backup" ]; then
  # Handle actual backup commandline arguments.
  RESTIC_BACKUP_PARS=()
  if [ "x${ONE_FILE_SYSTEM}" = "x1" ]; then
    RESTIC_BACKUP_PARS+=("--one-file-system")
  fi
  if [ "x${EXCLUDE_CACHES}" = "x1" ]; then
    RESTIC_BACKUP_PARS+=("--exclude-caches")
  fi
  if [ -n "${EXCLUDE_PATTERNS}" ]; then
    eval "excl_dir_array=($EXCLUDE_PATTERNS)"
    for excl_dir in "${excl_dir_array[@]}"; do
      RESTIC_BACKUP_PARS+=("--exclude")
      RESTIC_BACKUP_PARS+=("${excl_dir}")
    done
  fi
  if [ -n "${IEXCLUDE_PATTERNS}" ]; then
    eval "iexcl_dir_array=($IEXCLUDE_PATTERNS)"
    for iexcl_dir in "${iexcl_dir_array[@]}"; do
      RESTIC_BACKUP_PARS+=("--iexclude")
      RESTIC_BACKUP_PARS+=("${iexcl_dir}")
    done
  fi
  if [ -n "${EXCLUDE_IF_PRESENT_LIST}" ]; then
    eval "excl_if_present_array=($EXCLUDE_IF_PRESENT_LIST)"
    for excl_if_present in "${excl_if_present_array[@]}"; do
      RESTIC_BACKUP_PARS+=("--exclude-if-present")
      RESTIC_BACKUP_PARS+=("${excl_if_present}")
    done
  fi
  
  # Now finally the actual backup.
  SECONDS=0
  echo "Starting backup of \"${PATH_TO_BACKUP}\" to \"${RESTIC_REPOSITORY}\" at $(date)..."
  if [ -n "${PRE_BACKUP_COMMAND}" ]; then
    echo "Running pre-backup-command \"${PRE_BACKUP_COMMAND}\"..."
    $PRE_BACKUP_COMMAND
    echo "Done!"
  fi
  echo "Running restic..."
  restic --verbose=${VERBOSITY} backup "${RESTIC_BACKUP_PARS[@]}" ${PATH_TO_BACKUP}
  echo "Done!"
  if [ -n "${POST_BACKUP_COMMAND}" ]; then
    echo "Running post-backup-command \"${POST_BACKUP_COMMAND}\"..."
    $POST_BACKUP_COMMAND
    echo "Done!"
  fi
  echo "Backup finished at $(date) (after ${SECONDS} seconds)."
  
  # Forget metadata for old snapshots.
  RESTIC_FORGET_PARS=()
  if [ -n "${KEEP_WITHIN}" ]; then
    RESTIC_FORGET_PARS+=("--keep-within" "${KEEP_WITHIN}")
  fi
  if [ -n "${KEEP_LAST}" ]; then
    RESTIC_FORGET_PARS+=("--keep-last" "${KEEP_LAST}")
  fi
  if [ -n "${KEEP_HOURLY}" ]; then
    RESTIC_FORGET_PARS+=("--keep-hourly" "${KEEP_HOURLY}")
  fi
  if [ -n "${KEEP_DAILY}" ]; then
    RESTIC_FORGET_PARS+=("--keep-daily" "${KEEP_DAILY}")
  fi
  if [ -n "${KEEP_WEEKLY}" ]; then
    RESTIC_FORGET_PARS+=("--keep-weekly" "${KEEP_WEEKLY}")
  fi
  if [ -n "${KEEP_MONTHLY}" ]; then
    RESTIC_FORGET_PARS+=("--keep-monthly" "${KEEP_MONTHLY}")
  fi
  if [ -n "${KEEP_YEARLY}" ]; then
    RESTIC_FORGET_PARS+=("--keep-yearly" "${KEEP_YEARLY}")
  fi
  SECONDS=0
  echo "Starting forgetting of old snapshots at $(date)..."
  restic --verbose=${VERBOSITY} forget --cleanup-cache "${RESTIC_FORGET_PARS[@]}"
  echo "Forgetting of old snapshots finished at $(date) (after ${SECONDS} seconds)."
fi

if [ "$MODE" = "check-and-prune" ]; then
  SECONDS=0
  echo "Start of checking at $(date)."
  restic --verbose=${VERBOSITY} check --retry-lock 1h --read-data
  CHECK_RES=$?
  if [ $CHECK_RES -ne 0 ]; then
    echo "Error: Check was not successful, exiting here, not pruning!"
    exit 1
  fi
  echo "End of successful check at $(date). Duration: $SECONDS seconds."

  SECONDS=0
  echo "Start of pruning at $(date)."
  restic --verbose=${VERBOSITY} prune --repack-small
  echo "End of pruning at $(date). Duration: $SECONDS seconds."
fi

echo "########## STOP - $(date) ##########"
```
Afterwards, make it executable:
```
chmod +x /usr/local/bin/restic_backup.sh
```

Note the following assumes we will call the `restic-backup@.service` once every day via a timer (created below), which also marks snapshots for forgetting, but never prunes, as this might destroy data e.g. in case of bad RAM or otherwise corrupted backups.  
For that, there is `restic-check-and-prune@.service` which can be one-shotted manually when there is a stable connection. This will likely not be used on the road with a laptop, as it reads back all data before pruning. Make sure to check the logs when running this.

Now, create the timer for the backup, i.e. `/etc/systemd/system/restic-backup@.timer`:
```
[Unit]
Description=restic daily backup

[Timer]
OnCalendar=*-*-* 23:15:00
AccuracySec=5min
Persistent=true

[Install]
WantedBy=multi-user.target
```
Enable things:
```
systemctl daemon-reload
systemctl enable --now restic-backup@root.timer
systemctl enable --now restic-backup@home.timer
```
You may want to trigger the service units manually for the initial backup.
