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
and make it executable.
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
