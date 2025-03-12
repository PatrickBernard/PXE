# Configuration PXE

## Avant propos

Concerne Debian, mais probablement adaptable à vos besoin

Les installations PXE ont besoin de plusieurs éléments :

* un serveur dhcp
* un serveur tftp
* les fichiers de netboot
* la configurations des menus
* Recette d'installation distribué (preseed) par un serveur http (192.168.1.3)
* Script suplémentaire post installation (192.168.1.3)

On considère que les fichiers seront installer dans /srv/tftp-efi

## Création des fichiers netboot

Il est possible de créer les fichiers sur n'importe quel machine, il vaut mieux les générés depuis une debian parfaitement à jour à la derniere versions.

* Installation des paquets

```bash
  apt install syslinux syslinux-efi pxelinux
```

* Création de la structure des répertoires

```bash
  mkdir -p /srv/tftp-efi/boot/
  mkdir -p /srv/tftp-efi/bios/pxelinux.cfg/
  mkdir -p /srv/tftp-efi/efi32/pxelinux.cfg/
  mkdir -p /srv/tftp-efi/efi64/pxelinux.cfg/
  cd /srv/tftp-efi/bios && ln -s ../boot boot
  cd /srv/tftp-efi/efi32 && ln -s ../boot boot
  cd /srv/tftp-efi/efi64 && ln -s ../boot boot
```

* Récupération des images d'installation

Exemple en buster

```bash
  mkdir /tmp/buster
  wget http://ftp.fr.debian.org/debian/dists/buster/main/installer-amd64/current/images/netboot/netboot.tar.gz
  tar -zxf netboot.tar.gz 
  mv debian-installer/amd64/ /srv/tftp-efi/boot/debian/installer/buster
```

* Ajout des firmwares non-free à l'initrd

```bash
  wget http://cdimage.debian.org/cdimage/unofficial/non-free/firmware/buster/current/firmware.cpio.gz
  cp debian-installer/amd64/initrd.gz initrd.gz.orig
  cat initrd.gz.orig firmware.cpio.gz > initrd.gz
  cp initrd.gz /srv/tftp-efi/boot/debian/installer/buster/amd64/
```

* Configuration des menus

```bash
  vi /srv/tftp-efi/bios/pxelinux.cfg/default
```

```bash
  DEFAULT vesamenu.c32
  PROMPT 0
  TIMEOUT 300
  MENU LABEL TITLE Serveur PXE

  # dimensions du menu
  MENU LABEL WIDTH 80
  MENU LABEL MARGIN 10
  MENU LABEL ROWS 7
  MENU LABEL TABMSGROW 18
  MENU LABEL CMDLINEROW 12
  MENU LABEL ENDROW 24
  MENU LABEL TIMEOUTROW 20

  LABEL local
        MENU LABEL Boot normal sur la machine
        localboot 0

  LABEL buster-amd64
        MENU LABEL Debian Buster amd64
        KERNEL boot/debian/installer/buster/amd64/linux
        append vga=788 initrd=boot/debian/installer/buster/amd64/initrd.gz url=http://<serveur contenant le preseed>/preseed-buster.conf locale=fr_FR ++

  LABEL buster-user-amd64
        MENU LABEL Debian Buster amd64 - user
        KERNEL boot/debian/installer/buster/amd64/linux
        append vga=788 initrd=boot/debian/installer/buster/amd64/initrd.gz ++
```

La config est pour le moment identique pour efi32 et efi64 sauf le titre

```bash
  sed -s "s/^MENU LABEL TITLE Serveur PXE$/MENU LABEL TITLE Serveur PXE - efi32/g" bios/pxelinux.cfg/default > /srv/tftp-efi/efi32/pxelinux.cfg/default
  sed -s "s/^MENU LABEL TITLE Serveur PXE$/MENU LABEL TITLE Serveur PXE - efi64/g" bios/pxelinux.cfg/default > /srv/tftp-efi/efi64/pxelinux.cfg/default
```

## OBSOLETE depuis debian 12 : Script de mise à jour initrd/kernel (update-initrd-firmware.sh)

```bash
  cd /srv/tftp-efi
  mkdir scripts
  vi scripts/update-initrd-firmware.sh
```

```bash
  #!/bin/bash
  if [ -z $1 ] || [ -z $2 ] ; then
    echo "Usage : $0 distrib tftp-location"
    echo "example $0 buster /srv/tftp-efi"
    exit 1
  fi
  DISTRIB=$1
  DIR=$2
  mkdir -p /tmp/$DISTRIB
  cd /tmp/$DISTRIB
  echo "===Récupération des netboot et firmware pour $DISTRIB==="
  wget http://ftp.fr.debian.org/debian/dists/$DISTRIB/main/installer-amd64/current/images/netboot/netboot.tar.gz
  [ -f firmware.cpio.gz ] || wget http://cdimage.debian.org/cdimage/unofficial/non-free/firmware/$DISTRIB/current/firmware.cpio.gz
  echo "===Extraction initrd==="
  mkdir netboot
  tar -C netboot -xzf netboot.tar.gz
  cp netboot/debian-installer/amd64/initrd.gz initrd.gz.orig
  echo "===Ajout des firmwares non-free dans l'initrd==="
  cat initrd.gz.orig firmware.cpio.gz > initrd.gz
  echo "===Copie des nouveau fichiers dans le tftp==="
  cp initrd.gz $DIR/boot/debian/installer/$DISTRIB/amd64/
  cp netboot/debian-installer/amd64/linux $DIR/boot/debian/installer/$DISTRIB/amd64/
```

* Création d'un artefact deployable

```bash  
  cd /srv
  tar czf tftp-efi.tgz tftp-efi
```

## Config tftp

* Installation des paquets

```bash
  apt install tftpd-hpa
```

* Configuration

Les fichiers du netboot sont stockés dans /srv/tftp-efi

```bash
  # vi /etc/default/tftpd-hpa
```

```bash
  TFTP_USERNAME="tftp"
  TFTP_DIRECTORY="/srv/tftp-efi"
  TFTP_ADDRESS="0.0.0.0:69"
  TFTP_OPTIONS="++secure"
```

Charge la nouvelle configuration

```bash
  systemctl restart tftpd-hpa
```

Vérification que le tftp est bien lancer et écoute

```bash
  netstat -uap | grep tftp 
```

Devrait donner quelque chose comme ça :

```bash
  udp        0      0 *:tftp                  *:*                                 46724/in.tftpd 
```

## Config DHCP

C'est lors de la demande d'ip dhcp que le pxe indiqueras son mode de boot normal/efi32/efi64

* Installation des paquets

```bash
  apt install isc-dhcp-server
```

* Configuration

```bash
  vi /etc/dhcp/dhcpd.conf
```

On définit les valeurs permettant de selectionner le type de boot

```bash
  option space PXE;
  option PXE.mtftp-ip code 1 = ip-address;
  option PXE.mtftp-cport code 2 = unsigned integer 16;
  option PXE.mtftp-sport code 3 = unsigned integer 16;
  option PXE.mtftp-tmout code 4 = unsigned integer 8;
  option PXE.mtftp-delay code 5 = unsigned integer 8;
  option arch code 93 = unsigned integer 16;
```

Sélection l'emplacement physique selon le type de boot

```bash
  subnet 192.168.1.0 netmask 255.255.255.0 {
      range 192.168.1.100 192.168.1.250;
      option domain-search "local";
      option routers 192.168.1.1;

      # Boot PXE
      next-server 192.168.1.2;
      #filename "pxelinux.0";
      option time-offset -18000;

      #Boot PXE efi
          if option arch = 00:06 {
                  filename "efi32/syslinux.efi";
          } else if option arch = 00:07 {
                  filename "efi64/syslinux.efi";
          } else if option arch = 00:09 {
                  filename "efi64/syslinux.efi";
          } else {
                  filename "bios/pxelinux.0";
          }
  }
```

On recharge la configuration

```bash
  service isc-dhcp-server restart
```

## Configuration Preseed

Ajouter dans un serveur web votre preseed : /var/www/preseed-buster.conf

```bash

    #### Buster
    ### Préconfigurer la locale seule définit la langue, le pays et la locale.
    d-i debian-installer/locale string fr_FR.UTF-8
    
    ### Clavier
    # clavier fr
    d-i keyboard-configuration/xkb-keymap select fr(latin9)
    # désactivation de la séléction fine
    d-i keyboard-configuration/toggle select No toggling
    
    ### Configuration des interfaces réseaux
    d-i netcfg/choose_interface select auto
    d-i netcfg/get_hostname string unassigned-hostname
    d-i netcfg/get_domain string localdomain
    d-i netcfg/dhcp_failed note
    d-i netcfg/dhcp_options select Configure network manually
    d-i netcfg/wireless_wep string
    
    ### Sélection du miroir
    d-i mirror/country string manual
    d-i mirror/http/hostname string ftp.fr.debian.org
    d-i mirror/http/directory string /debian
    d-i mirror/http/proxy string
    
    # Sélection de la distribution
    d-i mirror/suite string buster
    
    ### Gestion des comptes utilisateurs
    # pas de compte de base
    d-i passwd/make-user boolean false
    # password bidon lors de l'installation initial
    # toujours garder un password facile a taper qq soit la disposition du clavier
    d-i passwd/root-password password debROOT2013
    d-i passwd/root-password-again password debROOT2013
    
    ### TimeZone
    # Cette commande permet de régler l'horloge matérielle sur UTC :
    d-i clock-setup/utc boolean true
    # Zone Europe/Paris
    d-i time/zone string Europe/Paris
    # régler l'heure via ntp pendant l'installation
    d-i clock-setup/ntp boolean true
    
    
    ### Partitionnement
    # Nous voulons une table de partition au format GPT
    d-i partman-basicfilesystems/choose_label string gpt
    d-i partman-basicfilesystems/default_label string gpt
    d-i partman-partitioning/choose_label string gpt
    d-i partman-partitioning/default_label string gpt
    d-i partman/choose_label string gpt
    d-i partman/default_label string gpt
    partman-partitioning partman-partitioning/choose_label select gpt
    
    # Partionnement en lvm 
    d-i partman-auto/method string lvm
    # Pour être sûr, on supprime une éventuelle configuration LVM
    d-i partman-lvm/device_remove_lvm boolean true
    d-i partman-lvm/device_remove_lvm_span boolean true
    d-i partman-auto/purge_lvm_from_device  boolean true
    # Même chose pour le RAID
    d-i partman-md/device_remove_md boolean true
    # On nomme le nouveau VG
    d-i partman-auto-lvm/new_vg_name string system
    
    # Recette de partionnement fine (expert)
    # attention tout doit etre sur une seul ligne
    # 2go swap,
    # 10go de / bootable,
    # partition lvm
    # 1.5go de /var,
    # 2.5go de /usr
    # 1go de /tmp,
    # 1go de /var/tmp,
    # 1go de /home,
    # 2go de /var/log,
    # le reste dans dans le lv dummy
    
    d-i partman-auto/expert_recipe string                           \
        server-part ::                                              \
            2048 100 2048 linux-swap                                \
                    $primary{ }                                     \
                    method{ swap } format{ }                        \
            .                                                       \
            10000 100 10000 ext4                                    \
                    $primary{ } $bootable{ }                        \
                    method{ format } format{ }                      \
                    use_filesystem{ } filesystem{ ext4 }            \
                    mountpoint{ / }                                 \
            .                                                       \
            10 100 1000000000 lvm                                   \
                    $primary{ } $defaultignore{ }                   \
                    method{ lvm } vg_name{ system }                 \
            .                                                       \
            1536 100 1536 ext4                                      \
                    $lvmok{ } in_vg{ system } lv_name{ var }        \
                    method{ lvm } format{ }                         \
                    use_filesystem{ } filesystem{ ext4 }            \
                    mountpoint{ /var }                              \
            .                                                       \
            2560 100 2560 ext4                                      \
                    $lvmok{ } in_vg{ system } lv_name{ usr }        \
                    method{ lvm } format{ }                         \
                    use_filesystem{ } filesystem{ ext4 }            \
                    mountpoint{ /usr }                              \
            .                                                       \
            1024 100 1024 ext4                                      \
                    $lvmok{ } in_vg{ system } lv_name{ tmp }        \
                    method{ lvm } format{ }                         \
                    use_filesystem{ } filesystem{ ext4 }            \
                    mountpoint{ /tmp }                              \
            .                                                       \
            1024 100 1024 ext4                                      \
                    $lvmok{ } in_vg{ system } lv_name{ var_tmp }    \
                    method{ lvm } format{ }                         \
                    use_filesystem{ } filesystem{ ext4 }            \
                    mountpoint{ /var/tmp }                          \
            .                                                       \
            1024 100 1024 ext4                                      \
                    $lvmok{ } in_vg{ system } lv_name{ home }       \
                    method{ lvm } format{ }                         \
                    use_filesystem{ } filesystem{ ext4 }            \
                    mountpoint{ /home }                             \
            .                                                       \
            2048 100 2048 ext4                                      \
                    $lvmok{ } in_vg{ system } lv_name{ var_log }    \
                    method{ lvm } format{ }                         \
                    use_filesystem{ } filesystem{ ext4 }            \
                    mountpoint{ /var/log }                          \
            .                                                       \
            10 100 1000000000 ext4                                  \
                    method{ lvm }                                   \
                    $lvmok{ } in_vg{ system } lv_name{ dummy }      \
                    mountpoint{ /opt }                              \
            .
      
    # On accepte tout automatiquement
    d-i partman-lvm/confirm boolean true
    d-i partman/confirm_write_new_label boolean true
    d-i partman/choose_partition select finish
    d-i partman/confirm boolean true
    
    # fstab utilisera des UUID plutôt que des noms de périphériques
    d-i partman/mount_style select uuid
    
    ### Installation du système
    # On ne souhaite pas installer les paquets recommandés
    # L'installation sera limitée aux paquets "essentials"
    d-i base-installer/install-recommends boolean false
    
    # Configuration Apt
    apt-cdrom-setup apt-setup/cdrom/set-first boolean false
    d-i apt-setup/non-free boolean true
    d-i apt-setup/contrib boolean true
    d-i apt-setup/use_mirror boolean true
    # On active les services security et updates
    d-i apt-setup/services-select multiselect security, updates
    d-i apt-setup/security_host string security.debian.org
    # Notez qu'on ne peut pas modifier la chaîne pour updates.
    # Pas d'envoi de rapport vers popcon
    popularity-contest popularity-contest/participate boolean false
    
    # Installation des paquets supplémentaires
    # Sélection des taches
    tasksel tasksel/first multiselect standard, ssh-server
    # Paquets suplémentaires
    d-i pkgsel/include string openssh-server less vim bridge-utils ifenslave-2.6 vlan ntpdate ntp
    # Mise à jour des paquets après debootstrap.
    d-i pkgsel/upgrade select safe-upgrade 
    
    ### Finalisation de l'installation
    # On lance un script custom après installation
    d-i preseed/late_command string \
    cd /target/root; \
    wget -q -O postinst.sh http://192.168.1.3/postinst.sh; \
    chmod 0700 postinst.sh; \
    in-target bash /root/postinst.sh
    
    # Boot loader installation
    # On préfère laisser le choix du disque pour installer grub
    # Certains configuraitons raid pouvant causer des problemes
    
    # Fin
    d-i finish-install/reboot_in_progress note
    d-i cdrom-detect/eject boolean true 
```

## Fichier postinst.sh

* Il s'agit d'un script custom lancer à la fin de l'installation par la machine cible
* A installer dans 192.168.1.3:/var/www/html/

```bash
  #!/bin/sh
  # Config SSH (root distant par mot de passe)
  sed -i 's/#PermitRootLogin\ prohibit-password/PermitRootLogin\ yes/' /etc/ssh/sshd_config
```

## Source

* <https://wiki.debian.org/DebianInstaller/NetbootFirmware>
* <https://wiki.debian-fr.xyz/PXE_avec_support_EFI>
* <https://medspx.fr/blog/Debian/preseed_snippets>
* <https://www.debian.org/releases/buster/arm64/apbs01.fr.html>
* <https://www.debian.org/releases/buster/example-preseed.txt>

## Erreur et solution

Lors de l'utilisation du PXE, on peut rencontrer plusieurs erreurs.

```bash
- Aucun module du noyau n'a été trouvé.
- La version du noyau utilisée par le programme d'installation est sans doute différente de celle présente sur l'archive Debian.
```

Pour corriger cet erreur, il faut se connecter sur le lxc-dhcp du site concerné et exécuter la commande suivante:

```bash
/srv/tftp-efi/scripts//initrd-firmware.sh buster /srv/tftp-efi
```

Voir le paragraphe ``Script de mise à jour initrd/kernel``
