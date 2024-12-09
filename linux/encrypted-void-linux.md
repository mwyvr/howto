# Encrypted Void Linux

This HOWTO summarizes the steps required to complete a `chroot` *almost*-full
disk encryption installation of Void Linux with encrypted root and swap volumes.

The defaults assume a modern SSD (nvme) and a no-dual-boot scenario.

**WARNING**: This guide wipes the target disk completely.

## End Result

- Unencrypted EFI partition for GRUB
- LUKS encrypted `root` and `swap` volumes
- No separate partition for `/home`
 
## Resources

In addition to this guide see also the [Void Linux
Handbook](https://docs.voidlinux.org/), specifically the [section on full disk
encryption](https://docs.voidlinux.org/installation/guides/fde.html).

Where this guide differs from the handbook:

- chronological layout with copy-buttons to make it easy to replicate when I need to.
- use of variables to make it easier to alter to differing target machines.
- target machine is UEFI equipped with NVME drives. You can adapt most of this guide to other targets.
- sufficient swap for ACPI S4 hibernation, based on a 16GB RAM device.

## Before You Begin

1. Read the **WARNING**: This guide wipes the target disk completely.
2. Prepare live [Void Linux installation
media](https://docs.voidlinux.org/installation/live-images/index.html) and boot
from the live ISO USB stick. 
3. Note that if using a BASE ISO with no GUI environment, you'll
want to `ssh` into the machine in order to be able to copy and paste from this
guide.
4. Establish a network connection from the target machine. `wpa_supplicant` is already enabled on the live ISOs, therefore to connect:

    wpa_cli
    add_network 0
    set_network 0 ssid "yourssid"
    set_network 0 psk "yourpassphrase"
    select_network 0

Exit `wpa_cli`; check for an IP address with `ip a`; recheck in a few moments
if needed. If not running a GUI on the live ISO, login to the target machine via
ssh at the ip with username `anon` and password `voidlinux`. Become `root` with `sudo -s`.

Let's begin!

## Define Variables

Edit and set the target disk for installation. 

    export DEVICE=/dev/nvme0n1

**WARNING**: This guide wipes the target disk completely.

Define the disk partitions:

    export EFI=${DEVICE}p1
    export CRYPT=${DEVICE}p2

Edit and set your non-root user:

    export YOURUSER=yourusername

Set the hostname for the installation target:

    export TARGETHOSTNAME=yournewhostname

If `ssh` into the machine, exporting a terminfo variable with a basic type will make the experience better:

    export TERM=xterm

## Disk Preparation

Attempt to deactivate existing volume groups, if any:

    vgchange -an

**WARNING**: `wipefs` wipes all filesystems from the target disk.

    wipefs -a $DEVICE

Create a new GPT partition table with a reasonably-sized EFI partition and a
"crypt" partition taking up the rest of the disk.

    sfdisk --force -w always -W always $DEVICE <<EOF
    label: gpt
    name=esp, size=500M, type="EFI System"
    name=crypt
    EOF

Creating EFI file system:

    mkfs.vfat -n EFI $EFI

Create LUKS1 / dm-crypt encrypted $CRYPT partition/volume for root and swap:

    # pass phrease required
    cryptsetup luksFormat --type luks1 $CRYPT

Open the encrypted volume:

    # pass phrase required
    cryptsetup luksOpen $CRYPT crypt

Create `volg` as a volume group, and swap and root logical volumes:

    vgcreate volg /dev/mapper/crypt
    lvcreate --name swap -L 25G volg # 1.5x RAM if using hibernation
    lvcreate --name root -l 100%FREE volg

Create filesystems:

    mkfs.xfs -L root /dev/volg/root
    mkswap /dev/volg/swap

Mount filesystems:

    mount /dev/volg/root /mnt
    mkdir -p /mnt/boot/efi
    mount $EFI /mnt/boot/efi

Prepare for `chroot`:

    # copy Void Linux signing keys to target system
    mkdir -p /mnt/var/db/xbps/keys
    cp /var/db/xbps/keys/* /mnt/var/db/xbps/keys/
    # Using the network, perform base installation:
    xbps-install -Sy -R https://repo-fastly.voidlinux.org/current -r /mnt base-system cryptsetup grub-x86_64-efi lvm2

Complete basic configuration:

    # enter the chroot
    xchroot /mnt

Configure the target:

    chown root:root /
    chmod 755 /

Setup root:

    # password required
    passwd root

Setup your non-root user:

    useradd $YOURUSER
    usermod -aG wheel,video $YOURUSER # for sudo/doas; video for seatd/Wayland
    # password required
    passwd $YOURUSER

Hostname and timezone:

    echo $TARGETHOSTNAME > /etc/hostname
    ln -svf /usr/share/zoneinfo/$TIMEZONE /etc/localtime

Locale for **glibc** systems only (not musl):

    echo "LANG=en_US.UTF-8" > /etc/locale.conf
    echo "en_US.UTF-8 UTF-8" >> /etc/default/libc-locales
    xbps-reconfigure -f glibc-locales

Populate `/etc/fstab`; do it manually if your setup is more complex.

    cat << EOF > /etc/fstab
    # <file system>	<dir>			<type>	<options>							<dump>	<pass>
    tmpfs						/tmp			tmpfs		defaults,nosuid,nodev		0				0
    /dev/volg/root	/					xfs			defaults								0				0
    /dev/volg/swap	swap			swap		defaults								0				0
    /dev/nvme0n1p1	/boot/efi	vfat		defaults								0				0
    EOF

Configure grub; first:

    echo "GRUB_ENABLE_CRYPTODISK=y" >> /etc/default/grub


# FOLLOWING TB COMPLETED 
And in grub the value reported by:

blkid -o value -s UUID $CRYPT
i.e.
4a71f4bf-3fd3-44e0-8a05-cc95f34568bf

GRUB_CMDLINE_LINUX_DEFAULT="loglevel=4 rd.lvm.vg=volg rd.luks.uuid=4a71f4bf-3fd3-44e0-8a05-cc95f34568bf"

dd bs=1 count=64 if=/dev/urandom of=/boot/volume.key
cryptsetup luksAddKey $CRYPT /boot/volume.key


     chmod 000 /boot/volume.key
    chmod -R g-rwx,o-rwx /boot

echo "volg    $CRYPT     /boot/volume.key     luks" | tee -a /etc/crypttab

cat <<EOF >/etc/dracut.conf.d/10-crypt.conf
install_items+=" /boot/volume.key /etc/crypttab "
EOF

grub-install $DEVICE
xbps-reconfigure -fa

All done.

    exit
    umount -R /mnt



echo "GRUB_ENABLE_CRYPTODISK=y" >> /etc/default/grub

Determine the UUID of the encrypted partition:

    blkid -o value -s UUID $CRYPT
    # 11ad858f-7749-4a2b-8d7c-7300906c778a # something like this

Edit /etc/default/grub and add, per this example, the following to GRUB_CMDLINE_LINUX_DEFAULT:

  rd.lvm.vg=volg rd.luks.uuid=11ad858f-7749-4a2b-8d7c-7300906c778a

## LUKS Key Setup

    dd bs=1 count=64 if=/dev/urandom of=/boot/volume.key
    cryptsetup luksAddKey $CRYPT /boot/volume.key
    chmod 000 /boot/volume.key
    chmod -R g-rwx,o-rwx /boot

    # add the volume.key to crypttab:
    echo "volg  $CRYPT  /boot/volume.key  luks" >> /etc/crypttab

    # include the key in the initramfs
    mkdir -p /etc/dracut.conf.d
    echo 'install_items+=" /boot/volume.key /etc/crypttab "' >> /etc/dracut.conf.d/10-crypt.conf

## Install Grub

    grub-install $DEVICE
    ln -svf /usr/share/zoneinfo/$LOCALTIME /etc/localtime

## Exit chroot


    exit
    umount -R /media/root


## Post installation configuration

xbps-install xtools NetworkManager
ln -sv etc

xbps-install -Su void-repo-nonfree
xbps-install -S
xbps-install intel-ucode

xbps-reconfigure --force linux6.6 #adjust as needed


# As user

    xi turnstile
    mkdir -p ~/.config/service/dbus
    ln -s /usr/share/examples/turnstile/dbus.run ~/.config/service/dbus/run
https://docs.voidlinux.org/config/services/user-services.html


## as root

socklog-void chrony seatd

usermod -aG _seatd mw
