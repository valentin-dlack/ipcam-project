# Projet Infrastructure et SI - Serveur de caméra IP

## Prérequis à l'installation :

- OS : Debian 10 avec 1 root, 1 user (au moins)
- Une IP Locale, un accès internet et une caméra IP avec un flux rtsp.

---

## Installation des paquets requis :

> Pensez à faire un `sudo apt update` & `sudo apt upgrade -y` avant d'installer ces paquets.

Il vous faudra plusieurs paquets afin de faire fonctionner votre serveur. En voici la liste :

- ffmpeg -> `sudo apt install ffmpeg`
- incron -> `sudo apt install incron`
- samba -> `sudo apt install samba`
- borg -> `sudo apt install borgbackup`

Après avoir installer les paquets, on peut commencer la mise en place du serveur.

---

## Setup du serveur :

### **Création de dossiers** :

Dans notre exemple, nous créons 3 dossiers indispensable au fonctionnement, vous pouvez changer le nom **MAIS** il faudra aussi le changer dans les variables des scripts.

Les dossiers requis sont :

- /home/*votre user*/backups
- /home/*votre user*/videos
- /home/*votre user*/scripts

### **Créations des scripts (record & supression)**

Note : Tout ces fichiers sont censés être dans `/home/[votre user]/scripts`  

Nous allons créer deux scripts, l'un pour supprimer les fichiers vidéos lorsqu'il y en a trop, l'autre pour récupérer le flux RTSP de la caméra.  

**Script de record :**  

> Nom du fichier : record.sh, n'oubliez pas de faire `sudo chmod +x [fichier]` pour le rendre executable.

```sh
#!/bin/bash

echo "[ --- Starting record --- ]"
ffmpeg -i rtsp://[ip_camera:554]/[channel de record] -c:v copy -c:a aac -map 0 -f segment -segment_time 600 -segment_format mp4 /home/[votre user]/videos/"out%03d.mp4"
```

**Script de suppression des ancient fichers :**  

> Nom du fichier : deleteOld.sh, n'oubliez pas de faire `sudo chmod +x [fichier]` pour le rendre executable. 

```sh
#!/bin/bash

files=( /home/user/videos/* )
pathd="/home/user/videos"
if [ ${#files[@]} -gt 50 ]
then
        rm $pathd/"$(ls -t $pathd | tail -1)"
fi
```

Les deux premiers scripts requis sont installés, nous allons maintenant mettre en place le service `record.service` qui permettra au record d'être constant et de se démarrer en même temps que la machine.  

Tout d'abord allez dans :  
`$ cd /etc/systemd/system/`  
et créez le fichier `record.service` pour mettre dedans ce contenu :  

```sh
[Unit]
Description= Permanent Recording Service.

[Service]
Type=oneshot
ExecStart=/home/[votre user]/scripts/record.sh

[Install]
WantedBy=multi-user.target
```

Pour mettre à jour les services et prendre en compte le nouveau, faites  
`$ systemctl daemon-reload`  

Nous pouvons enfin le démarrer et l'ajouter au démarrage automatique grace à ces deux commandes (dans l'ordre):  
`$ systemctl start record`  
`$ systemctl enable record`  

Et voilà ! votre flux RTSP est maintenant enregistré sur votre serveur ! Passons à la suppression des anciens fichiers automatique avec Incron

### **Setup Incrontab** :

Après avoir installé le paquet incron comme demandé dans les paquets requis, nous allons pouvoir le configurer.  

> Il faut être en SuperUser ("`$ su`") pour effectuer cette manipulation, faites CTRL+D après la manip pour quitter le mode superuser.  

Faites la commande  
`$ incrontab -e`  
pour accéder à la configuration de incron, ensuite tapez 

```sh
/home/user/videos       IN_CREATE       ./home/user/scripts/deleteOld.sh
```

Cela permettra d'éxecuter le script de suppression dès qu'un nouveau fichier est créé dans le dossier videos.

---

## Sécurisation du serveur :

- Vérifier qu'on est à jour: `apt-get update`  

- Installer le système ssh sur votre machine avec `apt-get install -y openssh-server`

- Ouvrir les configurations ssh : `nano /etc/ssh/ssh_config` 
- Remplacer le port 22 par 2222 (ou le port de votre choix) et décommenter la ligne

- Changer le *PermitRootLogin* en *no* (il est de base en *prohibit-password*)

- Redémarrer le service ssh: `/etc/init.d/ssh restart`

- Création du firewall : `nano /etc/init.d/firewall`

```sh
#!/bin/bash
 
ext=[carte réseau]
if [ $# -eq 1 ]
then
        case $1 in
        start)
                echo "Démarrage de mon superbe mur de feu"
                # vider les règles et les macros
                iptables -F
                iptables -X
                # création de la macro JOURNALDROP
                iptables -N JOURNALDROP
                iptables -A JOURNALDROP -j LOG --log-prefix '[=== FIREWALL ===]'
                iptables -A JOURNALDROP -j DROP
                # autoriser les trames ssh (je suis le serveur)
                # vérification : putty, filezilla (protocole sftp)
                iptables -t filter -A INPUT -i $ext -p tcp --dport 2222 -j ACCEPT
                iptables -t filter -A OUTPUT -o $ext -p tcp --sport 2222 -j ACCEPT
                # autoriser les trames DNS (je suis client)# vérification : nslookup google.fr (paquet dnsutils)
                iptables -A OUTPUT -o $ext -p udp --dport 53 -j ACCEPT
                iptables -A INPUT -i $ext -p udp --sport 53 -j ACCEPT
                # autoriser les trames HTTP (je suis client)# vérification : apt update et apt install
                iptables -A OUTPUT -o$ext -p tcp --dport 80 -j ACCEPT
                iptables -A INPUT -i $ext -p tcp --sport 80 -j ACCEPT
                # autoriser les trames HTTPS (je suis client)
                # vérification : navigateur internet en https
                iptables -A OUTPUT -o $ext -p tcp --dport 443 -j ACCEPT
                iptables -A INPUT -i $ext -p tcp --sport 443 -j ACCEPT
                # toutes les autres trames sont envoyées dans la macro
                                iptables -A INPUT -j JOURNALDROP
                iptables -A OUTPUT -j JOURNALDROP
                ;;
        stop)
                echo "Arrêt du firewall"
                # vider les règles et les macros
                iptables -F
                iptables -X
                # politique par défaut : ACCEPTER
                iptables -P INPUT ACCEPT
                iptables -P FORWARD ACCEPT
                iptables -P OUTPUT ACCEPT
                ;;
        status)
                echo "Etat du firewall :"
                iptables -L
                ;;
        *)
        echo "Paramètres possibles : {start|stop|status}"
                ;;esac
else
        echo "Il faut un paramètre..."
        echo "$(basename $0) {start|stop|status}"
fi
```

---

## Création du partage Samba:

- Garder une sauvegarde des configurations de bases (en cas de problème): `mv /etc/samba/smb.conf /etc/samba/smb.conf.bak`

- Copier l'ancien fichier de configuration pour modifier la copie comme on le souhaite: `cp /etc/samba/smb.conf.bak /etc/samba/smb.conf`

- Ouvrir et modifier le fichier copié: `nano /etc/samba/smb.conf`

    - Vérifier les informations suivante:  

        ```sh
        workgroup = WORKGROUP
        interfaces = 192.168.1.0/24 eth0
        ```

    - Créer une nouvelle section:

        ```text
        [share]
        comment = My new share
        path = home/partage
        browseable = yes
        read only = yes
        guest ok = no
        valid users = username
        ```

    > A la place de *username*, utiliser le nom de l'utilisateur de votre serveur qui aura l'accès au partage

- Vérifier que le dossier *partage* existe bien dans le *home*
    - Sinon le créer :

        ```sh
        mkdir partage
        chmod 777 partage
        ```

- Ajouter l'utilisateur au partage Samba: ``smbdpasswd -a username`` (A la place de *username*, utiliser le nom de l'utilisateur de votre serveur qui aura accès au partage)

Le partage Samba est désormais fonctionnel et vous pouvez y accéder en cherchant "\\\\*ip du serveur*" dans la barre de votre explorateur de fichier windows.

---

## Borg

> Backup des vidéos.

* Borgbackup packages

    En fonction de votre distribution les commandes peuvent changer, veuillez vous référez au [site de Borg](https://borgbackup.readthedocs.io/en/stable/installation.html).
    > Pour ce projet on tourne le serveur sous Debian.

    ```sh
    apt install borgbackup
    ```

* Initialisation de Borg

1. Création d'un repo sur le serveur vidéo 

    Lancer cette commande dans le terminal avec le chemin d'accès voulu pour votre backup.

    ```sh
    borg init --encryption=repokey /path/to/repo
    ```

    > Après avoir exécuté cette commande vous aurez besoin de définir un mot de passe qui sera nécessaire pour l'accès aux fichiers du repo.

2. Création du script pour le backup

    Pour cette étape il suffit d'utiliser le script ci-dessous et de modifier le chemin BORG_REPO avec le chemin d'accès de votre repo borg, puis d'insérer le mot de passe de votre répo entre les guillemets à la fin de la ligne export BORG_PASSPHRASE= . Enfin il faudra indiquer dans le borg create le chemin d'accès du dossier contenant les vidéos (à la place du /path/to/repo).

    ```sh
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

    ```sh
    borg list /path/to/repo
    ```

     Pour restaurer un backup  il faut utiliser la commande  `borg mount` en indiquand le chemin du repo, le nom de l'archive que l'on souhaite restaurer ainsi que le chemin d'accès du dossier vers lequel on souhaite restaurer le backup.

    ```sh
    borg mount /path/to/repo::name_of_archive    /path/to/restore
    ```

---

## Crédits

-- Kévin GRANGER (Borg Backup)  
-- Costa REYPE (Sécu & Samba)  
-- Valentin DAUTREMENT (Record/Sauvegarde, Suppression, Installation générale)
