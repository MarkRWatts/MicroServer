# MicroServer
Notes on use of an HP ProLiant MicroServer in 2025 for self hosting.

![image](https://github.com/user-attachments/assets/7d7cc73d-4575-4767-a445-345dc6caa1f5)

## Specification
This server is an HP ProLiant MicroServer Gen8:
- Intel Xeon E3-1265L V2 processor (£20 shipped from China through eBay)
- 16GB (2x8GB) Samsung DDR3 1600 ECC (£26 shipped from the UK through eBay)
- 4GB SanDisk microSD card (for OS booting) in the internal slot
- 2x Toshiba N300 14TB NAS drives
- 1x WD Blue 500GB SSD (for the OS) in the optical disk bay
- iLO 4 Advanced
- 1x 2TB Crucial P310 SSD in a USB 3.2 enclosure (for backups)

## The Plan

- Install TrueNAS Scale, making efficient use of available storage
- Install a bunch of Apps to self-host some things
  - Tailscale
  - Jellyfin (movies)
  - Roon (music)
  - Unifi Network
  - Immich (photos)
  - Homarr (dashboard)

# Installation notes

HP MicroServers have limited flexibility for boot ordering; you cannot easilly specify which SATA disk to boot from.
Exierience shows that the boot order is usually something like this:
1. External USB
2. Internal USB/SD Slot
3. SATA Controller(s)
   1. Disk Bay 1-4
   2. Disk Bay 5 (ODD)

> [!NOTE]
> The 4 SATA slots plus the 5th SATA port used for the optical disk drive are connected to a "HPE Dynamic Smart Array B120i Controller". This is a fakeraid controller which uses drivers and firmware to pretend to be a RAID controller. Use it AHCI mode instead.  Bays 1+2 are SATA 6G, while 3+4 + the ODD are 3G, so you may not get a clean /dev/sda-d being those 4 disks. Often /dev/sdc is either the SD card or the ODD bay.

I highly recommend using the `/dev/disk/by-uuid/` route for addressing disks instead of `/dev/sdX`.

If you're installing plain Linux, my prefered route for booting is to stick `/boot` onto the SD card, `/` and other filesystems on the SSD in the optical bay, then configure an mdadm RAID with the other disks. Dealers choice as to how you carve that up

# TrueNAS
> [!CAUTION]
> This install will WIPE all of your disks. You have been warned.

I used TrueNAS Scale 25.10.0.

1. Download the ISO and burn it to a USB stick with your choice of software. I used [Balena Etcher](https://etcher.balena.io/)
2. Boot the MicroServer from the USB stick. NB: If it's a USB 3.0 stick, use the blue ports on the rear. All the black ones are USB 2.0
> [!WARNING]
> I'm going to do an unsupported install of the OS using a 16GB filesystem on the SSD as I want to use the rest of it for apps.
3. If you have access to the iLO HTML5 console use that to do the install, otherwise use a standard KVM.
4. Select “Shell” from the Console Setup menu.
5. Execute the following command to modify the in-memory installer script:
```
sed -i 's/-n3:0:0/-n3:0:+16384M/g' /usr/lib/python3/dist-packages/truenas_installer/install.py
```
6. Type `exit` to return to the installer
7. Select “Install/Upgrade” from the Console Setup menu (without rebooting, first) and install to the SSD.
8. Remove the TrueNAS USB stick, insert the Ubuntu USB stick, and reboot.

## grub bootloader

1. Boot into an Ubuntu Live USB
2. Drop to a console (alt-F2 at the first installer screen)
3. Determine which `/dev/sdX` device your microSD card is (use `lsblk` and look for the one which is 4GB/3.7GiB in size)
4. Use `fdisk` to setup the card for booting. (These are abridged instructions).
```
fdisk /dev/sdX

Delete existing partitions (d)
Set the GPT partition type (g)
Create partitions (n)
  /dev/sdX1 = 1M BIOS Boot
  /dev/sdX2 = rest of disk, default linux partition type
Set the type of partition 1 to 'BIOS Boot' (t,5)
```
5. Create a filesystem
```
mkfs.ext2 /dev/sdX2
```
6. Setup Grub
```
mkdir /tmp/usb
mount /dev/sdX2 /tmp/usb
mkdir /tmp/usb/boot
grub-install --boot-directory=/tmp/usb/boot /dev/sdX
```
7. Edit `/tmp/usb/boot/grub/grub.cfg`. Fill it with:
```
set default='0'
set timeout='0'

menuentry 'TrueNAS' {
	set root=(hd3)
	chainloader +1
}
```
> [!NOTE]
> The above asssumes you have 2 disks in the removable bays, and you've installed the OS to the disk in the optical bay. If you have a different number of disks, you need to use the appropriate `hdX` for the number of disks you actually have, not the number of bays), or add multiple menuentry sections for each disk in decreasing order.
8. Unmount the microSD card and reboot, hopefully into TrueNAS
```
cd / 
umount /tmp/usb
reboot
```
## ZFS filesystem on the rest of the SSD
> [!CAUTION]
> Again, this is an unsupported configuration, but I plan to backup the apps datasets anyway, so it's a risk I'll take.
> It's based on (https://gist.github.com/gangefors/2029e26501601a99c501599f5b100aa6)

1. On the TrueNAS console, enter the `shell` and become root with `sudo su`
> [!NOTE]
> `parted` will probably complain about not being able to align the partition correctly if you just use that with the next available sector to start, and the last available sector to end the new partition. Easiest fix I've found here is to use `fdisk` to get the first/last sectors, then use `parted` to actually create the partition as per the above guide.
```
root@microserver[/]# fdisk /dev/sdf

Welcome to fdisk (util-linux 2.38.1).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

This disk is currently in use - repartitioning is probably a bad idea.
It's recommended to umount all file systems, and swapoff all swap partitions on this disk.

Command (m for help): p
Disk /dev/sdf: 465.76 GiB, 500107862016 bykes, 976773168 sectors
Disk model: WDC WDS500G1B0A-
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/0 size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: 3C716C7D-7F01-4569-AB27-0B341DE795A6

Device      Start     End       Sectors     Size Type
/dev/sdf1   4096      6143      2048        1M BIOS boot
/dev/sdf2   6144      1054719   1048576     512M EFI System
/dev/sdf3   1054720   33554432  32499713    15.56 Solaris /usr & Apple ZFS

Command (m for help): n
Partition number (4-128, default 4):
First sector (34-976773134, default 33556480) : <-- This number
Last sector, +/-sectors or +/-sizeK.M,G,T.P3 (33556480-976773134, default 976773119): <-- This number
```
> The highlighted numbers are what you'll need to use in `parted`
2. In `parted`
```
root@microserver[/]# parted /dev/sdf
(parted) mkpart apps 33556480s 976773119s
```
3. Now create the ZFS pool
```
root@microserver[/]# zpool create apps /dev/sdf4
root@microserver[/]# zpool export apps
```
4. Import the pool into TrueNAS
    - Storage > Import Pool
    - Select `apps` from the dropdown

## TrueNAS final disk layout
> Behold, the random disk dection ordering of Linux:
```
truenas_admin@microserver[~]$ lsblk
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sda      8:0    0  12.7T  0 disk <-- 14TB disk in bay 1
└─sda1   8:1    0  12.7T  0 part 
sdb      8:16   0   3.7G  0 disk <-- microSD card in internal slot
├─sdb1   8:17   0     1M  0 part 
└─sdb2   8:18   0   3.7G  0 part 
sdc      8:32   0 465.8G  0 disk <-- 500GB SSD in ODD bay
├─sdc1   8:33   0     1M  0 part 
├─sdc2   8:34   0   512M  0 part 
├─sdc3   8:35   0  15.5G  0 part 
└─sdc4   8:36   0 449.8G  0 part 
sdd      8:48   0  12.7T  0 disk <-- 14TB disk in bay 1
└─sdd1   8:49   0  12.7T  0 part 
sde      8:64   0   1.8T  0 disk <-- 2TB backup SSD in USB enclosure
└─sde1   8:65   0   1.8T  0 part
```

## LACP ethernet link aggregation (LAG)

I have a Netgear GS108T managed 8-port gigabit switch. This supports LACP link aggregation. So does TrueNAS.

> [!TIP]
> There's a potential race condition here as you need to configure both ends of the link at more-or-less the same time. Due to the way link aggregation works, the aggregated interface `bond1` in Linux will use the MAC address of the first interface, and if you're using DHCP (for now) you'll end up with the same IP as before. This *should* mean you don't end up being unable to connect to the web interface of TrueNAS.

### On the switch
1. Identify the two switch ports you've connected the two server interfaces into. Ports 6 & 7 for me.
2. Login to the switch (default password is `password`), and navigate to `Switching > LAG`
3. Check the box next to `LAG1`, change the `LAG Type` to `LACP` and hit `Apply`
4. Click on `LAG Membership`, and expand the dropdown in the orange box for `LAG1`
5. Check the boxes for the ports you want in the LAG, and hit `Apply`

### In TrueNAS
1. Navigate to `System > Network`
2. Click `Add` under `Interfaces`
   - Type: Link Aggregation
   - Name: bond1
   - DHCP: Get IP Address Automatically from DHCP
   - Link Aggregation Protocol: LACP (leave everything else in this section as-is)
   - Link Aggregation Interfaces: eno1, eno2
3. Click `Save`

> [!NOTE]
> that TrueNAS will give you 60 seconds to commit the change before it does a roll-back. Do some quick testing then commit.
> If you add a dashboard widget monitoring the `bond1` interface, you should see it listed as 2000Mb/s.


# TrueNAS configuration (ToDo)
- Hostname + custom domain
