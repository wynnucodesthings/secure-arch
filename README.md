This guide will be mostly be done within the arch linux installation guide, please have that open.
## Prerequisites: your computer has to have tpm support and yeah.

Before booting into arch, go to secure boot option and get setup mode to be turned on. in my case I had to clear my secure boot keys.


### Connect to the internet:
If you're using an ethernet port, just do `ping whateveryouwant.whateveryouwant`
and you should get a proper response and not get timed out.

Alternatively, if you want to use wifi; run `iwctl` and follow the arch install guide on how to do that.

### Partitioning:

Use the arch install guide to partition however you're comfortable.
Your partitioning should look something like this:

| partition  | type | Size |
| :----------------: | :------: | :----: |
| nvme0n1p1 | EFI system partition | 1G |
| nvme0n1p2 | Linux root (x86-64) | Rest of your disk |

### Encryption:
we're going to use Luks encryption to encrypt our root partition:

To do that you have to do: `ROOTDEV=/path/to/your/root/partition` and then follow that up by:
```
cryptsetup luksFormat --type luks2 $ROOTDEV
cryptsetup luksOpen $ROOTDEV root
ROOTDEV=/dev/mapper/root
```

### Formatting and stuff
It's mostly similar to the arch wifi
```
mkfs -t ext4 $ROOTDEV
mount --mkdir $ROOTDEV /mnt
mkfs.fat -F32 /path/to/your/efi/partition
mount --mkdir /path/to/your/efi/partition /mnt/efi
```
and you should be good with your partitions.

### os installation
```
# This just setup your mirrors and stuff I think the default is fine too.
reflector --save /etc/pacman.d/mirrorlist --protocol https --latest 5 --sort age
pacstrap -K /mnt base linux linux-firmware linux-headers less sudo git base-devel networkmanager vim man-db man-pages openssh htop
```
after that's done go into your system and the rest is also similar to the arch wiki but I'll go over them here once.
```
arch-chroot /mnt

ln -sf /usr/share/zoneinfo/Region/City /etc/localtime

MYHOSTNAME=fancyhostnameoowee
MYUSERNAME=jp
ROOTPASSWORD=securerootpassword
USERPASSWORD=secureuserpassword

hwclock --systohc

echo en_US.UTF-8 UTF-8 >> /etc/locale.gen
locale-gen

echo $MYHOSTNAME > /etc/hostname

chpasswd <<< root:$ROOTPASSWORD

useradd -m $MYUSERNAME
chpasswd <<< $MYUSERNAME:$USERPASSWORD
echo "$MYUSERNAME ALL=(ALL) ALL" > /etc/sudoers.d/$MYUSERNAME

systemctl enable NetworkManager.service
systemctl enable sshd.service
```

### bootloader
We are going to use systemd-boot because grub sucks.
