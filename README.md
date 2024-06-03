# arch-installation-tutorial
Una piccola guida non ufficiale all'installazione.

## Layout tastiera

_Utility:_ `localectl list-keymaps`
```sh
loadkeys it
```

## Font console

_Utility:_ `ls /usr/share/kbd/consolefonts/`
```sh
setfont ter-120b
```

## Modalità d'avvio

```sh
cat /sys/firmware/efi/fw_platform_size`
```
Se questo comando restituisce `64` allora il sistema è avviato in modalità UEFI 64-bit x64, se restituisce `32` è stato avviato in modalità UEFI 32-bit IA32, altrimenti in modalità BIOS (CSM).
Questo è buono saperlo per i passaggi finali dell'istallazione.
## Connessione a internet

### Ethernet
Collega il cavo.

### WiFi
_Utility:_ `iwctl device list`
_Utility:_ `iwctl station $dev scan`
_Utility:_ `iwctl station $dev get-networks`
```sh
iwctl --passphrase $pwd station $dev connect $ssid
```
`$pwd` rappresenta la password, `$dev` il nome della scheda di rete e `$ssid` l'ssid dell'access point.

## Partizionamento
_Utility:_ `lsblk`
```sh
cfdisk /dev/sdX
```
`X` rappresenta la lettera assegnata al dispositivo di archiviazione (es. sda, sdb, ...).
Questo è un esempio di partizionamento:

| Dimensioni | Nome | Tipo             |
| ---------- | ---- | ---------------- |
| 512M       | boot | `$boot_method`   |
| 16G        | swap | Linux swap       |
| 128G       | root | Linux filesystem |
| ...        | home | Linux filesystem |
`$boot_method` è EFI partition oppure Linux filesystem in base al tipo di avvio desiderato
>Write e Exit.

## Formattazione

```sh
mkfs.fat -F32 /dev/sdX1
mkswap /dev/sdX2
mkfs.ext4 /dev/sdX3
mkfs.ext4 /dev/sdX4
```

## Montaggio

_Utility:_ `lsblk`
**EFI**
```sh
mount --mkdir /dev/sdX1 /mnt/efi
swapon /dev/sdX2
mount /dev/sdX3 /mnt
mount --mkdir /dev/sdX4 /mnt/home
```

**BIOS**
```sh
mount --mkdir /dev/sdX1 /mnt/boot
swapon /dev/sdX2
mount /dev/sdX3 /mnt
mount --mkdir /dev/sdX4 /mnt/home
```

## Classifica mirror con reflector
_Utility:_ `cat /etc/pacman.d/mirrorlist`
### Backup lista dei mirror
```sh
cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.bck
```

### Generazione classifica
```sh
reflector --save /etc/pacman.d/mirrorlist --latest 10 --protocol https --sort rate
```

## Installazione pacchetti essenziali e strumenti

```sh
pacstrap -K /mnt base base-devel linux linux-firmware sudo nano
```
L'editor `nano` è consigliato per chi non ha familiarità con i comandi di Vim
## Configurazione

### Fstab
[fstab - ArchWiki (archlinux.org)](https://wiki.archlinux.org/title/Fstab)
```sh
genfstab -U /mnt >> /mnt/etc/fstab
```

### Cambio root
```sh
arch-chroot /mnt
```

### Lingua
```sh
nano /etc/locale.gen
```

> Cancellare il '#' vicino alla lingua preferita.
> Ctrl+S e Ctrl+X.

_Utility:_ `localectl list-keymaps`
```sh
locale-gen
echo LANG=$lang > /etc/locale.conf
export LANG=$lang
echo KEYMAP=$keymap >> /etc/vconsole.conf
```
$lang rappresenta la stringa corrispondente alla lingua scelta nel file '/etc/locale.gen' e $keymap il layout della tastiera.

### Fuso orario
_Utility:_ `ls /usr/share/zoneinfo`
_Utility:_ `ls /usr/share/zoneinfo/$region`
```sh
ln -sf /usr/share/zoneinfo/$region/$city /etc/localtime
hwclock --systohc
```
`$region` è il continente di appartenenza e `$city` la città.

### Pacman
Abilita Multilib
```sh
nano /etc/pacman.conf
```
> Cancellare il '#' vicino alle righe riguardanti Multilib.
> Opzionale: Impostare a 5 il numero di download paralleli.
> Ctrl+S e Ctrl+X.
## Configurazione di rete

### Impostazione hostname
```sh
echo $hostname > /etc/hostname
nano /etc/hosts
```
/etc/hosts
```
127.0.0.1 localhost
::1 localhost
127.0.1.1 $hostname.localdomain localhost
```
> Ctrl+S e Ctrl+X.

`$hostname` è il nome da dare alla macchina.
## Installazione driver

```sh
pacman -Syu
pacman -S networkmanager network-manager-applet \
bluez bluez-utils \
pulseaudio alsa-utils alsa-plugins alsa-ucm-conf sof-firmware
```

## Installazione strumenti

### Strumenti comuni
```sh
pacman -S curl wget git openssh
```

### Shell
- bash - `pacman -S bash bash-completion`
- [Zsh](https://www.zsh.org/) - `pacman -S zsh zsh-completions`
- [fish](https://fishshell.com/) - `pacman -S fish`
- Sono disponbili altre alternative ma queste sono le più comuni

### Altri strumenti
```sh
pacman -S bottom neofetch
```

## Abilitazione servizi

```sh
systemctl enable NetworkManager
systemctl enable bluetooth
systemctl enable sshd
```

## Creazione profilo utente e configurazione permessi

### Impostazione password root
```sh
passwd
```

### Creazione profilo utente
[Users and groups - ArchWiki (archlinux.org)](https://wiki.archlinux.org/title/users_and_groups#Group_list)
```sh
useradd -m -g users -G wheel,games,audio,storage -s /bin/$shell $username
passwd $username
```
`-m` Crea la directory `/home/$username`.
`-g` Assegna il gruppo di login principale.
`-G` Lista di gruppi da aggiungere all'utente.
`-s` Specifica una shell.

`$username` è il nome dell'utente da creare e `$shell` è il percorso alla shell.

### Configurazione permessi gruppo wheel
```sh
EDITOR=nano visudo
```
> Cancellare il '#' vicino alla configurazione preferita.
> Ctrl+S e Ctrl+X.

## Bootloader

```sh
pacman -S grub $microcode efibootmgr
mkinitcpio -p linux
```
Il pacchetto `efibootmgr` non è necessario se si scarica in modalità BIOS.
`$microcode` è `amd-ucode` per processori AMD o `intel-ucode`per processori Intel.

**64-bit UEFI**
`grub-install --target=x86_64-efi --efi-directory=/efi --bootloader-id=GRUB /dev/sdX`
**32-bit UEFI**
`grub-install --target=i386-efi --efi-directory=/efi --bootloader-id=GRUB /dev/sdX`
**BIOS**
`grub-install /dev/sdX`

`X` rappresenta la lettera assegnata al dispositivo di archiviazione (es. sda, sdb, ...).

### Applicazione della configurazione di GRUB
```sh
grub-mkconfig -o /boot/grub/grub.cfg
```

### Fatto!
```sh
exit
umount -R /mnt
poweroff
```
**Staccare il supporto d'installazione**
