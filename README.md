# Guide d’installation d’Arch Linux moderne et sécurisé

[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![Arch Linux](https://img.shields.io/badge/OS-Arch%20Linux-blue?logo=arch-linux)](https://archlinux.org/)
[![Last Commit](https://img.shields.io/github/last-commit/Mauro177/arch-modern-install)](https://github.com/Mauro177/arch-modern-install/commits/main)
[![Issues](https://img.shields.io/github/issues/Mauro177/arch-modern-install)](https://github.com/Mauro177/arch-modern-install/issues)

Installation complète d’Arch Linux incluant le chiffrement LUKS2 avec déverrouillage automatique via TPM 2.0, Unified Kernel Image (UKI), Secure Boot, le système de fichiers Btrfs, ainsi qu’un environnement de bureau KDE Plasma ou GNOME.

## Table des matières
1. [Préparation du live CD](#1-préparation-du-live-cd)
2. [Partitionnement et chiffrement du disque](#2-partitionnement-et-chiffrement-du-disque)
3. [Configuration Btrfs et sous-volumes](#3-configuration-btrfs-et-sous-volumes)
4. [Installation minimale du système de base](#4-installation-minimale-du-système-de-base)
5. [Configuration de base du système (via chroot)](#5-configuration-de-base-du-système-via-chroot)
6. [Unified Kernel Image (UKI)](#6-unified-kernel-image-uki)
7. [Post-installation](#7-post-installation)
8. [Secure Boot + TPM 2.0](#8-secure-boot--tpm-20)
9. [Utilisateur et AUR Helper](#9-utilisateur-et-aur-helper)
10. [Environnement de bureau](#10-environnement-de-bureau)
11. [Snapshots Btrfs avec `Snapper`](#11-snapshots-btrfs-avec-snapper)
12. [Configurations finales](#12-configurations-finales)
13. [Vérifications finales](#13-vérifications-finales)
14. [Résumé de la stack](#14-résumé-de-la-stack)

---

## Vue d'ensemble

Ce guide met en place une installation d’Arch Linux moderne, sécurisée et minimaliste :

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
ping ping.archlinux.org

# Configurer le fuseau horaire et activer NTP pour obtenir la date et l'heure correctes
timedatectl set-timezone Europe/Brussels
timedatectl set-ntp true
# Vérification
date
```

---

## 2. Partitionnement et chiffrement du disque

### Identifier le disque cible pour l’installation d’Arch Linux

```bash
fdisk -l
```

*Repérez votre disque cible (ex: `/dev/nvme0n1`, `/dev/sda`).*

### Partitionner le disque

```bash
# Stocker dans une variable le disque cible de l’installation (ex. /dev/nvme0n1, /dev/sda)
# Cela facilitera la suite du guide pour copier-coller les commandes
export disk=monDisque

cfdisk $disk
```

*Créez un schéma GPT avec deux partitions :*

| Partition | Taille | Type | Point de montage |
|-----------|--------|------|------------------|
| `1` | 512M | EFI System | `/boot/efi` |
| `2` | Reste | Linux filesystem | `/` (chiffrée) |

> ⚠️ **Attention** si une partition EFI existe déjà (dual boot), ne la reformatez pas.

### Formater la partition EFI

```bash
# ⚠️ UNIQUEMENT si la partition EFI est nouvelle
mkfs.fat -F32 ${disk}1
```

### Chiffrement LUKS2 de la partition racine

```bash
# Initialiser la partition chiffrée (vous devrez choisir une passphrase)
# ⚠️ ATTENTION : veillez à remplacer le '2' si vous installez Arch Linux sur une autre partition
# (voir `fdisk -l $disk`)
cryptsetup -v luksFormat ${disk}2

# Déchiffrer la partition (la rendre accessible sous /dev/mapper/root)
cryptsetup open ${disk}2 root
```

> LUKS2 chiffre l’intégralité de la partition. Sans la passphrase (ou sans TPM 2.0 configuré ultérieurement), les données restent illisibles.

---

## 3. Configuration Btrfs et sous-volumes

Btrfs permet de créer des **sous-volumes** indépendants sur une même partition, ce qui est essentiel pour les snapshots ciblés et pour gérer finement les options de montage.

### Créer le système de fichiers et les sous-volumes

```bash
# Formater la partition en Btrfs et la monter dans /mnt
mkfs.btrfs -L root /dev/mapper/root
mount /dev/mapper/root /mnt

# Créer les sous-volumes
btrfs subvolume create /mnt/@           # Racine du système
btrfs subvolume create /mnt/@home       # Données utilisateur
btrfs subvolume create /mnt/@log        # Journaux système
btrfs subvolume create /mnt/@cache      # Cache
btrfs subvolume create /mnt/@tmp        # Fichiers temporaires
btrfs subvolume create /mnt/@vms        # Images de machines virtuelles (si besoin)
btrfs subvolume create /mnt/@snapshots  # Snapshots Snapper

# Désactiver le Copy-on-Write pour les images de VMs (meilleure performances I/O)
chattr +C /mnt/@vms

umount /mnt
```

### Monter les sous-volumes Btrfs et la partition EFI

```bash
# Sous-volumes Btrfs
mount -o compress=zstd,subvol=@ /dev/mapper/root /mnt
mount --mkdir -o compress=zstd,subvol=@home      /dev/mapper/root /mnt/home
mount --mkdir -o compress=zstd,subvol=@log       /dev/mapper/root /mnt/var/log
mount --mkdir -o noatime,compress=zstd,subvol=@cache /dev/mapper/root /mnt/var/cache
mount --mkdir -o noatime,compress=zstd,subvol=@tmp   /dev/mapper/root /mnt/var/tmp
mount --mkdir -o noatime,subvol=@vms             /dev/mapper/root /mnt/var/lib/libvirt/images
mount --mkdir -o compress=zstd,subvol=@snapshots /dev/mapper/root /mnt/.snapshots

# Partition EFI
mount --mkdir ${disk}1 /mnt/boot/efi
```

> `noatime` désactive la mise à jour de l'heure d'accès aux fichiers — cela évite des écritures inutiles pour les données de cache ou temporaires.  
> `compress=zstd` active la compression, ce qui réduit l'utilisation du disque et peut accélérer la lecture ou l'écriture dans certains cas.

---

## 4. Installation minimale du système de base

### Configurer les miroirs les plus rapides

```bash
reflector --verbose --sort rate -l 5 -p https --save /etc/pacman.d/mirrorlist
```

### Installer les paquets de base du système

```bash
pacstrap -K /mnt base linux linux-firmware intel-ucode btrfs-progs efibootmgr networkmanager
```

> Remplacez `intel-ucode` par `amd-ucode` si vous utilisez un processeur AMD.  
> Remplacez `linux` par `linux-lts` si vous préférez installer le noyau LTS.

### Générer le fichier fstab

```bash
genfstab -U /mnt >> /mnt/etc/fstab
```

---

## 5. Configuration de base du système (via chroot)

```bash
arch-chroot /mnt
```

### Fuseau horaire et configuration de l’horloge

```bash
# Adapter la localisation en conséquence
ln -sf /usr/share/zoneinfo/Europe/Brussels /etc/localtime
hwclock --systohc
```

### Langue et disposition du clavier

```bash
# À adapter en conséquence
# ex. : fr_FR pour le français (France) ou en_US pour l’anglais (États-Unis)
sed -i '/fr_BE.UTF-8/s/^#//' /etc/locale.gen
locale-gen
echo "LANG=fr_BE.UTF-8" > /etc/locale.conf

# (us) pour QWERTY
echo "KEYMAP=fr" > /etc/vconsole.conf
```

### Nom de la machine et configuration réseau

```bash
# Choisissez un nom pour votre machine
export hostname=maMachine

echo $hostname > /etc/hostname
echo "127.0.1.1		   $hostname" >> /etc/hosts

# Désactiver systemd-networkd (géré par NetworkManager)
systemctl mask systemd-networkd
systemctl enable NetworkManager
```

### Hooks `mkinitcpio` (initramfs)

```bash
echo "HOOKS=(base systemd keyboard autodetect microcode modconf kms sd-vconsole block sd-encrypt filesystems fsck)" \
  > /etc/mkinitcpio.conf.d/hooks.conf
```

> `sd-encrypt` gère le déverrouillage LUKS au démarrage (équivalent systemd du hook `encrypt`).  
> `sd-vconsole` charge la disposition du clavier tôt au démarrage (utile pour saisir la passphrase).

### Paramètres du noyau (ligne de commande)

```bash
# Récupérer l'UUID de la partition chiffrée et l'intégrer dans la ligne de commande du noyau
# Remplacez le '2' si Arch Linux est installée sur une autre partition
echo "rd.luks.name=$(blkid -s UUID -o value ${disk}2)=root root=/dev/mapper/root \
rootflags=subvol=/@ rw quiet nowatchdog" > /etc/kernel/cmdline

# Désactiver le watchdog (inutile sur une machine non serveur)
echo "blacklist iTCO_wdt" > /etc/modprobe.d/nowatchdog.conf
echo "blacklist sp5100_tco" >> /etc/modprobe.d/nowatchdog.conf
```

---

## 6. Unified Kernel Image (UKI)

Une **Unified Kernel Image (UKI)** regroupe le noyau, l’initramfs et les paramètres du noyau dans un seul fichier bootable `.efi`. Cela simplifie le Secure Boot (un seul fichier à signer) et supprime le besoin d’un bootloader séparé, comme GRUB. Cette méthode est également bien plus sécurisée que l’approche traditionnelle, où un bootloader signé lance séparément le noyau signé, l’initramfs (non signé) et la ligne de commande du noyau (modifiable via le bootloader).

### Préparer le preset `mkinitcpio`

```bash
# Remplacez linux.preset par linux-lts.preset si vous utilisez le noyau LTS
sed -e '/default_image/s/^/#/' -e '/default_uki/s/^#//' -e '/default_uki/s@"/@"/boot/@'   \
	-i /etc/mkinitcpio.d/linux.preset
```

> Dans ce fichier, on **commente** la ligne `default_image` et **décommente** `default_uki` pour générer des UKI au lieu des initramfs classiques. On ajoute également `/boot` au début du chemin de `default_uki` pour l’adapter au bon emplacement.

### Créer les répertoires et générer l’UKI

```bash
mkdir -p /boot/efi/EFI/{Linux,BOOT}

# Nettoyer les anciens initramfs
rm -f /boot/initramfs-*

# Générer le fichier .efi [UKI]
mkinitcpio -P
```

### Enregistrer l’entrée de démarrage EFI dans le micrologiciel du PC

```bash
# arch-linux-lts.efi si vous utilisez le noyau LTS
efibootmgr --create --disk $disk --part 1 --label "Arch Linux" \
  --loader '\EFI\Linux\arch-linux.efi' --unicode
```

### Définir le mot de passe root et quitter le live CD

```bash
passwd
exit

umount -R /mnt
cryptsetup close root
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

# Récupérez l'adresse IP avec (ip a), car elle aura probablement changé
systemctl start sshd
```

### Paramètres de base du système

```bash
# (us) pour QWERTY
localectl set-keymap fr

# À adapter
timedatectl set-timezone Europe/Brussels
timedatectl set-ntp true
hwclock --systohc

# Vérification
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
pacman -S gcc python which pkgconf automake autoconf bison debugedit fakeroot texinfo \
libtool make patch git wget sudo-rs pacman-contrib reflector wireless-regdb ufw htop \
btop zram-generator udisks2 fwupd virt-what tuned tuned-ppd sbctl bash-completion \
pkgfile 7zip unrar zip unzip bzip3 ntfs-3g dosfstools udftools perl-locale-gettext \
fzf eza bat nano vim rsync inetutils tmux man-db tldr

# Créer des liens symboliques pour vi/editor
ln -s /usr/bin/vim /usr/local/bin/vi
# Remplacez par nano si vous préférez cet éditeur
ln -s /usr/bin/nano /usr/local/bin/editor
```

*Déconnectez-vous et reconnectez-vous pour activer la complétion Bash.*

### Configurer automatiquement les miroirs les plus rapides pour `pacman`

```bash
sed '/^--sort/s/[^ ]*$/rate/' -i /etc/xdg/reflector/reflector.conf

# Mise à jour hebdomadaire des miroirs
systemctl enable --now reflector.timer
```

### Activation des services et timers utiles

```bash
# Nettoyage régulié du cache pacman
systemctl enable --now paccache.timer
# TRIM hebdomadaire (SSD uniquement)
systemctl enable --now fstrim.timer
# Profils d'alimentation dynamiques
systemctl enable --now tuned
systemctl enable --now tuned-ppd
# Retrouve les paquets des commandes introuvables
systemctl enable --now pkgfile-update.timer
pkgfile -u
```

### ZRAM (swap en RAM compressée)

```bash
nano /etc/systemd/zram-generator.conf
```

```ini
[zram0]
zram-size = ram / 2
compression-algorithm = zstd
priority = 100
```

```bash
systemctl daemon-reload
systemctl start systemd-zram-setup@zram0.service

# Vérifier
zramctl
swapon -s
```

> ZRAM crée un espace de swap compressé en RAM — beaucoup plus rapide qu’un swap sur disque et réduit l’usure du SSD.

### DNS over TLS avec `systemd-resolved`

```bash
mkdir -p /etc/systemd/resolved.conf.d

# Désactiver le DNS de secours par défaut (évite les fuites)
printf "[Resolve]\nFallbackDNS=\n" > /etc/systemd/resolved.conf.d/fallback_dns.conf

# Active le DNS over TLS si disponible
printf "[Resolve]\nDNSOverTLS=opportunistic\n" > /etc/systemd/resolved.conf.d/dns_over_tls.conf

# Active le DNSSEC si disponible
printf "[Resolve]\nDNSSEC=allow-downgrade\n" > /etc/systemd/resolved.conf.d/dnssec.conf

# Active systemd-resolved
ln -sf ../run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
systemctl enable --now systemd-resolved

# Vérifier la configuration DNS
resolvectl
```

### Pare-feu UFW

```bash
systemctl enable --now ufw
ufw default deny incoming
ufw default allow outgoing
ufw logging off

# Limite les tentatives de bruteforce SSH (si vous utilisez SSH)
ufw limit ssh

ufw enable

# Vérifier
ufw status verbose
```

### Mises à jour du firmware `fwupd`

```bash
printf "[uefi_capsule]\nDisableShimForSecureBoot=true\n" >> /etc/fwupd/fwupd.conf

systemctl restart fwupd
systemctl enable --now fwupd-refresh.timer
fwupdmgr refresh
```

### Configurer sudo-rs

```bash
visudo-rs
```
*Ajouter :*
```ini
# Keep your editor when running visudo
Defaults!/usr/bin/visudo-rs env_keep += "SUDO_EDITOR EDITOR VISUAL"
# The same if you choose to symlink visudo-rs to visudo
Defaults!/usr/local/bin/visudo env_keep += "SUDO_EDITOR EDITOR VISUAL"

# Sanitize your path
Defaults secure_path="/usr/local/sbin:/usr/local/bin:/usr/bin"
 
# "root", and all members of the group "wheel" can run any command after providing a password.
root ALL=(ALL:ALL) ALL
%wheel ALL=(ALL:ALL) ALL
```

*Ajouter le PAM pour sudo :*
```bash
wget --output-document=/etc/pam.d/sudo \
https://gitlab.archlinux.org/archlinux/packaging/packages/sudo/-/raw/main/sudo.pam

ln -s /etc/pam.d/sudo /etc/pam.d/sudo-i
```

*Liens symboliques pour ne pas devoir écrire le -rs*
```bash
ln -s /usr/bin/sudo-rs /usr/local/bin/sudo
ln -s /usr/bin/su-rs /usr/local/bin/su
ln -s /usr/bin/visudo-rs /usr/local/bin/visudo
ln -s /usr/bin/sudoedit-rs /usr/local/bin/sudoedit

# Vérification
sudo --version
```

> `sudo-rs` est une réimplémentation de `sudo`, écrite en Rust et plus sécurisée.

---

## 8. Secure Boot + TPM 2.0

Le Secure Boot garantit que seule une UKI signée avec vos propres clés peut démarrer. Le TPM 2.0 permet de déverrouiller automatiquement le disque chiffré si la chaîne de démarrage est intacte (PCR 0 + 7).

### Créer et enrôler les clés pour le Secure Boot

```bash
sbctl create-keys

# -m conserve les clés Microsoft (⚠️ OBLIGATOIRE, sinon risque de brick)
sbctl enroll-keys -m
```

> En cas d’erreur : redémarrez et passez le Secure Boot en "Setup Mode" dans l’UEFI.

### Signer les fichiers `.efi`

```bash
sbctl sign -s /boot/efi/EFI/Linux/arch-linux.efi

# Pour `fwupd` (mises à jour du firmware)
sbctl sign -s -o /usr/lib/fwupd/efi/fwupdx64.efi.signed /usr/lib/fwupd/efi/fwupdx64.efi

# Régénérer l’UKI (le hook `sbctl` la re-signera automatiquement)
mkinitcpio -P

systemctl reboot --firmware-setup
```

*Dans le BIOS, **activez** le Secure Boot si ce n’est pas déjà fait. **Vérifiez** que le TPM 2.0 est également activé.*

### Enrôler le TPM 2.0

```bash
# Vérifier le Secure Boot
sbctl status
# Lister les TPM disponibles
systemd-cryptenroll --tpm2-device=list

# Lier le déverrouillage LUKS2 au TPM 2.0 (PCR 0 = firmware, PCR 7 = Secure Boot)
systemd-cryptenroll --tpm2-device=auto --tpm2-pcrs=0+7 /dev/PartitionChiffrée

reboot
```

> Après cette étape, le disque se déverrouille automatiquement au démarrage tant que la chaîne de démarrage est valide (Secure Boot actif avec sa base de clés inchangée et firmware du PC non altéré). En cas de modification du firmware ou du Secure Boot, la passphrase sera demandée. **IMPORTANT** : les mises à jour du BIOS (UEFI) modifient le firmware, donc la passphrase sera à nouveau demandée, et il faudra relier le déverrouillage LUKS2 au TPM 2.0 avec la commande ci-dessus.

---

## 9. Utilisateur et AUR Helper

### Créer un utilisateur et sécuriser le compte root

```bash
useradd -m -G users,wheel {user}
passwd {user}

# Désactiver l'accès SSH via root
sed -E -e '/^#PermitRootLogin/s/^#//' -e '/^PermitRootLogin/s/(yes|prohibit-password)/no/' \
	-i /etc/ssh/sshd_config
rm -f /etc/ssh/sshd_config.d/permitrootlogin.conf

reboot
```

*Connectez-vous désormais avec votre utilisateur et utilisez `sudo`.*

### AUR Helper (yay)

```bash
git clone https://aur.archlinux.org/yay-bin.git
cd yay-bin
makepkg -sirc
cd ..
rm -rf yay-bin

# Vérifier
yay
```

> Permet d’installer, de supprimer et de mettre à jour tous les paquets du dépôt d’Arch (en utilisant `pacman`), mais inclut également tous les paquets disponibles dans l’[AUR](https://aur.archlinux.org/).

---

## 10. Environnement de bureau

### Pilotes graphiques et Wayland

<details>
  <summary>Intel</summary>

```bash
yay -S wayland wayland-utils xorg-xwayland wl-clipboard mesa vulkan-intel intel-media-driver libva-utils
```

</details>

<details>
  <summary>AMD / Radeon</summary>

```bash
yay -S wayland wayland-utils xorg-xwayland wl-clipboard mesa vulkan-radeon libva-utils
```

</details>

<details>
  <summary>Nvidia</summary>

```bash
# Nvidia – minimum RTX 2000 / GTX 1600
# Remplacez `nvidia-open` par `nvidia-open-lts` si vous utilisez le noyau LTS
yay -S wayland wayland-utils xorg-xwayland wl-clipboard nvidia-open nvidia-utils libva-nvidia-driver libva-utils

# Nvidia – GTX 1000 ou inférieure
yay -S wayland wayland-utils xorg-xwayland wl-clipboard nvidia-580xx-dkms nvidia-580xx-utils libva-nvidia-driver libva-utils

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

### Installation de l’environnement de bureau

Veuillez choisir entre KDE Plasma ou GNOME. Vous pouvez consulter des illustrations sur Internet ou des vidéos YouTube pour vous aider à faire votre choix. KDE Plasma est plus proche de Windows, tandis que GNOME se rapproche de l’interface de macOS.

<details>
  <summary>KDE Plasma</summary>

```bash
yay -S plasma-meta konsole dolphin ark kdenetwork-filesharing firefox usbutils gnu-free-fonts \
noto-fonts qt6-multimedia-ffmpeg bluez bluez-utils kgamma kwrite kalk okular elisa mpv krdc \
kwalletmanager libvncserver gwenview filelight partitionmanager ksystemlog khelpcenter kio-admin \
dolphin-plugins qbittorrent ffmpegthumbs kolourpaint yt-dlp networkmanager-openvpn \
networkmanager-openconnect imagemagick

sudo systemctl enable bluetooth
sudo systemctl enable plasmalogin
sudo reboot
```

</details>

<details>
  <summary>GNOME</summary>

```bash
# Les paquets listés dans le `grep` sont exclus, car jugés inutiles à mon sens
# Si un paquet vous intéresse dans cette liste d’exclusion, retirez-le simplement
yay -S $(yay -Sgq gnome | grep -Ev "epiphany|snapshot|gnome-calendar|gnome-contacts|gnome-maps|gnome-music|gnome-system-monitor|gnome-weather|gnome-clocks|gnome-tour|rygel|sushi") \
firefox resources usbutils gnu-free-fonts noto-fonts bluez bluez-utils mpv libvncserver flatpak \
transmission-gtk yt-dlp networkmanager-openvpn networkmanager-openconnect imagemagick gnome-tweaks

sudo systemctl enable bluetooth
sudo systemctl enable gdm
sudo reboot
```

</details>

### Imprimante et mDNS (si nécessaire)

```bash
yay -S cups ghostscript system-config-printer nss-mdns

# Désactiver le mDNS de `systemd-resolved` (géré par Avahi)
printf "[Resolve]\nMulticastDNS=no\n" | sudo tee /etc/systemd/resolved.conf.d/mdns.conf
sudo systemctl restart systemd-resolved

# Configurer `nsswitch` pour utiliser Avahi (résolution des noms en .local)
sudo sed -i '/^hosts/s/^.*/hosts : mymachines mdns_minimal [NOTFOUND=return] resolve [!UNAVAIL=return] files myhostname dns/' /etc/nsswitch.conf

sudo systemctl enable --now avahi-daemon
sudo systemctl enable --now cups.socket
```

### Polices

```bash
# Installation des polices Microsoft (Arial, Times New Roman, ...)
# Ajouter également une police Nerd pour le terminal afin d’afficher les icônes
yay -S ttf-firacode-nerd ttf-ms-fonts
```

*Configurez ensuite la police Nerd dans les paramètres du terminal.*

---

## 11. Snapshots Btrfs avec `Snapper`

`Snapper` gère automatiquement des snapshots avant et après chaque mise à jour `pacman` (via `snap-pac`) et de manière périodique, permettant un rollback rapide en cas de problème.

```bash
yay -S snapper snap-pac

# Reconfigurer le sous-volume `.snapshots` (Snapper recrée le sien)
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
sudo sed -e '/NUMBER_LIMIT/s/".*"/"30"/' \
		 -e '/TIMELINE_LIMIT_HOURLY/s/".*"/"0"/' \
		 -e '/TIMELINE_LIMIT_DAILY/s/".*"/"8"/' \
		 -e '/TIMELINE_LIMIT_WEEKLY/s/".*"/"5"/' \
		 -e '/TIMELINE_LIMIT_MONTHLY/s/".*"/"0"/' \
		 -e '/TIMELINE_LIMIT_QUARTERLY/s/".*"/"0"/' \
		 -e '/TIMELINE_LIMIT_YEARLY/s/".*"/"0"/' \
		 -i /etc/snapper/configs/root
```

> Vous pouvez adapter le nombre maximal de snapshots pour chaque période selon vos besoins.

### Empêcher `Snapper` de réinstaller sa tâche cron (évite le conflit avec le timer systemd)

```bash
sudo sed -ei '/^#NoExtract/s/^#//' -e '/^NoExtract/s@$@ etc/cron.hourly/snapper@' /etc/pacman.conf

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

## 12. Configurations finales

### Désactiver le compte root

*Une fois certain que votre utilisateur dispose bien des droits `sudo` :*

```bash
sudo passwd -dl root
```

---

## 13. Vérifications finales

```bash
# Vérifier qu’il n’y a pas d’erreurs critiques dans le noyau
sudo dmesg | grep -i failed

# Vérifier qu’aucun service n’a planté
systemctl --failed
```

---

## 14. Résumé de la stack

| Composant | Choix |
|-----------|-------|
| Chiffrement | LUKS2 |
| Système de fichiers | Btrfs + sous-volumes + compression zstd |
| Noyau | linux ou linux-lts |
| Bootloader | UKI (systemd-stub, sans bootloader) |
| Secure Boot | sbctl (clés personnelles) |
| Déverrouillage auto | TPM 2.0 (PCR 0+7) |
| Init | systemd |
| Réseau | NetworkManager |
| DNS | systemd-resolved (DNS over TLS) |
| Audio | PipeWire |
| Compositeur | Wayland |
| Bureau | KDE Plasma ou GNOME |
| Snapshots | Snapper + snap-pac |
| AUR helper | yay |
