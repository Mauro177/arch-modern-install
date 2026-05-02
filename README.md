# Guide d'installation d'Arch Linux moderne et sécurisé

[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![Arch Linux](https://img.shields.io/badge/OS-Arch%20Linux-blue?logo=arch-linux)](https://archlinux.org/)
[![Last Commit](https://img.shields.io/github/last-commit/Mauro177/arch-modern-install)](https://github.com/Mauro177/arch-modern-install/commits/main)
[![Issues](https://img.shields.io/github/issues/Mauro177/arch-modern-install)](https://github.com/Mauro177/arch-modern-install/issues)

Installation complète d'Arch Linux incluant le chiffrement LUKS2 avec déverrouillage automatique via TPM 2.0, Unified Kernel Image (UKI), Secure Boot, le système de fichiers Btrfs, ainsi qu'un environnement de bureau KDE Plasma ou GNOME.

## Table des matières
1. [Préparation du live CD](#1-préparation-du-live-cd)
2. [Partitionnement et chiffrement du disque](#2-partitionnement-et-chiffrement-du-disque)
3. [Configuration Btrfs et sous-volumes](#3-configuration-btrfs-et-sous-volumes)
4. [Installation minimale du système de base](#4-installation-minimale-du-système-de-base)
5. [Configuration de base du système](#5-configuration-de-base-du-système)
6. [Unified Kernel Image (UKI)](#6-unified-kernel-image-uki)
7. [Post-installation](#7-post-installation)
8. [Secure Boot + TPM 2.0](#8-secure-boot--tpm-20)
9. [Utilisateur et AUR Helper](#9-utilisateur-et-aur-helper)
10. [Hook Kernel-install](#10-hook-kernel-install)
11. [Environnement de bureau](#11-environnement-de-bureau)
12. [Snapshots Btrfs avec `Snapper`](#12-snapshots-btrfs-avec-snapper)
13. [Configurations finales](#13-configurations-finales)
14. [Vérifications finales](#14-vérifications-finales)
15. [Résumé de la stack](#15-résumé-de-la-stack)

---

## Vue d'ensemble

Ce guide met en place une installation d'Arch Linux moderne, sécurisée et minimaliste :

- **Chiffrement complet du disque** - LUKS2 (déverrouillage automatique via TPM 2.0 au démarrage)
- **Système de fichiers Btrfs** - sous-volumes et compression Zstd
- **Unified Kernel Image (UKI)** — noyau, initramfs et ligne de commande regroupés dans un fichier EFI bootable
- **Secure Boot** - UKI signée (via `sbctl`)
- **Environnement de bureau** - KDE Plasma ou GNOME (Wayland)
- **Snapshots automatiques** - Snapper

---

## Prérequis

- Une clé USB bootable avec l'ISO d'Arch Linux ([téléchargement](https://archlinux.org/download/))
- Un PC 64 bits avec UEFI (pas de BIOS Legacy)
- TPM 2.0 et Secure Boot (désactivé au début)
- Accès à Internet (câble recommandé, sinon voir [ici](https://wiki.archlinux.org/title/Iwd#iwctl) pour se connecter en Wi-Fi depuis le live CD)
- Environ 25 Go d'espace disque minimum

> **Connexion SSH optionnelle** : depuis le live CD, vous pouvez définir un mot de passe temporaire (`passwd`) et récupérer l'adresse IP (`ip a`) pour vous connecter en SSH depuis une autre machine afin de faciliter le copier-coller.

---

## 1. Préparation du live CD

```bash
# Configurer le clavier (AZERTY)
loadkeys fr

# Vérifier la connexion à Internet
ping archlinux.org

# Configurer le fuseau horaire et activer NTP pour obtenir la date et l'heure exactes
timedatectl set-timezone Europe/Paris
timedatectl set-ntp true
hwclock --systohc

# Vérification
date
```

---

## 2. Partitionnement et chiffrement du disque

### Identifier le disque cible pour l'installation d'Arch Linux

*Repérez votre disque cible (ex: `nvme0n1`, `sda`) :*
```bash
lsblk | grep disk
```

### Partitionner le disque

*Stocker dans une variable le disque cible de l'installation (ex. /dev/nvme0n1, /dev/sda) afin de faciliter la suite du guide pour permettre le copier-coller des commandes :*

```bash
export disk=/dev/monDisque
```

*⚠️ Attention, effacement complet du disque si vous ne faites pas de Dual Boot :*

```bash
wipefs -a $disk
```

*Créez un schéma GPT avec deux partitions :*

| Partition | Taille | Type |
|-----------|--------|------|
| `1` | 1G | EFI System |
| `2` | Reste | Linux filesystem |

```bash
cfdisk $disk
```
> ⚠️ **Attention** si une partition EFI existe déjà et que vous voulez du Dual Boot, ne la recréez pas.

### Formater la partition EFI

*Stocker dans une variable la partition EFI :*

<details>
  <summary>SATA</summary>

```bash
export efip=${disk}1
```

</details>

<details>
  <summary>NVMe</summary>

```bash
export efip=${disk}p1
```

</details>

*⚠️ FORMATAGE UNIQUEMENT si la partition EFI est nouvelle :*

```bash
mkfs.fat -F32 $efip
```

### Chiffrement LUKS2 de la partition racine

*Stocker dans une variable la partition cible d'Arch Linux. Si vous ne faites pas de Dual Boot, garder le '2'. Sinon, vérifier le numéro avec la commande 'lsblk $disk' :*

<details>
  <summary>SATA</summary>

```bash
export part=${disk}2
```

</details>

<details>
  <summary>NVMe</summary>

```bash
export part=${disk}p2
```

</details>

*Initialiser la partition chiffrée (vous devrez choisir une passphrase temporairement, le temps de mettre en place le TPM 2.0) :*

```bash
cryptsetup luksFormat $part
```

*Déchiffrer la partition (la rendre accessible sous /dev/mapper/cryptroot) :*


<details>
  <summary>SSD</summary>

```bash
cryptsetup --allow-discards --perf-no_read_workqueue --perf-no_write_workqueue --persistent open $part cryptroot
```

</details>

<details>
  <summary>HDD</summary>

```bash
cryptsetup open $part cryptroot
```

</details>

> LUKS2 chiffre l'intégralité de la partition.
---

## 3. Configuration Btrfs et sous-volumes

Btrfs permet de créer des **sous-volumes** indépendants sur une même partition, ce qui est pratique pour des snapshots ciblés et pour gérer finement les options de montage.

### Créer le système de fichiers et les sous-volumes

*Formater la partition :*

```bash
mkfs.btrfs -L root /dev/mapper/cryptroot
```

*Monter dans /mnt :*

```bash
mount /dev/mapper/cryptroot /mnt
```

*Créer les sous-volumes :*

```bash
btrfs subvolume create /mnt/@           # Racine du système
btrfs subvolume create /mnt/@home       # Données utilisateur
btrfs subvolume create /mnt/@log        # Journaux système
btrfs subvolume create /mnt/@cache      # Cache
btrfs subvolume create /mnt/@tmp        # Fichiers temporaires
btrfs subvolume create /mnt/@vms        # Images de machines virtuelles (si besoin)
btrfs subvolume create /mnt/@snapshots  # Snapshots Snapper
```

*(Si utilisé) Désactiver le Copy-on-Write pour les images de VMs :*

```bash
chattr +C /mnt/@vms
```
> Optimisation nécessaire pour éviter la fragmentation et améliorer les performances des VMs.

*Démontage :*

```bash
umount /mnt
```

### Monter le futur système

*/ :*

```bash
mount -o nodev,compress=zstd,subvol=@ /dev/mapper/cryptroot /mnt
```
*/home :*

```bash
mount --mkdir -o nosuid,nodev,compress=zstd,subvol=@home /dev/mapper/cryptroot /mnt/home
```
*/var/log :*

```bash
mount --mkdir -o noexec,nosuid,nodev,compress=zstd,subvol=@log /dev/mapper/cryptroot /mnt/var/log
```
*/var/cache :*

```bash
mount --mkdir -o noexec,nosuid,nodev,noatime,compress=zstd,subvol=@cache /dev/mapper/cryptroot /mnt/var/cache
```
*/var/tmp :*

```bash
mount --mkdir -o nosuid,nodev,noatime,compress=zstd,subvol=@tmp /dev/mapper/cryptroot /mnt/var/tmp
```
*/var/lib/libvirt/images : (Si vous utilisez libvirt)*

```bash
mount --mkdir -o nosuid,nodev,noatime,subvol=@vms /dev/mapper/cryptroot /mnt/var/lib/libvirt/images
```
*/.snapshots :*

```bash
mount --mkdir -o nodev,compress=zstd,subvol=@snapshots /dev/mapper/cryptroot /mnt/.snapshots
```

*[EFI] /boot :*

Je choisis `/boot`, car avec `kernel-install` le noyau et l'initramfs ne sont plus copiés dans ce répertoire lorsqu'on utilise les UKIs, ce qui en fait un répertoire vide pour accueillir la partition EFI.

```bash
mount --mkdir -o nosuid,nodev,noexec,noatime,nosymfollow,umask=0077 $efip /mnt/boot
```

> `noatime` désactive la mise à jour de l'heure d'accès aux fichiers — cela évite des écritures inutiles pour les données de cache ou temporaires.  
> Les points de montage sont également sécurisés avec `nodev` `nosuid` `noexec`. Finalement `compress=zstd` active la compression, ce qui réduit l'utilisation du disque et peut accélérer la lecture ou l'écriture dans certains cas.


---

## 4. Installation minimale du système de base

### Configurer les miroirs

J'utilise exclusivement les miroirs de `pkgbuild.com` (en https pour la sécurité), car ils sont gérés par la team DevOps d'Arch Linux. Ils sont en top tier 1, très rapides, ne seront jamais désynchronisés ni abandonnés, et sont de confiance vu leur provenance.

*Ajout des miroirs :*

```bash
echo 'Server = https://geo.mirror.pkgbuild.com/$repo/os/$arch' > /etc/pacman.d/mirrorlist
echo 'Server = https://frankfurt.mirror.pkgbuild.com/$repo/os/$arch' >> /etc/pacman.d/mirrorlist
echo 'Server = https://berlin.mirror.pkgbuild.com/$repo/os/$arch' >> /etc/pacman.d/mirrorlist
echo 'Server = https://london.mirror.pkgbuild.com/$repo/os/$arch' >> /etc/pacman.d/mirrorlist
echo 'Server = https://fastly.mirror.pkgbuild.com/$repo/os/$arch' >> /etc/pacman.d/mirrorlist
echo 'Server = https://losangeles.mirror.pkgbuild.com/$repo/os/$arch' >> /etc/pacman.d/mirrorlist
```
> Les miroirs configurés ici seront également automatiquement repris dans notre futur système.

### Installation minimaliste des paquets de base du système

```bash
pacstrap -K /mnt base linux linux-firmware intel-ucode btrfs-progs networkmanager dracut systemd-ukify
```

> Remplacez `intel-ucode` par `amd-ucode` si vous utilisez un processeur AMD.  
> Remplacez `linux` par `linux-lts` si vous préférez installer le noyau LTS.

### Générer le fichier fstab

```bash
genfstab -U /mnt >> /mnt/etc/fstab
```

---

## 5. Configuration de base du système

*"Entrer" dans le système :*

```bash
arch-chroot /mnt
```

### Fuseau horaire

*Adapter la localisation en conséquence :*

```bash
ln -sf /usr/share/zoneinfo/Europe/Paris /etc/localtime
```

### Langue et disposition du clavier

*Configuration de la langue ; ex. : fr_FR pour le français (France) ou en_US pour l'anglais (États-Unis) :*

```bash
export LOCALE="fr_FR.UTF-8"
```

```bash
sed -i "/$LOCALE/s/^#//" /etc/locale.gen
locale-gen

echo "LANG=$LOCALE" > /etc/locale.conf
```

*Clavier : ex. : us pour QWERTY, fr pour AZERTY :*

```bash
echo "KEYMAP=fr" > /etc/vconsole.conf
```

### Nom de la machine et configuration réseau

*Choisissez un nom pour votre machine :*

```bash
export hostname=maMachine
```

*Configuration :*

```bash
echo $hostname > /etc/hostname
echo "127.0.1.1	$hostname" >> /etc/hosts

systemctl enable NetworkManager
```

---

## 6. Unified Kernel Image (UKI)

Une **Unified Kernel Image (UKI)** regroupe le noyau, l'initramfs et les paramètres du noyau (ligne de commande) dans un seul fichier bootable `.efi`. Cela simplifie le Secure Boot (un seul fichier à signer) et supprime le besoin d'un bootloader séparé, comme GRUB. Cette méthode est également bien plus sécurisée que l'approche traditionnelle, où un bootloader signé lance séparément le noyau signé, l'initramfs (non signé) et la ligne de commande du noyau (modifiable via le bootloader).
Le choix ici est d'utiliser `kernel-install` en tant qu'orchestrateur, qui lance `dracut` pour générer l'initramfs, puis `systemd-ukify` pour générer l'UKI à l'aide de l'initramfs créé par `dracut` en mémoire, avec la ligne de commande et le noyau fournis par `kernel-install`.

### Configuration

*Optimisation `dracut` :*


```bash
echo "enhanced_cpio=\"yes\"" > /etc/dracut.conf.d/cpio.conf
```

> L'option 'enhanced_cpio' est une optimisation pour les systèmes de fichiers Copy-On-Write comme Btrfs.

*Désactivation des hooks `dracut` (ils seront remplacés par ceux de `kernel-install`) :*

```bash
mkdir -p /etc/pacman.d/hooks
ln -s /dev/null /etc/pacman.d/hooks/60-dracut-remove.hook 
ln -s /dev/null /etc/pacman.d/hooks/90-dracut-install.hook
```

*Empêcher `pacman` d'installer le microcode dans `/boot` (inutile avec `dracut` et UKI) :*

```bash
sed -e '/^#NoExtract/s/^#//' -e '/^NoExtract/s@$@ boot/*-ucode.img@' -i /etc/pacman.conf
```

*Ligne de commande du noyau :*

```bash
echo "rd.luks.name=$(blkid -s UUID -o value ${part})=cryptroot rd.luks.options=$(blkid -s UUID -o value ${part})=tpm2-device=auto,tpm2-measure-pcr=yes \
root=/dev/mapper/cryptroot rootflags=subvol=@ ro quiet splash" > /etc/kernel/cmdline
```

*Activer la génération d'UKI avec `kernel-install` :*

```bash
echo "layout=uki" > /etc/kernel/install.conf
```

*Nettoyage de `/boot` :*

```bash
rm -f /boot/{*-ucode.img,vmlinuz-linux*,initramfs-linux*.img}
```

### Générer l'UKI
```bash
kernel-install add-all
```
> L'UKI est présente dans `/boot/EFI/Linux`.

### Définir le mot de passe root

```bash
passwd
```

### Installation de `systemd-boot`
`systemd-boot` est entièrement compatible avec les UKIs, est inclus de base, et ne nécessite aucune configuration pour trouver les UKIs dans le bon répertoire. Par défaut aucun menu n'est affiché au démarrage — si vous avez plusieurs UKIs ou plusieurs OS installés, appuyez sur `espace` au démarrage pour afficher le menu.

*Activation des mises à jour automatiques du bootloader :*
```bash
systemctl enable systemd-boot-update.service
```

*Installation du bootloader :*
```bash
exit
```
```bash
bootctl install --esp-path=/mnt/boot
```

### Quitter le live CD

```bash
umount -R /mnt
cryptsetup close cryptroot
poweroff
```
*Débranchez la clé USB avant de redémarrer.*

---

## 7. Post-installation

*Se connecter en root. Utilisez un câble réseau ou configurez le Wi-Fi via `nmtui`.*

### SSH (optionnel, mais pratique pour le copier-coller)

```bash
pacman -S openssh

# PermitRootLogin yes (temporaire !)
echo "PermitRootLogin yes" > /etc/ssh/sshd_config.d/permitrootlogin.conf

systemctl start sshd
```

### Paramètres de base du système

*Clavier (us pour QWERTY) :*
```bash
localectl set-keymap fr
```

*Localisation (à adapter) :*

```bash
timedatectl set-timezone Europe/Paris
```

*Vérification :*

```bash
localectl
timedatectl
```

### Optimiser `pacman`

```bash
sed -e '/^#Color/s/^#//' -e '/^#VerbosePkgLists/s/^#//' \
	-e '/^# Misc options/a ILoveCandy' -i /etc/pacman.conf
```

Cela décommente et ajoute :

- `Color`  
- `VerbosePkgLists`  
- `ILoveCandy` (because you use arch BTW)

### Installation des outils essentiels et utiles

```bash
pacman -S base-devel git wget jq pacman-contrib fwupd wireless-regdb firewalld sbctl htop btop \
zram-generator udisks2 virt-what tuned tuned-ppd bash-completion pkgfile 7zip unrar zip unzip \
bzip3 ntfs-3g dosfstools udftools fzf eza bat nano vim rsync inetutils tmux man-db tldr \
plymouth logrotate efibootmgr chrony tpm2-tools
```

*Déconnectez-vous et reconnectez-vous pour activer la complétion Bash.*

### Configurer la synchronisation de l'horloge via Internet (`Chrony`)
`Chrony` est bien meilleur que `systemd-timesyncd`, surtout sur des systèmes de type laptop qui ne sont pas toujours connectés à Internet (voir [ici](https://wiki.archlinux.org/title/Laptop#Network_time_syncing)). Il est de plus largement plus utilisé par la plupart des distributions populaires.

*Configuration :*

```bash
# Désactive les serveurs par défaut
sed -E '/^(server|pool)/s/^/! /' -i /etc/chrony.conf

# Ajoute un serveur NTS (version sécurisée du protocole NTP)
sed '/^! pool/a server time.cloudflare.com iburst nts offline' -i /etc/chrony.conf
```
> Le mode offline permet à `chrony` de ne pas essayer de se connecter au serveur si `NetworkManager` lui indique qu'il n'y a aucune connexion.

*Configurer `NetworkManager` pour le mode offline :*
```bash
cp /usr/share/doc/chrony/examples/chrony.nm-dispatcher.onoffline /etc/NetworkManager/dispatcher.d/20-chrony-onoffline.sh
chmod 700 /etc/NetworkManager/dispatcher.d/20-chrony-onoffline.sh
```

*Activation :*
```bash
systemctl enable --now chronyd.service
```

*Vérification :*
```bash
timedatectl
```

### Tri des miroirs configurés par les plus rapides

```bash
mv /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.bak
rankmirrors /etc/pacman.d/mirrorlist.bak | tee /etc/pacman.d/mirrorlist
rm /etc/pacman.d/mirrorlist.bak
```

### Activation des services et timers utiles

*Nettoyage régulier du cache `pacman` :*
```bash
systemctl enable --now paccache.timer
```
*TRIM hebdomadaire (SSD uniquement) :*
```bash
systemctl enable --now fstrim.timer
```
*Rotation et nettoyage réguliers des journaux non gérés par systemd-journald :*
```bash
systemctl enable --now logrotate.timer
```
*Profils d'alimentation :*
```bash
systemctl enable --now tuned
systemctl enable --now tuned-ppd
```
*Retrouve les paquets des commandes introuvables :*
```bash
systemctl enable --now pkgfile-update.timer
pkgfile -u
```

### ZRAM (swap en RAM compressée)

*Configuration :*
```bash
nano /etc/systemd/zram-generator.conf
```

```ini
[zram0]
zram-size = ram / 2
compression-algorithm = zstd
priority = 100
```

*Activation :*

```bash
systemctl daemon-reload
systemctl start systemd-zram-setup@zram0.service
```

*Vérifications :*

```bash
zramctl
swapon -s
```

> ZRAM crée un espace de swap compressé en RAM — beaucoup plus rapide qu'un swap sur disque et réduit l'usure du SSD.

### DNS over TLS avec `systemd-resolved`

*Configuration :*

```bash
mkdir -p /etc/systemd/resolved.conf.d

# Désactiver le DNS de secours par défaut (évite les fuites)
printf "[Resolve]\nFallbackDNS=\n" > /etc/systemd/resolved.conf.d/fallback_dns.conf

# Active le DNS over TLS si disponible
printf "[Resolve]\nDNSOverTLS=opportunistic\n" > /etc/systemd/resolved.conf.d/dns_over_tls.conf

# Active le DNSSEC si disponible
printf "[Resolve]\nDNSSEC=allow-downgrade\n" > /etc/systemd/resolved.conf.d/dnssec.conf
```

*Activation :*

```bash
ln -sf ../run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
systemctl enable --now systemd-resolved
```

*Vérification :*
```bash
resolvectl
```

### Pare-feu `firewalld`

```bash
systemctl enable --now firewalld
```

### Mises à jour firmware `fwupd`

*Configurer :*
```bash
printf "[uefi_capsule]\nDisableShimForSecureBoot=true\n" >> /etc/fwupd/fwupd.conf
```

*Activation :*
```bash
systemctl restart fwupd
systemctl enable --now fwupd-refresh.timer
fwupdmgr refresh
```

### sudo

*Permettre aux membres du groupe `wheel` d'utiliser `sudo` :*

```bash
echo "%wheel ALL=(ALL:ALL) ALL" > /etc/sudoers.d/wheel
```
>`sudo` est conservé pour des raisons de compatibilité, mais veillez à l'utiliser le moins possible et préférez `run0`, qui est bien plus sécurisé.

## 8. Secure Boot + TPM 2.0

Le Secure Boot garantit que seule une UKI signée avec vos propres clés peut démarrer. Le TPM 2.0 permet de déverrouiller automatiquement le disque chiffré si la chaîne de démarrage est intacte (via les PCRs).

### Créer et enrôler les clés pour le Secure Boot

*Création des clés :*

```bash
sbctl create-keys
```

*Enrôlement des clés :*

```bash
# -m conserve les clés Microsoft (⚠️ OBLIGATOIRE, sinon risque de brick)
sbctl enroll-keys -m
```
> En cas d'erreur : redémarrez et passez le Secure Boot en "Setup Mode" dans l'UEFI. Ensuite refaites la commande.

### Signer les fichiers .efi

*fwupd (mises à jour du firmware) :*
```bash
sbctl sign -s -o /usr/lib/fwupd/efi/fwupdx64.efi.signed /usr/lib/fwupd/efi/fwupdx64.efi
```
*systemd-boot (bootloader) :*
```bash
sbctl sign -s -o /usr/lib/systemd/boot/efi/systemd-bootx64.efi.signed /usr/lib/systemd/boot/efi/systemd-bootx64.efi
```
*UKI :*
```bash
sbctl sign /boot/EFI/Linux/*
```
*Mise à jour de systemd-boot signé :*
```bash
cp /usr/lib/systemd/boot/efi/systemd-bootx64.efi.signed /boot/EFI/BOOT/BOOTX64.efi
cp /usr/lib/systemd/boot/efi/systemd-bootx64.efi.signed /boot/EFI/systemd/systemd-bootx64.efi
```
*Redémarrer :*
```bash
systemctl reboot --firmware-setup
```
>Dans le BIOS, activez le Secure Boot si ce n'est pas déjà fait. Vérifiez que le TPM 2.0 est également activé.

### Ajouter la liste de révocations (DBX)
Le problème avec `sbctl` est qu'il n'ajoute aucune liste de révocation (DBX), ce qui représente un sérieux problème de sécurité. Sans DBX, le Secure Boot continue de fonctionner, mais ne protège plus contre des binaires signés connus comme compromis. Restaurer la DBX est donc fortement recommandé : il faut l'installer une fois manuellement, puis `fwupd` pourra la mettre à jour lors des prochaines mises à jour disponibles.

*Installer les outils nécessaires :*
```bash
pacman -S cabextract dbxtool
```

*Télécharger la liste DBX à jour ([ici](https://fwupd.org/lvfs/devices/com.microsoft.dbx.x64.firmware)) :*

```bash
mkdir ~/dbx && \
wget -O ~/dbx/dbx.cab https://fwupd.org/downloads/7178302fa23fcb875e7540900e299fb30a76758663efb7e1c56edc25cd3f316a-DBXUpdate-20250902-x64.cab
```
> Veillez à mettre à jour le lien si une version plus récente que 2025-09-02 existe.

*Décompresser l'archive :*

```bash
cabextract ~/dbx/dbx.cab -d ~/dbx
```

*Installer la liste :*

```bash
dbxtool --apply ~/dbx/DBXUpdate-*.x64.bin
```

*Redémarrage pour prise en compte :*

```bash
reboot
```

*Nettoyage :*
```bash
rm -rf ~/dbx && pacman -Rsn cabextract dbxtool
```
### Configurer `kernel-install` pour signer automatiquement les UKIs générés

Nous allons créer deux clés supplémentaires qui permettront de sécuriser encore davantage nos UKIs et le déverrouillage du TPM 2.0. Cela permettra au TPM de déverrouiller la partition UNIQUEMENT si les UKIs sont signées avec nos propres clés et que la chaîne de démarrage est intacte (PCR 11 policies). L'intérêt d'avoir deux clés séparées est que le TPM ne déverrouille la partition qu'au démarrage, et non pendant que le système tourne (phase initrd).

*Création des clés :*

```bash
ukify genkey \
        --pcr-private-key=/etc/systemd/tpm2-pcr-private-key.pem \
        --pcr-public-key=/etc/systemd/tpm2-pcr-public-key.pem
ukify genkey \
        --pcr-private-key=/etc/systemd/tpm2-pcr-private-key-initrd.pem \
        --pcr-public-key=/etc/systemd/tpm2-pcr-public-key-initrd.pem
```

*Configuration pour la signature automatique des UKIs :*

```bash
nano /etc/kernel/uki.conf
```

```ini
[UKI]
SecureBootSigningTool=systemd-sbsign
SignKernel=true
PCRBanks=sha256
SecureBootPrivateKey=/var/lib/sbctl/keys/db/db.key
SecureBootCertificate=/var/lib/sbctl/keys/db/db.pem

[PCRSignature:all]
PCRPrivateKey=/etc/systemd/tpm2-pcr-private-key.pem
PCRPublicKey=/etc/systemd/tpm2-pcr-public-key.pem

[PCRSignature:initrd]
Phases=enter-initrd
PCRPrivateKey=/etc/systemd/tpm2-pcr-private-key-initrd.pem
PCRPublicKey=/etc/systemd/tpm2-pcr-public-key-initrd.pem
```

*Désactivation du hook `sbctl` (plus nécessaire) :*
```bash
ln -sf /dev/null /usr/lib/kernel/install.d/91-sbctl.install
```

*Re-génération de l'UKI signée automatiquement :*

```bash
kernel-install add-all
```

*Redémarrez pour préparer l'enrôlement du TPM 2.0 :*
```bash
reboot
```

### Enrôler le TPM 2.0

*Création d'une clé de récupération :*
```bash
systemd-cryptenroll --wipe-slot all /dev/<partition_chiffrée> --recovery-key
```
> ⚠️ AVERTISSEMENT CRITIQUE : votre passphrase est supprimée et remplacée par cette clé de récupération, bien plus sécurisée. Stockez-la en lieu sûr et ne la perdez surtout pas !

*Enrôlement du TPM 2.0 :*
```bash
systemd-cryptenroll --wipe-slot tpm2 --tpm2-device auto \
                    --tpm2-public-key /etc/systemd/tpm2-pcr-public-key-initrd.pem \
                    --tpm2-pcrs=7 \
                    /dev/<partition_chiffrée>
```
> Après cette étape, le disque se déverrouille automatiquement au démarrage tant que la chaîne de démarrage est valide et que le Secure Boot est actif avec sa base de clés inchangée. En cas de modification du Secure Boot ou d'UKI incorrectement signée, la clé de récupération sera demandée et il faudra refaire cette commande.

Depuis une version très récente de `systemd` (v260), une nouvelle option (Pinning LUKS volume) permet de déverrouiller une partition uniquement si elle correspond au bon conteneur `LUKS`, évitant ainsi toute attaque basée sur la copie des métadonnées de la partition.

*Modification de la ligne de commande pour le Pinning LUKS volume :*
```bash
LUKS_VOLUME=$(grep volume-key /run/log/systemd/tpm2-measure.log | jq -r --seq '.digests[].digest')
sed -i "s|tpm2-measure-pcr=yes|fixate-volume-key=$LUKS_VOLUME|" /etc/kernel/cmdline
```

*Re-générer l'UKI pour prise en compte :*
```bash
kernel-install add-all
```

*Redémarrage — profitez du déverrouillage automatique avec TPM 2.0 :) :*
```bash
reboot
```

---

## 9. Utilisateur et AUR Helper

### Créer un utilisateur

```bash
useradd -m -G wheel <nom_utilisateur>
passwd <nom_utilisateur>
```

### Suppression de la connexion root via SSH

```bash
sed -E -e '/^#PermitRootLogin/s/^#//' -e '/^PermitRootLogin/s/(yes|prohibit-password)/no/' \
	-i /etc/ssh/sshd_config
rm -f /etc/ssh/sshd_config.d/permitrootlogin.conf

systemctl reload sshd
```

*Connectez-vous désormais avec votre utilisateur :*

```bash
su - <nom_utilisateur>
```

### AUR Helper (yay)

```bash
git clone https://aur.archlinux.org/yay-bin.git
cd yay-bin
makepkg -sirc
cd ..
rm -rf yay-bin
```

> `yay` permet d'installer, de supprimer et de mettre à jour tous les paquets du dépôt d'Arch (en utilisant `pacman`), mais inclut également tous les paquets disponibles dans l'[AUR](https://aur.archlinux.org/).

---

## 10. Hook Kernel-install

Installation d'un hook qui appellera `kernel-install` automatiquement lorsque le noyau ou des paquets nécessitant de regénérer l'initramfs seront mis à jour. Autrement dit, l'UKI sera regénérée automatiquement lorsque cela est nécessaire.

*Installation du hook :*

```bash
yay -S pacman-hook-kernel-install
```

## 11. Environnement de bureau

### Pilotes graphiques et Wayland

<details>
  <summary>Intel</summary>

```bash
yay -S wayland wayland-utils xorg-xwayland wl-clipboard mesa vulkan-intel intel-media-driver
```

</details>

<details>
  <summary>AMD / Radeon</summary>

```bash
yay -S wayland wayland-utils xorg-xwayland wl-clipboard mesa vulkan-radeon
```

</details>

<details>
  <summary>Nvidia</summary>

```bash
# Nvidia – minimum RTX 2000 / GTX 1600
# Remplacez `nvidia-open` par `nvidia-open-lts` si vous utilisez le noyau LTS
yay -S wayland wayland-utils xorg-xwayland wl-clipboard nvidia-open nvidia-utils libva-nvidia-driver

# Nvidia – GTX 1000 ou inférieure
yay -S wayland wayland-utils xorg-xwayland wl-clipboard nvidia-580xx-dkms nvidia-580xx-utils libva-nvidia-driver

sudo reboot
```

</details>

### Audio (PipeWire)

```bash
yay -S pipewire pipewire-audio pipewire-alsa pipewire-pulse pipewire-jack pipewire-v4l2 pipewire-zeroconf sof-firmware
```

### Codecs multimédia

```bash
yay -S ffmpeg gstreamer gst-libav gst-plugins-bad gst-plugins-base gst-plugins-good gst-plugins-ugly libheif gd openh264
```

### Installation de l'environnement de bureau

Veuillez choisir entre KDE Plasma ou GNOME. Vous pouvez consulter des illustrations sur Internet ou des vidéos YouTube pour vous aider à faire votre choix. KDE Plasma est plus proche de Windows, tandis que GNOME se rapproche de l'interface de macOS.

<details>
  <summary>KDE Plasma</summary>

```bash
yay -S plasma-meta konsole dolphin ark kdenetwork-filesharing firefox usbutils gnu-free-fonts \
noto-fonts qt6-multimedia-ffmpeg bluez bluez-utils kgamma kwrite kalk okular elisa mpv krdc \
kwalletmanager libvncserver gwenview filelight partitionmanager ksystemlog khelpcenter kio-admin \
dolphin-plugins qbittorrent ffmpegthumbs kolourpaint yt-dlp networkmanager-openvpn qrca \
networkmanager-openconnect imagemagick firewall-config

sudo systemctl enable bluetooth
sudo systemctl enable plasmalogin
sudo reboot
```

</details>

<details>
  <summary>GNOME</summary>
Les paquets listés dans le `grep` sont exclus, car jugés inutiles à mon sens. Si un paquet vous intéresse dans cette liste d'exclusion, retirez-le simplement.

```bash
yay -S $(yay -Sgq gnome | grep -Ev "epiphany|snapshot|gnome-calendar|gnome-contacts|gnome-maps|gnome-music|gnome-system-monitor|gnome-weather|gnome-clocks|gnome-tour|gnome-console|rygel|sushi") \
firefox ptyxis resources usbutils gnu-free-fonts noto-fonts bluez bluez-utils mpv libvncserver flatpak decoder \
fragments yt-dlp networkmanager-openvpn networkmanager-openconnect imagemagick gnome-tweaks firewall-config collision

sudo systemctl enable bluetooth
sudo systemctl enable gdm
sudo reboot
```

</details>

### Imprimante et mDNS (si nécessaire)

*Installation :*

```bash
yay -S cups ghostscript system-config-printer nss-mdns
```

*Configuration :*

```bash
printf "[Resolve]\nMulticastDNS=no\n" | sudo tee /etc/systemd/resolved.conf.d/mdns.conf
sudo systemctl restart systemd-resolved

sudo sed -i '/^hosts/s/^.*/hosts : mymachines mdns_minimal [NOTFOUND=return] resolve [!UNAVAIL=return] files myhostname dns/' /etc/nsswitch.conf
```

*Activation :*

```bash
sudo systemctl enable --now avahi-daemon
sudo systemctl enable --now cups.socket
```

### Polices
Installation des polices Microsoft (Arial, Times New Roman, ...) et ajout d'une police Nerd pour le terminal afin d'afficher des icônes.

```bash
yay -S ttf-firacode-nerd ttf-ms-fonts
```

*Configurez ensuite la police Nerd dans les paramètres du terminal.*

---

## 12. Snapshots Btrfs avec `Snapper`

`Snapper` gère automatiquement des snapshots avant et après chaque mise à jour `pacman` (via `snap-pac`) et de manière périodique, permettant un rollback rapide en cas de problème.

*Installation :*

```bash
yay -S snapper snap-pac
```

*Reconfigurer le sous-volume `.snapshots` (Snapper recrée le sien) :*

```bash
sudo umount /.snapshots
sudo rm -r /.snapshots
sudo snapper -c root create-config /
sudo btrfs subvolume delete /.snapshots
sudo mkdir /.snapshots
sudo chmod 750 /.snapshots
sudo mount -a
```

### Configurer les limites de snapshots

```bash
sudo sed -e '/NUMBER_LIMIT/s/".*"/"25"/' \
		 -e '/TIMELINE_LIMIT_HOURLY/s/".*"/"0"/' \
		 -e '/TIMELINE_LIMIT_DAILY/s/".*"/"6"/' \
		 -e '/TIMELINE_LIMIT_WEEKLY/s/".*"/"3"/' \
		 -e '/TIMELINE_LIMIT_MONTHLY/s/".*"/"0"/' \
		 -e '/TIMELINE_LIMIT_QUARTERLY/s/".*"/"0"/' \
		 -e '/TIMELINE_LIMIT_YEARLY/s/".*"/"0"/' \
		 -i /etc/snapper/configs/root
```

> Vous pouvez adapter le nombre maximal de snapshots pour chaque période selon vos besoins.

### Empêcher `Snapper` de réinstaller sa tâche cron (évite le conflit avec le timer systemd)

```bash
sudo sed -e '/^#NoExtract/s/^#//' -e '/^NoExtract/s@$@ etc/cron.hourly/snapper@' -i /etc/pacman.conf

sudo rm -f /etc/cron.hourly/snapper
```

### Configurer le timer des snapshots (quotidien)

```bash
sudo systemctl edit snapper-timeline.timer --drop-in=snapper-timeline-daily.timer
```

```ini
[Timer]
OnCalendar=
OnCalendar=daily
Persistent=true
AccuracySec=5m
```

### Activation des services de snapshots et du nettoyage automatique

```bash
sudo systemctl enable --now snapper-timeline.timer
sudo systemctl enable --now snapper-cleanup.timer
```

---

## 13. Configurations finales

### Désactiver le compte root

*Une fois certain que votre utilisateur dispose bien des droits `run0` et `sudo` :*

```bash
run0 passwd -dl root
```

---

## 14. Vérifications finales

*Vérifier qu'il n'y a pas d'erreurs critiques dans le noyau :*

```bash
run0 dmesg | grep -i failed
```

*Vérifier qu'aucun service n'a planté :*

```bash
systemctl --failed
```

---

## 15. Résumé de la stack

| Composant | Choix |
|-----------|-------|
| Chiffrement | LUKS2 |
| Système de fichiers | Btrfs + sous-volumes + compression zstd |
| Noyau | linux ou linux-lts |
| Bootloader | UKI (systemd-boot et systemd-stub) |
| Secure Boot | sbctl |
| Déverrouillage auto | TPM 2.0 (PCR 7 et 11 policies) |
| Init | systemd |
| Réseau | NetworkManager |
| DNS | systemd-resolved (DNS over TLS) |
| Audio | PipeWire |
| Compositeur | Wayland |
| Bureau | KDE Plasma ou GNOME |
| Snapshots | Snapper + snap-pac |
| AUR helper | yay |
