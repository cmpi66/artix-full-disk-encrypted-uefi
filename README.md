# Artix Full Disk encryption install

A guide on how to install Artix  with full disk encryption using LUKS1, UEFI and BTRFS snapshots with subvolumes. We're also adding some hardening configurations to our volumes and implementing Apparmor.

First we'll securely wipe our drive, this is recommended even if your drives are brand new. Although it will take time even on an NVME drive. The recommended amount of times to pass it through would be three but you decide how many times you want to do it. The reason why we're doing this is to basically write over the free storage space with one's and zero's to make the data unrecoverable for rogue individuals.

`shred --verbose --random-source /dev/urandom --iterations 3 /dev/nvme0n1`

Create two partitions with fdisk one for root and the other for boot and make the boot type efi, give efi about 512Mb or 1GB and the rest to the root partition:

`fdisk /dev/nvme0n1`

Encrypt and open disk:

`cryptsetup luksFormat --type=luks1 /dev/nvme0n1p2`

`cryptsetup luksOpen /dev/nvme0n1p2 crypt`

Create the filesystems:

`mkfs.vfat -F32 /dev/nvme0n1p1`

`mkfs.btrfs /dev/mapper/crypt`

## BTRFS setup 

This is where we'll create our subvolumes and bind them to the correct locations on our system:

```
BTRFS="/dev/mapper/crypt"
```

```
mount -o clear_cache,nospace_cache $BTRFS /mnt
```

```
btrfs su cr /mnt/@ 
btrfs su cr /mnt/@/.snapshots 
mkdir -p /mnt/@/.snapshots/1 
btrfs su cr /mnt/@/boot/
btrfs su cr /mnt/@/home btrfs su cr /mnt/@/root
btrfs su cr /mnt/@/srv 
btrfs su cr /mnt/@/opt 
btrfs su cr /mnt/@/tmp 
btrfs su cr /mnt/@/var
btrfs su cr /mnt/@/usr_local
btrfs su cr /mnt/@/cryptkey 
```

```
chattr +C /mnt/@/boot
chattr +C /mnt/@/srv
chattr +C /mnt/@/var
chattr +C /mnt/@/usr_local
chattr +C /mnt/@/opt
chattr +C /mnt/@/tmp
chattr +C /mnt/@/cryptkey
```

```
btrfs subvolume set-default "$(btrfs subvolume list /mnt | grep "@/.snapshots/1/snapshot" | grep -oP '(?<=ID )[0-9]+')" /mnt
```

```
cat << EOF >> /mnt/@/.snapshots/1/info.xml
<?xml version="1.0"?>
<snapshot>
  <type>single</type>
  <num>1</num>
  <date>1999-03-31 0:00:00</date>
  <description>First Root Filesystem</description>
  <cleanup>number</cleanup>
</snapshot>
EOF
```
```
chmod 600 /mnt/@/.snapshots/1/info.xml

umount /mnt

mount -o ssd,noatime,space_cache,compress=zstd:15 $BTRFS /mnt

mkdir -p /mnt/{boot,root,home,.snapshots,srv,tmp,var,opt,usr/local,cryptkey}
```

```
mount -o ssd,noatime,space_cache=v2,autodefrag,compress=zstd:15,discard=async,nodev,nosuid,noexec,subvol=@/boot $BTRFS /mnt/boot
mount -o ssd,noatime,space_cache=v2,autodefrag,compress=zstd:15,discard=async,nodev,nosuid,subvol=@/root $BTRFS /mnt/root
mount -o ssd,noatime,space_cache=v2,autodefrag,compress=zstd:15,discard=async,nodev,nosuid,subvol=@/home $BTRFS /mnt/home
mount -o ssd,noatime,space_cache=v2,autodefrag,compress=zstd:15,discard=async,nodev,nosuid,subvol=@/usr_local $BTRFS /mnt/usr/local
mount -o ssd,noatime,space_cache=v2,autodefrag,compress=zstd:15,discard=async,subvol=@/.snapshots $BTRFS /mnt/.snapshots
mount -o ssd,noatime,space_cache=v2,autodefrag,compress=zstd:15,discard=async,subvol=@/srv $BTRFS /mnt/srv
mount -o ssd,noatime,space_cache=v2,autodefrag,compress=zstd:15,discard=async,nodev,nosuid,subvol=@/var $BTRFS /mnt/var
mount -o ssd,noatime,space_cache=v2,autodefrag,compress=zstd:15,discard=async,nodev,nosuid,subvol=@/opt $BTRFS /mnt/opt
mount -o ssd,noatime,space_cache=v2,autodefrag,compress=zstd:15,discard=async,nodev,nosuid,subvol=@/tmp $BTRFS /mnt/tmp
mount -o ssd,noatime,space_cache=v2,autodefrag,compress=zstd:15,discard=async,nodatacow,nodev,nosuid,noexec,subvol=@/cryptkey $BTRFS /mnt/cryptkey
```

```
mkdir -p /mnt/boot/efi
mount -o nodev,nosuid,noexec /dev/nvme0n1p1 /mnt/boot/efi
```


## Basestrap
```
basestrap /mnt base base-devel openrc elogind-openrc cryptsetup cryptsetup-openrc mkinitcpio linux-hardened linux-hardened-headers intel-ucode openssh openssh-openrc grub linux-firmware grub wpa_supplicant wpa_supplicant-openrc os-prober efibootmgr networkmanager-openrc networkmanager git ansible cups cups-openrc cronie cronie-openrc pcsclite pcsclite-openrc syslog-ng syslog-ng-openrc postgresql postgresql-openrc docker docker-openrc nm-connection-editor firewalld firewalld-openrc apparmor apparmor-openrc snapper lvm2-openrc lvm2 snapper vim nfs-utils nfs-utils-openrc audit-openrc usbutils bubblewrap-suid dnsmasq dnsmasq-openrc
```


## If you want qemu extra that can't be basestrapped
`qemu-full virt-manager virt-viewer dnsmasq vde2 openbsd-netcat bridge-utils libvirt-openrc`


## fstab

`fstabgen -U /mnt >> /mnt/etc/fstab`

### BTRFS cont.
Change this to the right subvol number

```
sed -i 's#,subvolid=257,subvol=/@/.snapshots/1/snapshot,subvol=@/.snapshots/1/snapshot##g' /mnt/etc/fstab
```

## Chroot into system

```
artix-chroot /mnt bash
ln -sf /usr/share/zoneinfo/America/New_York /etc/localtime
hwclock --systohc
```

`vim /etc/locale.gen`
`locale-gen`

``` 
echo export LANG="en_US.UTF-8" >> /etc/locale.conf
echo export LC_COLLATE="C" >> /etc/locale.conf
```

Edit mkinitcpio config file:

`sed -i "s/modconf block/modconf block encrypt/g" /etc/mkinitcpio.conf`


### BTRFS contd.

`sed -i 's,#COMPRESSION="zstd",COMPRESSION="zstd",g' /etc/mkinitcpio.conf`

Create key so you won't have to put in your password twice during boot:

```
dd if=/dev/random of=/cryptkey/crypto_keyfile.bin bs=512 count=8 iflag=fullblock
chmod 000 /cryptkey/crypto_keyfile.bin
sed -i 's#FILES=()#FILES=(/cryptkey/crypto_keyfile.bin)#g' /etc/mkinitcpio.conf
cryptsetup luksAddKey /dev/nvme0n1p2 /cryptkey/crypto_keyfile.bin
```


### BTRFS Snapper conf
```
umount /.snapshots
rm -r /.snapshots
snapper --no-dbus -c root create-config /
btrfs subvolume delete /.snapshots
mkdir /.snapshots
mount -a
chmod 750 /.snapshots
```

### BTRFS Contd.

Configuring Grub with key and UUID:

```
sed -i "s#quiet#cryptdevice=UUID=`blkid -s UUID -o value /dev/nvme0n1p2`:crypt root=/dev/mapper/crypt lsm=landlock,lockdown,yama,apparmor,bpf cryptkey=rootfs:/cryptkey/crypto_keyfile.bin#g" /etc/default/grub
```


```
sed -i "s/#GRUB_ENABLE_CRYPTODISK/GRUB_ENABLE_CRYPTODISK/g" /etc/default/grub

echo -e "# Booting with BTRFS subvolume\nGRUB_BTRFS_OVERRIDE_BOOT_PARTITION_DETECTION=true" >> /etc/default/grub
```

## mkinit

Rebuild mkinit image:

`mkinitcpio -P`

Install Grub:

`grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=GRUB --modules="normal test efi_gop efi_uga search echo linux all_video gfxmenu gfxterm_background gfxterm_menu gfxterm loadenv configfile gzio part_gpt cryptodisk luks gcry_rijndael gcry_sha256 btrfs" --disable-shim-lock`

`grub-mkconfig -o /boot/grub/grub.cfg`

Add parallel downloads to pacman conf:

`sed -Ei 's/^#(Color)$/\1\nILoveCandy/;s/^#(ParallelDownloads).*/\1 = 10/' /etc/pacman.conf`

## Finalizing, and adding user and services
```
passwd
useradd -m -G users,wheel,audio,video,docker -s /bin/bash USER
passwd USER 
```


Configure sudoers file to allow wheel sudo access:

`visudo`


## APParmor
```
sed -i 's/#write-cache/write-cache/g' /etc/apparmor/parser.conf
sed -i 's,#Include /etc/apparmor.d/,Include /etc/apparmor.d/,g' /etc/apparmor/parser.conf
echo 'Optimize=compress-fast' | sudo tee -a /etc/apparmor/parser.conf
```

Add a hostname:

`vim /etc/hostname`

`vim /etc/hosts`

```
127.0.0.1        localhost
::1              localhost
127.0.1.1        local.localdomain  local
```

Initiate postgresql:

`su - postgres`

`initdb /var/lib/postgres/data`

## configure services

```
rc-service NetworkManager start
rc-update add NetworkManager default

rc-service postgresql start
rc-update add postgresql default

rc-service cupsd start
rc-update add cupsd default

rc-service cronie start
rc-update add cronie default

rc-service pcscd start
rc-update add pcscd default

rc-service syslog-ng start
rc-update add syslog-ng default

rc-service libvirtd start
rc-update add libvirtd default

rc-service docker start
rc-update add docker default

rc-update add apparmor boot
rc-update add auditd default
rc-update add firewalld default
```

```
exit 
umount -R /mnt
reboot
```


## afterward if you used a simple password just to get things up and running

`cryptsetup luksChangeKey /dev/nvme0n1p2`

