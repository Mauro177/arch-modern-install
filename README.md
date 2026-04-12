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
5. [Configuration de base du système](#5-configuration-de-base-du-système)
6. [Unified Kernel Image (UKI)](#6-unified-kernel-image-uki)
7. [Post-installation](#7-post-installation)
8. [Utilisateur et AUR Helper](#8-utilisateur-et-aur-helper)
9. [Secure Boot + TPM 2.0](#9-secure-boot--tpm-20)
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
- **Secure Boot** - UKI signée (via `shim`)
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

*Repérez votre disque cible (ex: `nvme0n1`, `sda`) :*
```bash
lsblk | grep disk
```

### Partitionner le disque

*Stocker dans une variable le disque cible de l’installation (ex. /dev/nvme0n1, /dev/sda) afin de faciliter la suite du guide pour permettre le copier-coller des commandes :*

```bash
export disk=/dev/monDisque
```

*Effacement complet du disque si vous ne faites pas de Dual Boot :*

```bash
wipefs -a $disk
```

*Créez un schéma GPT avec deux partitions :*

| Partition | Taille | Type |
|-----------|--------|------|
| `1` | 512M | EFI System |
| `2` | Reste | Linux filesystem |

```bash
cfdisk $disk
```
> ⚠️ **Attention** si une partition EFI existe déjà et que vous voulez du Dual Boot, ne la recréer pas.

### Formater la partition EFI

*⚠️ UNIQUEMENT si la partition EFI est nouvelle :*

```bash
mkfs.fat -F32 ${disk}1
```

### Chiffrement LUKS2 de la partition racine

*Stocker dans une variable la partition cible d'Arch Linux. Si vous ne faites pas de Dual Boot, garder le '2'. Sinon, vérifier le numéro avec la commande 'lsblk $disk' :*

```bash
export part=${disk}2
```

*Initialiser la partition chiffrée (vous devrez choisir une passphrase temporairement le temps de mettre en place le TPM 2.0) :*

```bash
cryptsetup luksFormat $part
```

*Déchiffrer la partition (la rendre accessible sous /dev/mapper/root) :*


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

> LUKS2 chiffre l’intégralité de la partition.
---

## 3. Configuration Btrfs et sous-volumes

Btrfs permet de créer des **sous-volumes** indépendants sur une même partition, ce qui est pratique pour des snapshots ciblés et gérer finement des options de montage.

### Créer le système de fichiers et les sous-volumes

```bash
# Formater la partition en Btrfs et la monter dans /mnt
mkfs.btrfs -L root /dev/mapper/cryptroot
mount /dev/mapper/cryptroot /mnt

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

*Sous-volumes Btrfs :*

```bash
mount -o compress=zstd,subvol=@ /dev/mapper/cryptroot /mnt
mount --mkdir -o compress=zstd,subvol=@home      /dev/mapper/cryptroot /mnt/home
mount --mkdir -o compress=zstd,subvol=@log       /dev/mapper/cryptroot /mnt/var/log
mount --mkdir -o noatime,compress=zstd,subvol=@cache /dev/mapper/cryptroot /mnt/var/cache
mount --mkdir -o noatime,compress=zstd,subvol=@tmp   /dev/mapper/cryptroot /mnt/var/tmp
mount --mkdir -o noatime,subvol=@vms             /dev/mapper/cryptroot /mnt/var/lib/libvirt/images
mount --mkdir -o compress=zstd,subvol=@snapshots /dev/mapper/cryptroot /mnt/.snapshots
```

*Partition EFI :*

```bash
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
pacstrap -K /mnt base linux linux-firmware intel-ucode btrfs-progs efibootmgr networkmanager dracut
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

### Fuseau horaire et configuration de l’horloge

*Adapter la localisation en conséquence :*

```bash
ln -sf /usr/share/zoneinfo/Europe/Brussels /etc/localtime
hwclock --systohc
```

### Langue et disposition du clavier

*Configuration de la langue ; ex. : fr_FR pour le français (France) ou en_US pour l’anglais (États-Unis) :*

```bash
sed -i '/fr_BE.UTF-8/s/^#//' /etc/locale.gen
locale-gen
echo "LANG=fr_BE.UTF-8" > /etc/locale.conf
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
echo "127.0.1.1		   $hostname" >> /etc/hosts

# Désactiver systemd-networkd (géré par NetworkManager)
systemctl mask systemd-networkd
systemctl enable NetworkManager
```

---

## 6. Unified Kernel Image (UKI)

Une **Unified Kernel Image (UKI)** regroupe le noyau, l’initramfs et les paramètres du noyau (ligne de commande) dans un seul fichier bootable `.efi`. Cela simplifie le Secure Boot (un seul fichier à signer) et supprime le besoin d’un bootloader séparé, comme GRUB. Cette méthode est également bien plus sécurisée que l’approche traditionnelle, où un bootloader signé lance séparément le noyau signé, l’initramfs (non signé) et la ligne de commande du noyau (modifiable via le bootloader).

### Configurer `dracut` pour UKI

*Générer des UKIs à la place des images Initramfs :*

```bash
echo "uefi=\"yes\"" > /etc/dracut.conf.d/uki.conf
echo "enhanced_cpio=\"yes\"" >> /etc/dracut.conf.d/uki.conf
```

> La seconde option 'enhanced_cpio' est une optimisation pour Btrfs.

*Ligne de commande du noyau :*

```bash
echo "kernel_cmdline=\"rd.luks.name=$(blkid -s UUID -o value ${part})=cryptroot root=/dev/mapper/cryptroot \
rootflags=subvol=@ ro quiet\"" >> /etc/dracut.conf.d/uki.conf
```

### Générer l’UKI

```bash
mkdir -p /boot/efi/EFI/ARCH

# Nettoyer les anciens initramfs
rm -f /boot/initramfs-*

# Générer le fichier .efi [UKI]
dracut -f /boot/efi/EFI/ARCH/shimx64.efi
```

> Je prends de l’avance en nommant l’UKI « shimx64.efi » : cela réduira le nombre de configurations nécessaires lors de la mise en place du Secure Boot.

### Enregistrer l’entrée de démarrage EFI dans le micrologiciel du PC

```bash
efibootmgr --create --disk $disk --part 1 --label "Arch Linux" \
  --loader '\EFI\ARCH\shimx64.efi' --unicode
```

### Définir le mot de passe root

```bash
passwd
```

### Quitter le live CD

```bash
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

systemctl start sshd
```

### Paramètres de base du système

*Clavier (us pour QWERTY) :*
```bash
localectl set-keymap fr
```

*Localisation (à adapter) et horloge automatique (NTP) :*

```bash
timedatectl set-timezone Europe/Brussels
timedatectl set-ntp true
hwclock --systohc
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
pacman -S gcc python which automake autoconf bison debugedit fakeroot texinfo libtool make patch \
git wget sudo-rs pacman-contrib reflector wireless-regdb firewalld firewall-config htop btop \
zram-generator udisks2 fwupd virt-what tuned tuned-ppd bash-completion pkgfile 7zip unrar zip unzip \
bzip3 ntfs-3g dosfstools udftools perl-locale-gettext fzf eza bat nano vim rsync inetutils tmux man-db \
tldr plymouth logrotate
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
# Rotation et nettoyage régulié des journaux non systemd-journald
systemctl enable --now logrotate.timer
# Profils d'alimentation dynamiques
systemctl enable --now tuned
systemctl enable --now tuned-ppd
# Retrouve les paquets des commandes introuvables
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

> ZRAM crée un espace de swap compressé en RAM — beaucoup plus rapide qu’un swap sur disque et réduit l’usure du SSD.

### DNS over TLS avec `systemd-resolved`

*Configurations :*

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

*Verification :*
```bash
resolvectl
```

### Pare-feu `firewalld`

```bash
systemctl enable --now firewalld
```

### Mises à jour firmware `fwupd`

```bash
systemctl enable --now fwupd-refresh.timer
fwupdmgr refresh
```

### sudo-rs

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
```

*Vérification :*

```bash
sudo --version
```

> `sudo-rs` est une réimplémentation de `sudo`, écrite en Rust et plus sécurisée.

### Hook pour regénération automatique de l'UKI `dracut`

Lorsqu’une installation ou une mise à jour de paquets nécessite de régénérer l’UKI, le hook automatique présent par défaut est actuellement buggé et ne tient pas compte de l’utilisation des UKI. Il faut donc reconfigurer correctement ce hook pour pallier ce problème. Lorsque ce bug sera corrigé, je supprimerais cette section du guide.

*Création des répertoires nécessaires :*

```bash
mkdir -p /etc/pacman.d/{hooks,scripts}
```

*Récupération du hook par défaut :*

```bash
cp /usr/share/libalpm/hooks/*dracut-install* /etc/pacman.d/hooks/
```

*Corrrections :*

```bash
sed '/^Exec/s@= .*@= /etc/pacman.d/scripts/dracut-install@' -i /etc/pacman.d/hooks/90-dracut-install.hook
```

```bash
nano /etc/pacman.d/scripts/dracut-install
```

```ini
#!/usr/bin/env bash

# If KERNEL_INSTALL_INITRD_GENERATOR is set, disable this hook
if [ -n "$KERNEL_INSTALL_INITRD_GENERATOR" ]; then
    exit 0
fi

args=('--force' '-L 3')

while read -r line; do
	if [[ "$line" == 'usr/lib/modules/'+([^/])'/pkgbase' ]]; then
		read -r pkgbase < "/${line}"
		kver="${line#'usr/lib/modules/'}"
		kver="${kver%'/pkgbase'}"

		echo "--> Building UKI for ${pkgbase} (${kver})"
		install -Dm0644 "/${line%'/pkgbase'}/vmlinuz" "/boot/vmlinuz-${pkgbase}"
		dracut "${args[@]}" "/boot/efi/EFI/ARCH/grubx64.efi" --kver "$kver"
		break
	else	
		echo "--> Updating UKI"
		dracut "${args[@]}" "/boot/efi/EFI/ARCH/grubx64.efi"
		break
	fi
done
```

*Rendre le script exécutable :*

```bash
chmod +x /etc/pacman.d/scripts/dracut-install
```

---

## 8. Utilisateur et AUR Helper

### Créer un utilisateur

```bash
useradd -m -G wheel {user}
passwd {user}
```

### Supprimer root via SSH

```bash
sed -E -e '/^#PermitRootLogin/s/^#//' -e '/^PermitRootLogin/s/(yes|prohibit-password)/no/' \
	-i /etc/ssh/sshd_config
rm -f /etc/ssh/sshd_config.d/permitrootlogin.conf

systemctl reload sshd
```

*Connectez-vous désormais avec votre utilisateur et utilisez `sudo`.*

### AUR Helper (yay)

```bash
git clone https://aur.archlinux.org/yay-bin.git
cd yay-bin
makepkg -sirc
cd ..
rm -rf yay-bin
```

> Permet d’installer, de supprimer et de mettre à jour tous les paquets du dépôt d’Arch (en utilisant `pacman`), mais inclut également tous les paquets disponibles dans l’[AUR](https://aur.archlinux.org/).

---

## 9. Secure Boot + TPM 2.0

Le Secure Boot garantit que seule une UKI signée avec vos propres clés peut démarrer. Le TPM 2.0 permet de déverrouiller automatiquement le disque chiffré si la chaîne de démarrage est intacte (via les PCRs).

### Installation de SHIM et outils nécessaire

```bash
yay -S shim-signed mokutil sbsigntools
```

*Placer shim au démarrage pour le Secure Boot :*

```bash
sudo cp /usr/share/shim-signed/shimx64.efi /boot/efi/EFI/ARCH/
sudo cp /usr/share/shim-signed/mmx64.efi /boot/efi/EFI/ARCH/
```

> shimx64.efi est signé avec une clé Microsoft ; il contient également une clé Ubuntu codée en dur. MokManager (mmx64.efi) est signé avec la clé d’Ubuntu. shimx64.efi peut lancer des binaires EFI signés (grubx64.efi par défaut) soit avec des clés Microsoft, soit avec la clé Ubuntu intégrée, ou encore avec nos propres clés que nous aurons importées (via MOK) et qui serviront à signer nos UKI. Il s’agit de la méthode la plus classique, utilisée par des distributions comme Debian, Ubuntu ou Fedora pour activer le Secure Boot, et elle permet de ne pas avoir à modifier la base de clés du firmware.

*Création de notre clé et certification pour signer les UKIs :*

```bash
sudo mkdir /usr/share/sbsign

sudo openssl req -newkey rsa:4096 -nodes -keyout /usr/share/sbsign/MOK.key -new -x509 -sha256 -days 3650 -subj "/CN=my Machine Owner Key/" -out /usr/share/sbsign/MOK.crt

sudo openssl x509 -outform DER -in /usr/share/sbsign/MOK.crt -out MOK.cer
```

*Importation du certificat créé dans MOK :*

```bash
sudo mokutil --import MOK.cer

sudo rm -f MOK.cer
```

> Le mot de passe temporaire demandé vous sera redemandé lorsque vous allez enrôler le certificat au redémarrage.

*Configuration de dracut pour signer automatiquement les UKIs générées :*

```bash
echo "uefi_secureboot_cert=\"/usr/share/sbsign/MOK.crt\"" | sudo tee -a /etc/dracut.conf.d/uki.conf
echo "uefi_secureboot_key=\"/usr/share/sbsign/MOK.key\"" | sudo tee -a /etc/dracut.conf.d/uki.conf
```

*Regénerer l'UKI qui sera signé automatiquement :*

```bash
sudo dracut -f /boot/efi/EFI/ARCH/grubx64.efi
```

> L'UKI est nommée en "grubx64.efi", car c'est le fichier que SHIM lance par défaut.

*Redémarrer dans le BIOS et activé Secure Boot :*

```bash
sudo systemctl reboot --firmware-setup
```

MokManager vous accueillera avec un grand écran bleu. Faites simplement : Enter, Enroll key, entrez le mot de passe temporaire que vous aurez spécifié plus tôt, puis Reboot.

### Enrôler le TPM 2.0

```bash
# Lier le déverrouillage LUKS2 au TPM 2.0 (PCR 0 = firmware, PCR 7 = Secure Boot)
systemd-cryptenroll --tpm2-device=auto --tpm2-pcrs=0+7 /dev/PartitionChiffrée

reboot
```

> Après cette étape, le disque se déverrouille automatiquement au démarrage tant que la chaîne de démarrage est valide (Secure Boot actif avec sa base de clés inchangée et firmware du PC non altéré). En cas de modification du firmware ou du Secure Boot, la passphrase sera demandée. **IMPORTANT** : les mises à jour du BIOS (UEFI) modifient le firmware, donc la passphrase sera à nouveau demandée, et il faudra relier le déverrouillage LUKS2 au TPM 2.0 avec la commande ci-dessus.

---

## 10. Environnement de bureau

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
| Secure Boot | shim |
| Déverrouillage auto | TPM 2.0 (PCR 0+7) |
| Init | systemd |
| Réseau | NetworkManager |
| DNS | systemd-resolved (DNS over TLS) |
| Audio | PipeWire |
| Compositeur | Wayland |
| Bureau | KDE Plasma ou GNOME |
| Snapshots | Snapper + snap-pac |
| AUR helper | yay |
