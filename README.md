# ipcam-project
This is an ip camera project for my school.

## Borg 
* Borgbackup packages
Depending on your distribution the commande might differ you can refer on [the borg website](https://borgbackup.readthedocs.io/en/stable/installation.html)
```
apt install borgbackup
```
* Borgbackup Initialization
1. Create a repositrory on a remote server
> You'll need to install borg on your remote server too
```
borg init --encryption=repokey user@hostname:/path/to/repo
```
> then you'll need to insert a passphrase for your repo which will be needed in the next steps

2. Create a script to autmoate backups

```bash
#!/bin/sh
export BORG_REPO=ssh:/username@ip:port/path

export BORG_PASSPHRASE=""

info() { printf "\n%s %s\n\n" "$( date )" "$*" >&2; }
trap 'echo $( date ) Backup interrupted >&2; exit 2' INT TERM

info "Starting backup"

borg create                         \
    --verbose                       \
    --filter AME                    \
    --list                          \
    --stats                         \
    --show-rc                       \
    --compression lz4               \
                                    \
    ::'{hostname}-{now}'            \
    /path/to/file                          \

backup_exit=$?

info "Pruning repository"

borg prune                          \
    --list                          \
    --prefix '{hostname}-'          \
    --show-rc                       \
    --keep-daily    7               \
    --keep-weekly   4               \
    --keep-monthly  6               \

prune_exit=$?

global_exit=$(( backup_exit > prune_exit ? backup_exit : prune_exit ))

if [ ${global_exit} -eq 0 ]; then
    info "Backup and Prune finished successfully"
elif [ ${global_exit} -eq 1 ]; then
    info "Backup and/or Prune finished with warnings"
else
    info "Backup and/or Prune finished with errors"
fi

exit ${global_exit}


```

3. Use cron to run the script at a specific interval
4. Restore a backup
Use `borg list` to see all the archives present in the backup folder
```bash
borg list /path/to/repo
```
Then to restore the backup you'll need to use the `borg mount`
```bash
borg mount /path/to/repo::name_of_archive /path/to/restore
```
