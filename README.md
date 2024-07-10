Welcome to yet another arch install! This arch install will be mostly based off of the [arch installation guide](https://wiki.archlinux.org/title/installation_guide), however with the addition of things such as SecureBoot, luksEncrpytion and TPM 2.0 and additional tweaks that I will do along the way, that therefore make the system extremely secure and optimized.

### Prerequisites
---
- Your device should support SecureBoot(Obviously) and you should be able to put your SecureBoot status to setup mode.
- Your device should have a TPM 2.0 device.
- Your cpu should be a x64 architecture.
- USB with arch linux flashed into it(I will not go through this).
- And finally an internet connection.

### Connect to the internet
---
If you already have an ethernet connection to your computer, just ping a server to see if you have a connection going:

```shell
ping example.com
```

If you get a response and are able to ping a server, then congrats! you are connected to the internet.

If you do not have a ethernet connection, the process will be a bit different. If you plan on using WIFI you should instead use [iwctl](https://wiki.archlinux.org/title/Iwd#iwctl):

```shell
iwctl
# To list your devices
device list
# If the device or its corresponding adapter is turned off, turn it on:
device device_name set-property Powered on
adapter adapter_name set-property Powered on
# Then, to initiate a scan for networks (note that this command will not output anything):
station name scan
# You can then list all available networks:
station name get-networks
# Finally, to connect to a network:
station name connect SSID
```
(You can use TAB to auto complete your device and adapter names.)
### Partitioning disks
---
We want two partitions:
- An efi partition that's usually about 1G in size.
- And a root partition that will take the rest of the disk.
Start creating your partitions by:
```shell
# check your disks and find which one you want to partition:
lsblk
```
In my cafe the output of that command looks like this:
```shell
NAME        MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINTS  
sda           8:0    0 931.5G  0 disk     
zram0       253:0    0   7.8G  0 disk  [SWAP]  
nvme0n1     259:0    0 119.2G  0 disk     
├─nvme0n1p1 259:1    0     1G  0 part  /efi  
└─nvme0n1p2 259:2    0   118G  0 part     
 └─root    254:0    0   118G  0 crypt /mnt/hdd  
                                      /
```
As you can see my ssd(nvme0n1) has already been partitioned as I'm writing this on a live system, but yours should be empty(It's alright if it's not). 

Use `cfdisk` to partition disk:
```shell
cfdisk /dev/NAME-OF-YOUR-DISK
```

You should get something like this:
![[Pasted image 20240710211444.png]]
Delete any existing partition there is, and make two partitions:
- 1G efi partition(You can select the type in the Type tab when you select it).
- xG Linux root (x86-64) partition which will be your root partition(To select the type repeat the previous step).
Once you make these two partitions, press enter on Write and follow any steps that it tells you and you should be done with partitioning.

### Encyption
---
We will use luks encypt to encrypt our root partition, unlock and map it so we can use it.
Encrypting the Linux partition gives you some extra security if your machine’s disk is no longer in your possession, for instance:

- if the machine (or its disk) gets stolen
- if you need to send back the machine or the disk for replacement or repair (and can’t wipe the disk before)

On modern machines, the performance and CPU overhead of disk encryption is negligible.

A single hassle about disk encryption is that you'll have to input a passphrase every single time you boot into your machine. This will be solved when we use TPM.

To start name your root partition and encrypt it and follow that up by mapping it. It will ask you for some details which will be pretty simple.(Put a dummy passphrase such as "1234" we will change it later).
```shell
ROOTDEV=/dev/path-to-root-partition

cryptsetup luksFormat --type luks2 $ROOTDEV
cryptsetup luksOpen $ROOTDEV root

ROOTDEV=/dev/mapper/root
```
You have now fully encrypted your disk.

### Mounting and installing linux
This is all mostly copied and pasted from the Installation guide, and is fairly straightforward.
```shell
mkfs -t ext4 $ROOTDEV
mount --mkdir $ROOTDEV /mnt
mkfs -t vfat /dev/path-to-your-efi-partition
mount --mkdir /dev/path-to-your-efi-partition /mnt/efi
```
Configure your pacman mirrors:
```shell
reflector --save /etc/pacman.d/mirrorlist \
--protocol https --latest 5 --sort age
```
Now install the base system and a few extra packages:
```shell
pacstrap -K /mnt base linux linux-firmware linux-headers \
less sudo git base-devel networkmanager vim man-db man-pages openssh
```
Chroot into your system and finish it up:
```shell
arch-chroot /mnt

ln -sf /usr/share/zoneinfo/Region/City /etc/localtime

MYHOSTNAME=fancyhostnameoowee
MYUSERNAME=youruseername
ROOTPASSWORD=securerootpassword
USERPASSWORD=secureuserpassword

hwclock --systohc
echo en_US.UTF-8 UTF-8 >> /etc/locale.gen
locale-gen

echo $MYHOSTNAME > /etc/hostname

# Set a root password
chpasswd <<< root:$ROOTPASSWORD

# Create a user (strongly recommended :))
useradd -m $MYUSERNAME
chpasswd <<< $MYUSERNAME:$USERPASSWORD
echo "$MYUSERNAME ALL=(ALL) ALL" > /etc/sudoers.d/$MYUSERNAME

systemctl enable NetworkManager.service
systemctl enable sshd.service
```
### Setting up the bootloader
---
For the bootloader I've used systemd-boot as it's much more secure than grub.

```shell
bootctl install
```

Edit `/efi/loader/loader.conf`:\
```
console-mode auto
default @saved
timeout 10
```
At this point, we’re supposed to generate `/etc/fstab`, but we won’t do it. Instead, we’re going to use a “fancy” initrd, based on systemd, which will automatically detect our various partitions, using GPT partition types. That’s why we had to set the partition types correctly earlier.
### Generating the initrd
---
To generate the initrd, we need to first edit `/etc/mkinitcpio.conf` and update the `HOOKS` line to use the fancy systemd initrd mentioned previously:

```
HOOKS=(base systemd autodetect microcode modconf kms keyboard sd-vconsole block sd-encrypt filesystems fsck)
```

- `systemd` is here for the partition detection
- `sd-encrypt` will detect LUKS partitions and unlock them (prompting us for the password if necessary)

We can now build the initrd:

```
mkinitcpio --allpresets
```

Then we configure [systemd-boot](https://wiki.archlinux.org/title/Systemd-boot) to add a Linux entry:

```
cat >/efi/loader/entries/arch.conf <<EOF
title Arch Linux
linux /vmlinuz-linux
#initrd /amd-ucode.img
#initrd /intel-ucode.img
initrd /initramfs-linux.img
#options root=LABEL=xxx rootfstype=ext4
EOF
```

I’ve commented out the microcode files; feel free to install the relevant package for your CPU and uncomment the corresponding line.

I’ve also left a commented out `options` line just in case you don’t want to use partition autodetection.

At that point, we can already reboot into the newly installed system if want.
### Setting up SecureBoot
---
Now this might brick your Computer so be careful and follow the steps exactly as instructed.

Install the SecureBoot package:

```
pacman -S sbctl sbsigntools
```

check that “Setup Mode” is “Enabled”:

```
sbctl status
```
If not restart your Computer and go back into bios and get your secure boot status into "setup mode".

Create your own signing keys:

```
sbctl create-keys
```

Sign the systemd bootloader:

```
sbctl sign -s \
  -o /usr/lib/systemd/boot/efi/systemd-bootx64.efi.signed \
  /usr/lib/systemd/boot/efi/systemd-bootx64.efi
```

Enroll your custom keys:

```
sbctl enroll-keys --microsoft
```

Now we need to configure `mkinitcpio` so that it generates UKI in addition to “normal” initrds.

First, if you haven’t done it already, edit `/etc/mkinitcpio.conf` and update the `HOOKS` line:

```
HOOKS=(systemd autodetect microcode modconf kms keyboard sd-vconsole block sd-encrypt filesystems fsck)
```

Then edit `/etc/mkinitcpio.d/linux.preset`, and uncomment the `default_uki` and `fallback_uki` lines. Change the start of their paths from `/efi/..`  to `/efi/EFI/..`

Note: if `/etc/mkinitcpio.d/linux.preset` doesn’t exist, make sure that the `mkinitcpio` and `linux` packages are installed. Re-install them if necessary.

Build the new initrd and our new Unified Kernel Images:

```
mkinitcpio --allpresets
```

Now sign all your EFI binaries:
```
sbctl sign -s /efi/EFI/Linux/arch-linux.efi
sbctl sign -s /efi/EFI/Linux/arch-linux-fallback.efi
sbctl sign -s /efi/EFI/systemd/systemd-bootx64.efi
sbctl sign -s /efi/EFI/BOOT/BOOTX64.EFI
sbctl verify
```

We can now reboot using the new EFI images.
### Enrolling a TPM key
---
```
# Install the TPM tools
pacman -S tpm2-tools

# Check the name of the kernel module for our TPM
systemd-cryptenroll --tpm2-device=list

# Generate a recovery key (not mandatory but strongly recommended)
systemd-cryptenroll --recovery-key /dev/gpt-auto-root-luks

# Generate a key in the TPM2 and add it to a key slot in the LUKS device
systemd-cryptenroll --tpm2-device=auto /dev/gpt-auto-root-luks --tpm2-pcrs=7

```
Run `mkinitcpio --allpresets` one more time, reboot, and this time you shouldn’t have to enter a password to unlock the root volume!

### Finishing line.
---
You've done it. You finally setup your super cool and secure arch linux installation.
Now you can do whatever tweaking and installation you would like to do(Installing a DE or applications and such).

In the next following steps I'll show you the extra tweaks I was talking about and then this documentation will come to an end.

UNDER CONSTRUCTION!!!
