## Borg 
> Backup des vidéos.
* Borgbackup packages

    En fonction de votre distribution les commandes peuvent changer, veuillez vous référez au [site de  Borg](https://borgbackup.readthedocs.io/en/stable/installation.html).
    > Pour ce projet on tourne le serveur sous Debian.
    ```           
    apt install borgbackup
    ```
* Initialisation de Borg
1. Création d'un repo sur le serveur vidéo

    Lancer cette commande dans le terminal avec le chemin d'accès voulu pour votre backup.
    ```
    borg init --encryption=repokey /path/to/repo
    ```
    > Après avoir exécuté cette commande vous aurez besoin de définir un mot de passe qui sera nécessaire pour l'accès aux fichiers du repo.

2. Création du script pour le backup

    Pour cette étape il suffit d'utiliser le script ci-dessous et de modifier le chemin BORG_REPO avec le chemin d'accès de votre repo borg, puis d'insérer le mot de passe de votre répo entre les guillemets à la fin de la ligne export BORG_PASSPHRASE= . Enfin il faudra indiquer dans le borg create le chemin d'accès du dossier contenant les vidéos (à la place du /path/to/repo).
    

    ```bash
    #!/bin/sh
    export BORG_REPO=/path/to/repo

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
        /path/to/folder                 \

    backup_exit=$?

    info "Pruning repository"

    borg prune                          \
        --list                          \
        --prefix '{hostname}-'          \
        --show-rc                       \
        --keep-daily    5               \

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
    Ce script permettra d'effectuer un backup des fichiers choisis et de garder seulement de 5 jours différents, au 6e jour l'archive la plus ancienne et supprimer et remplacer par une nouvelle.

3. Exécution du script à un intervalle régulier
    
    Il est possible d'utiliser diverse méthodes afin d'automatiser la backup, on en présentare 2, une avec crontab et l'autre avec systemd, cependant sur ce projet on utilisera un service systemd.

    * En utilisant crontab : 
        
        Exécutez la commande `crontab -e`  sur votre terminal, ensuite sélectionnez votre éditeur de texte, puis insérez la commande `*/5 * * * *     /path/to/script`, cette commande permet à la machine d'éxécuter le script toutes les 5 minutes
    * En utilisant un service systemd : 

        Allez au dossier `/etc/systemd/system` et créez un fichier '.service' avec ce contenu  en modifiant la ligne 'ExecStart' avec le chemin d'accès de votre répo:
    
        ```service
        [Unit]
        Description=Service for automating backup

        [Service]
        Type=oneshot
        ExecStart=/path/to/script
        ```
        Ensuite créez un fichier '.timer' qui sera utilisé pour exécuter le service à intervalle régulier.
        **Important :** le fichier '.timer' et '.service' doivent posséder le même nom.
        >Pour notre projet le service sera exécuter tout les jours à minuit, si le serveur est éteint à cette heure le script sera exécuter dès que le serveur sera en ligne.
    
        ```timer
        [Unit]
        Description =Timer for our backup service

        [Timer]
        OnCalendar=daily
        Persistent=true

        [Install]
        WantedBy=timers.target
        ```
        Importation du service:
        -Recharger les fichiers services : `systemctl daemon-reload`
        -Allumer le service : `systemctl start name.timer`
        -Permettre au service de s'exécuter au lancement : `systemctl enable name.timer`
5. Restoration du backup
    Utiliser `borg list` pour voir toutes les archives présentes dans le repo du backup.
    ```
    borg list /path/to/repo
    ```
     Pour restaurer un backup  il faut utiliser la commande  `borg mount` en indiquand le chemin du repo, le nom de l'archive que l'on souhaite restaurer ainsi que le chemin d'accès du dossier vers lequel on souhaite restaurer le backup.
     ```
    borg mount /path/to/repo::name_of_archive    /path/to/restore
    ```
