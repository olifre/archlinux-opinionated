# Regular system maintenance

## Installing updates and purging things
```
yay
yay -Qtdq | yay -Rns -
```
The last line will for example remove build-only dependencies.

## Prune old Restic backups
As outlined in [Set up Backup: Restic](../02_base_setup/03_backup.md#restic), you should regularly run the following (when there is a good and stable connection, as this will read back all data):
```
systemctl start restic-check-and-prune@root.service
systemctl start restic-check-and-prune@home.service
```
Check logs after this!
