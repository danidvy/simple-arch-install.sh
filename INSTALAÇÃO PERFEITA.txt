#!/bin/bash
# encoding: utf-8

##################################################
#                   Variaveis                    #
##################################################
# Nome do Computador
HOSTN=PanBox-OS

# Localização. Verifique o diretório /usr/share/zoneinfo/<Zone>/<SubZone>
LOCALE=America/Sao_Paulo

# Senha de Root do sistema após a instalação
ROOT_PASSWD=14

# File System das partições
ROOT_FS=ext4

######## Variáveis menos suscetíveis a mudanças
KEYBOARD_LAYOUT=br-abnt2
LANGUAGE=pt_BR

##################################################
#                   functions                    #
##################################################

function cria_fs
{
        ERR=0
        # Formata partições root para o File System
        echo "Formatando partição root"
        mkfs.$ROOT_FS /dev/sda3 -L Root 1>/dev/null || ERR=1
        
        if [[ $ERR -eq 1 ]]; then
                echo "Erro ao criar File Systems"
                exit 1
        fi
}

function monta_particoes
{
        ERR=0
        echo "Montando partições"
        # Monta partição root
        mount /dev/sda3 /mnt || ERR=1
        
        if [[ $ERR -eq 1 ]]; then
                echo "Erro ao criar File Systems"
                exit 1
        fi
}

function configurando_pacman
{
        echo "Configurando pacman"
        cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.bkp
        sed "s/^Ser/#Ser/" /etc/pacman.d/mirrorlist > /tmp/mirrors
        sed '/Brazil/{n;s/^#//}' /tmp/mirrors > /etc/pacman.d/mirrorlist

        if [ "$(uname -m)" = "x86_64" ]
        then
                cp /etc/pacman.conf /etc/pacman.conf.bkp
                # Adiciona o Multilib
                sed '/^#\[multilib\]/{s/^#//;n;s/^#//;n;s/^#//}' /etc/pacman.conf > /tmp/pacman
                mv /tmp/pacman /etc/pacman.conf

        fi
}

function instalando_sistema
{
        ERR=0
        echo "Rodando pactrap system install"
        pacstrap /mnt base base-devel inux linux-firmware exfat-utils acpi dosfstools intel-ucode ntfs-3g nfs-utils mtools usbutils autofs ntp poppler python2-xdg ttf-dejavu xdg-user-dirs xdg-utils gvfs gvfs-smb gvsf-afc gvsf-mtp mtpfs android-file-transfer grub os-prober bash-completion gpm udevil fontconfig gsfonts git usb_modeswitch gptfdisk xf86-video-intel xf86-input-keyboard xf86-input-mouse xf86-input-libinput xorg-server xorg-xinit alsa-plusins alsa-utils alsa-tools pulseaudio pulseaudio-alsa pulseaudio-equalizer pulseaudio-zeroconf pulseaudio-qt pavucontrol volumeicon
networkmanager network-manager-applet wireless_tools net-tools dialog nmap wget nano mousepad neofetch htop conky gnome-icon-theme gnome-disk-utility 
gparted Baobab yakuake xterm gnome-terminal audacious audacious-plugins vcl mplayer xine-lib ffmpeg transmission-gtk transmission-cli wine winetricks playonlinux xcursor-simpleandsoft xcursor-themes xcursor-vanilla-dmz-aa xcursor-bluecurve feh youtube-dl zathura mupdf gimp synapse bc gnome-calculator java-runtime java-openjfx
dotnet-host dotnet-runtime ufw BleachBit opera opera-ffmpeg-codecs p7zip go gecko xfce4 xfce4-goodies lightdm lightdm-gtk-greeter || ERR=1
        echo "Rodando genfstab"
        genfstab -p /mnt >> /mnt/etc/fstab || ERR=1

        if [[ $ERR -eq 1 ]]; then
                echo "Erro ao instalar sistema"
                exit 1
        fi
}

##################################################
#                   Script                       #
##################################################
# Carrega layout do teclado ABNT2
loadkeys $KEYBOARD_LAYOUT

#### Instalação
configurando_pacman
instalando_sistema

#### Entra no novo sistema (chroot)
arch-chroot /mnt << EOF
# Configura hostname
echo $HOSTN > /etc/hostname
cp /etc/hosts /etc/hosts.bkp
sed 's/localhost$/localhost '$HOSTN'/' /etc/hosts > /tmp/hosts
mv /tmp/hosts /etc/hosts

# Configura layout do teclado
echo 'KEYMAP='$KEYBOARD_LAYOUT > /etc/vconsole.conf
echo 'FONT=lat0-16' >> /etc/vconsole.conf
echo 'FONT_MAP=' >> /etc/vconsole.conf

# Configura locale.gen
cp /etc/locale.gen /etc/locale.gen.bkp
sed 's/^#'$LANGUAGE'/'$LANGUAGE/ /etc/locale.gen > /tmp/locale
mv /tmp/locale /etc/locale.gen
locale-gen

# Configura locale.conf
export LANG=$LANGUAGE'.utf-8'
echo 'LANG='$LANGUAGE'.utf-8' > /etc/locale.conf
echo 'LC_COLLATE=C' >> /etc/locale.conf
echo 'LC_TIME='$LANGUAGE'.utf-8' >> /etc/locale.conf

# Configura hora
ln -s /usr/share/zoneinfo/$LOCALE /etc/localtime
echo $LOCALE > /etc/timezone
hwclock --systohc --utc

# Configura ambiente ramdisk inicial
mkinitcpio -p linux

# Instala e gera configuração do GRUB Legacy
grub-install /dev/sda
grub-mkconfig -o /boot/grub/grub.cfg

# Altera a senha do usuário root
echo -e $ROOT_PASSWD"\n"$ROOT_PASSWD | passwd
EOF

echo "Umounting partition"
umount -R /mnt
reboot