## Introduction

I want a full-disk encrypted, deniable installation of NixOS, and to
format my disk using ZFS, with a header/boot on a separate device.
Here's how I did it.

The boot is currently not encrypted because I ran into an issue I don't
have time to troubleshoot right now. It's stored on an SD card or USB
drive. So I intend to eventually alter this setup so that the kernel in
my boot is encrypted, and use ZFS to check it for consistency.

This system was built from an existing NixOS build, with internet
conenection. NixOS can be installed from another partition, from your
existing NixOS distribution, from the same partition (in place), from
your existing non-NixOS Linux distribution using NIXOS_LUSTRATE, or from
the Live CD of any Linux distribution. You can also create bootable
.isos from your existing partition, though you need internet access to
cross-compile unless you've already downloaded the dependencies.

## Identifying Disks

Here I determined which disks I'd be writing to, and which I'd be
leaving alone.

```
$ lsblk
NAME          MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINT
sda             8:0    0 465.8G  0 disk  
├─sda1          8:1    0   499M  0 part  
└─sda2          8:2    0 465.3G  0 part  
sdb             8:16   1 476.7G  0 disk  
├─sdb1          8:17   1   549M  0 part  /boot
└─sdb2          8:18   1 476.2G  0 part  
  └─luksroot  254:0    0 476.2G  0 crypt 
        ├─vg-swap 254:1    0     8G  0 lvm   [SWAP]
        └─vg-root 254:2    0 468.2G  0 lvm   /
sde             8:64   1  29.8G  0 disk  
nvme0n1       259:0    0 476.9G  0 disk  
├─nvme0n1p1   259:1    0  17.1G  0 part  
├─nvme0n1p2   259:2    0   100M  0 part  
├─nvme0n1p3   259:3    0    16M  0 part  
├─nvme0n1p4   259:4    0 458.8G  0 part  
└─nvme0n1p5   259:5    0  1000M  0 part  
```

`sda` is the target disk for the filesystem/OS
`sdb` was the existing NixOS partition, running on a microSD card.
`sde` is the microSD card which will contain `/boot` and the LUKS header.
`nvme0n1` is the windows partition, and is left alone.

If the microSD card isn't plugged in, sda should look like it contains
random noise, and the laptop will boot to the windows partition.

## Preparing Disks

Here I fill both the microSD `/boot` partition and primary device with
random data, to make the size of the encrypted content undeterminable.
From here, all commands are entered from root ($`sudo su -`, anything
with `#` at the start of the line indicates `root`)

The typical method is to use `dd if=/dev/urandom of=/dev/sda` but this
takes hours to complete. So while I was waiting I learned about and
searched for a faster method, which I didn't find.

There are several options for filling something with random data:
[`random` `arandom` `urandom` `frandom`] the first is most secure, but
slowest. The last one is fastest, but least secure.
<https://wiki.archlinux.org/title/frandom> warns that `frandom`
shouldn't be used for randomizing large hard drives prior to
data-at-rest encryption, since it uses the vulnerable RC4 algorithm,
which is distinguishable from random noise.

`urandom` is recommended.

The time it takes to do this could be sped up by changing the block size
<https://wiki.archlinux.org/title/Securely_wipe_disk> &
<https://wiki.archlinux.org/title/Advanced_Format> of the dd command
with `bs=`

Wiping the disk on the laptop & filling it with random data:

```
# dd if=/dev/urandom of=/dev/sda
# dd if=/dev/urandom of=/dev/sde 
```

## Creating Partitions, Formatting boot


For the primary drive:
```
# parted /dev/sda -- mklabel gpt
# parted /dev/sda -- mkpart primary 0% 100%
```

For the usbstick or SD card:
```
# parted /dev/sde -- mklabel gpt
# parted /dev/sde -- mkpart ESP fat32 0% 50%
# parted /dev/sde -- set 1 boot on
# parted /dev/sde -- mkpart primary 50% 100%
# mkfs.fat -F 32 -n boot /dev/sde1
```

"It is not necessary to make a filesystem for the LUKS2 header partition
/dev/sdb2. This is because the LUKS2 header will reside as a raw header
without any underlying file system. This avoids the complications of the
bootloader having to mount a filesystem before reading the header file."

## Luks1 Encryption Container with Detatched Headers

This is one area where I ran into an issue. LUKS2 is better in that it
uses the argon2 hash algorithm for passwords, which is more secure than
the LUKS1 PBKDF2 algorithm. But when I originally did this I ran into an
issue with grub failing to read the header.

The default for LUKS is to use a 256 bit key. I want to use a 512 bit
key.

```
# cryptsetup luksFormat  --type luks1 -c aes-xts-plain64 -s 256 -h sha512  /dev>
# dd if=/dev/urandom of=./keyfile0.bin bs=1024 count=4
# cryptsetup luksAddKey /dev/sda1 keyfile0.bin --header /dev/sde2
# cryptsetup luksOpen /dev/sda1 crypted-nixos -d keyfile0.bin --header /dev/sde2

```

The cryptsetup commands will prompt for a password.

## Set up LVM

I'm not using the typical name for the volume group, 'vg' because 'vg'
already exists on my device! My former NixOS partition also used lvm.
`pvdisplay` and `vgdisplay` can be used to check the volumes.

```
# pvcreate /dev/mapper/crypted-nixos
# vgcreate vg1 /dev/mapper/crypted-nixos
# lvcreate -L 15.5G -n swap vg1
# lvcreate -l '100%FREE' -n root vg1
```

## Create Filesystem and Swap

* I'm copying from <https://grahamc.com/blog/erase-your-darlings> for
layout.

* ZFS apparently has its own encryption, with the tradeoffs:
https://arstechnica.com/gadgets/2021/06/a-quick-start-guide-to-openzfs-native-e>

* With encrypted ZFS you can send the raw, still-encrypted data stream
to a remote site, without it being decrypted on the other end. I can't
tell if this will cause issues with keyfile storage among other things,
and I don't imagine many usecases where I'll want to send raw encrypted
data. And it's also buggy:
<https://docs.google.com/spreadsheets/d/1OfRSXibZ2nIE9DGK6swwBZXgXwdCPKgp4SbPZw>

* ZFS here is primarily being used for the compression and checksumming.

* If `/boot` resides on ZFS when using GRUB you must only enable
features supported by GRUB otherwise GRUB will not be able to read the
pool.

* i used the cheatcheat here:
<https://web.archive.org/web/20220112193205/https://jrs-s.net/2018/08/17/zfs-tu>

Create the pool:

```
# POOL=rpool # insert your name here.
# zpool create -f  -O atime=off -O xattr=sa -O mountpoint=none "${POOL}"  /dev/>
```

Create root:

```
# zfs create -p -o compression=on -o mountpoint=legacy "${POOL}/local/root"
# zfs snapshot "${POOL}/local/root@blank" # for stateless only. see: <https://g>
```

Create other files:

```
# zfs create -o compression=on -o mountpoint=legacy "${POOL}/local/nix"
# zfs create -o compression=on -o mountpoint=legacy "${POOL}/safe/home"
# zfs create -o compression=on -o mountpoint=legacy "${POOL}/safe/persist"
# zfs create -o compression=on -o refreservation=10G -o quota=10G -o mountpoint>
```

Swap:

```
# mkswap -L swap /dev/vg1/swap
```

## Mount the Pools
```
# mount -t zfs "${POOL}/local/root" /mnt
# mkdir /mnt/nix
# mount -t zfs "${POOL}/local/nix" /mnt/nix
# mkdir /mnt/home
# mount -t zfs "${POOL}/safe/home" /mnt/home
# mkdir /mnt/persist
# mount -t zfs "${POOL}/safe/persist" /mnt/persist
```

```
# mkdir -p /mnt/boot
# mount /dev/sde1 /mnt/boot
# swapoff -a
# swapon /dev/vg1/swap # running swap on an SSD will wear down the SSD.
```

## Save the Keys

```
# mkdir -p /mnt/persist/secrets/initrd/
# cp keyfile0.bin  /mnt/persist/secrets/initrd
# chmod 000 /mnt/persist/secrets/initrd/keyfile*.bin
```

## NixOS Configuration

In general, I recommend building with the smallest possible
configuration file first, to make it easier to troubleshoot.

This would go into `/etc/nixos/configuration.nix`:

```
  boot.initrd.luks.devices = { # nixos must be informed of luks
     crypted = { # you will want the blkid of the device + header
        device = "/dev/disk/by-partuuid/<yours goes here>"; # $ blkid
        header = "/dev/disk/by-partuuid/<yours goes here>"; # $ blkid
        allowDiscards = true; # device = SSD
        preLVM = true;
     };
  };
  networking = {
     hostName = "<yours here>";
     hostId = "<yours here>"; # $ head -c 8 /etc/machine-id
     wireless.enable = false; # wpa_supplicant or network manager
     networkmanager.enable = true; # cannot be enabled with wpa_supplicant
     useDHCP = false; # global, deprecated
     interfaces = {
        wlo1.useDHCP = true;
        enp2s0.useDHCP = true;
     };
  };


services.zfs.trim.enable = true;
boot.loader.grub.copyKernels = true;  # https://nixos.wiki/wiki/ZFS#What_works
boot.kernelParams = [ "nohibernate" ];
boot.kernelParams = [ "zfs.zfs_arc_max=2884901888" ];
services.zfs.autoScrub.enable = true;
services.zfs.autoScrub.pools = [ "zpool" ]; # your pool name here

grub.efiInstallAsRemovable = true;

services.nfs.server.enable = true;

networking.firewall.allowedTCPPorts = [ 2049 ]; # 2049=nfs, 

nix.gc.automatic = true; # reclaim disk space by deleting unused packages
nix.settings.auto-optimise-store = true; # reclaim disk space by optimising nix>
networking.hostId = "#####"; #requirement for ZFS do `head -c 8 /etc/machine-id>

swapDevices = lib.mkForce [ ]; # https://nixos.wiki/wiki/Swap
```

If you create a zpool in the installer, make sure you run zpool export
<pool name> after nixos-install, or else when you reboot into your new
system, zfs will fail to import the zpool.

```
# zfs set sharenfs="ro=192.168.1.0/24,all_squash,anonuid=70,anongid=70" rpool/m>
```

Then run `nixos-install`.

## What if Something Goes Wrong?

If the installation fails, or you are unable to boot into the new OS,
you may need to run some commands to get into the configuration file to
find out what went wrong.

Re-open the encrypted drive using this command:

```
# cryptsetup luksOpen /dev/sda1 crypted-nixos -d keyfile0.bin --header /dev/sdc2
```

Then import the zpool. You may need the ID of the zpool.

```
# zpool import -d /dev/mapper/vg1-root
# zpool import <ID here>
```

Then mount all the folders again:

```
# mount -t zfs zpool/local/root /mnt
# mount -t zfs zpool/local/nix /mnt/nix
# mount -t zfs zpool/safe/home /mnt/home
# mount -t zfs zpool/safe/persist /mnt/persist
# mount /dev/sdc1 /mnt/boot
```

## Configuration

Once it boots and is able to connect to the internet, etc., then install
your usual configuration.

## Credits
https://shen.hong.io/installing-nixos-with-encrypted-root-partition-and-seperate-boot-partition/
https://gist.github.com/ladinu/bfebdd90a5afd45dec811296016b2a3f
