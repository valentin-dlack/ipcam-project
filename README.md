## Borg 
* Borgbackup packages

Depending on your distribution the command might differ you can refer on [the borg website](https://borgbackup.readthedocs.io/en/stable/installation.html)
> For this projet we are running our server on debian
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

2. Create a script to automate backups

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
    --keep-daily    ?               \
    --keep-weekly   ?               \
    --keep-monthly  ?               \

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

3. Run the script at regular intervals
    * By creating a crontab job : 
        Run `crontab -e`  in your terminal, select your text editor
        Then in your crontab file insert this line `*/5 * * * *     /path/to/script`, this line means that the script will be run every 5         minutes
    * By creating a systemd service :
    First  go to `/etc/systemd/system` directory and then create a             '.service' file
    
    ```service
    [Unit]
    Description= ...
        
    [Service]
    Type=oneshot
    ExecStart=/path/to/script
    ```
    Then create a '.timer' file which will be used to run the service at a     specific date
    **Important :** the '.timer' and '.service' need to have the same name
    >For the example the script will be run every day at         00:00am, and if the machine is down it during this period it will be executed when starting the machine
    
    ```timer
    [Unit]
    Description = ...
    
    [Timer]
    OnCalendar=daily
    Persistent=true
    
    [Install]
    WantedBy=timers.target
    ```
    Importation of the service:
    -Reload the service files : `systemctl daemon-reload`
    -Start the service : `systemctl start name.timer`
    -Enable service on startup : `systemctl enable name.timer`
5. Restore a backup
Use `borg list` to see all the archives present in the backup folder
```
borg list /path/to/repo
```
 To restore the backup you'll need to use the `borg mount` command
```
borg mount /path/to/repo::name_of_archive /path/to/restore
```
