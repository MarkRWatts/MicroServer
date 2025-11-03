# MicroServer
Notes on use of an HP ProLiant MicroServer in 2025 for self hosting

## Specification
This server is an HP ProLiant MicroServer Gen8:
- Celeron G1610T processor
  - To be upgraded with an Intel Xeon E3-1265L V2 processor
- 16GB (2x8GB) Samsung DDR3 1600 ECC
- 4GB SanDisk microSD card (for OS booting) in the internal slot
- 4x Seagate 500GB SATA disks (for data) in the front drive bays
  - Likely to be replaced with a pair of ~20TB drives
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

HP MicroServer's have limited flexibility for boot ordering; you cannot easilly specify which SATA disk to boot from.
Exierience shows that the boot order is usually something like this:
1. External USB
2. Internal USB/SD Slot
3. SATA Controller(s)
   1. Disk Bay 1-4
   2. Disk Bay 5 (ODD)

Note that the 4 SATA slots are on two different controllers (1+2 are SATA 6G, 3+4 are 3G) so you may not get a clean /dev/sda-d being those 4 disks. Often /dev/sdc is either the SD card or the ODD bay. On Ubuntu I see:
```
/dev/sda = Bay 1
/dev/sdb = Bay 2
/dev/sdc = microSD
/dev/sdd = Bay 3
/dev/sde = Bay 4
/dev/sdf = ODD
```
I highly recommend using the `/dev/disk/by-uuid/` route for addressing disks instead of `/dev/sdX`.

If you're installing plain Linux, my prefered route for booting is to stick `/boot` onto the SD card, `/` and other filesystems on the SSD in the optical bay, then configure an mdadm RAID with the other disks. Dealers choice as to how you carve that up

# TrueNAS
> **This install will WIPE all of your disks. You have been warned**

I used TrueNAS Scale 25.10.0.

1. Download the ISO and burn it to a USB stick with your choice of software. I used [Balena Etcher] (https://etcher.balena.io/
2. Boot the MicroServer from the USB stick. NB: If it's a USB 3.0 stick, use the blue ports on the rear. All the black ones are USB 2.0
> **I'm going to do an unsupported install of the OS using a 16GB filesystem on the SSD as I want to use the rest of it for apps.**
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
3. Determine which `/dev/sdX` device your microSD card is (use lsblk and look for the one which is 4GB/3.7GiB in size)
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
	set root=(hd5)
	chainloader +1
}
```
> NB: The above asssumes you have all 4 disks and you've installed the OS to the disk in the optical bay. If you have fewer disks, you need to use the appropriate `hdX` for the number of disks you actually have, not the number of bays), or add multiple menuentry sections for each disk in decreasing order.
8. Unmount the microSD card and reboot, hopefully into TrueNAS
```
cd / 
umount /tmp/usb
reboot
```
