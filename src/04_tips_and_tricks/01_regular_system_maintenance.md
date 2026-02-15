# Regular system maintenance

## Staying informed
You will want to subscribe to the very low traffic [Arch-announce mailing list](https://lists.archlinux.org/mailman3/lists/arch-announce.lists.archlinux.org/), which announces important packaging changes. 

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
