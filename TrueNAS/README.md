# Installation notes

HP MicroServers have limited flexibility for boot ordering; you cannot easily specify which SATA disk to boot from.
Experience shows that the boot order is usually something like this:
1. External USB
2. Internal USB/SD Slot
3. SATA Controller(s)
   1. Disk Bay 1-4
   2. Disk Bay 5 (ODD)

> [!NOTE]
> The 4 SATA slots plus the 5th SATA port used for the optical disk drive are connected to a "HPE Dynamic Smart Array B120i Controller". This is a fakeraid controller which uses drivers and firmware to pretend to be a RAID controller. Use it AHCI mode instead.  Bays 1+2 are SATA 6G, while 3+4 + the ODD are 3G, so you may not get a clean /dev/sda-d being those 4 disks. Often /dev/sdc is either the SD card or the ODD bay.

I highly recommend using the `/dev/disk/by-uuid/` route for addressing disks instead of `/dev/sdX`.

If you're installing plain Linux, my preferred route for booting is to stick `/boot` onto the SD card, `/` and other filesystems on the SSD in the optical bay, then configure an mdadm RAID with the other disks. Dealer's choice as to how you carve that up

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
> The above assumes you have 2 disks in the removable bays, and you've installed the OS to the disk in the optical bay. If you have a different number of disks, you need to use the appropriate `hdX` for the number of disks you actually have, not the number of bays), or add multiple menuentry sections for each disk in decreasing order.
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
Disk /dev/sdf: 465.76 GiB, 500107862016 bytes, 976773168 sectors
Disk model: WDC WDS500G1B0A-
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: 3C716C7D-7F01-4569-AB27-0B341DE795A6

Device      Start     End       Sectors     Size Type
/dev/sdf1   4096      6143      2048        1M BIOS boot
/dev/sdf2   6144      1054719   1048576     512M EFI System
/dev/sdf3   1054720   33554432  32499713    15.56 Solaris /usr & Apple ZFS

Command (m for help): n
Partition number (4-128, default 4):
First sector (34-976773134, default 33556480) : <-- This number
Last sector, +/-sectors or +/-sizeK,M,G,T,P (33556480-976773134, default 976773119): <-- This number
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
> Behold, the random disk detection ordering of Linux:
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
sdd      8:48   0  12.7T  0 disk <-- 14TB disk in bay 2
└─sdd1   8:49   0  12.7T  0 part 
sde      8:64   0   1.8T  0 disk <-- 2TB backup SSD in USB enclosure
└─sde1   8:65   0   1.8T  0 part
```

## 2.5GbE networking

The two onboard 1GbE NICs (`eno1`/`eno2`) are no longer used. They were
originally bonded together as an LACP link aggregation to a Netgear GS108T; that
configuration has been removed in favour of a single 2.5GbE Intel I226-V card in
the Gen8's PCIe slot.

> [!NOTE]
> The Gen8 has one PCIe x16 slot (x8 electrically), which is plenty for a 2.5GbE
> card. The I226-V is supported by the in-tree `igc` driver, so TrueNAS picks it
> up with no extra work. It enumerates as a `enp<bus>s0`-style name rather than
> `enoX`, so don't expect it to reuse the onboard naming.

1. Power off, fit the card, and reboot.
2. Navigate to `System > Network` and confirm the new interface is listed.
3. Click the new interface and configure it (DHCP, or a static address if you've
   moved off DHCP).
4. Remove any leftover `bond1` interface and clear the configuration on `eno1`
   and `eno2` so they don't grab a second address on the same subnet.
5. Click `Save`, then `Test Changes`.

> [!WARNING]
> TrueNAS gives you 60 seconds to commit network changes before it rolls them
> back. If you're changing the interface that carries the web UI, you may lose
> the session — reconnect on the new address and commit before the timer runs
> out, or the rollback will save you.

Add a dashboard widget for the interface to confirm it negotiates at 2500Mb/s.
If it comes up at 1000Mb/s, suspect the cable or the switch port rather than the
card — 2.5GBASE-T needs Cat5e or better and a switch port that actually does
multi-gig.

# SMB shares

Besides the two apps, this box is a backup target for the rest of the network:
Time Machine for the Macs, and vzdump for a Proxmox host. Both go over SMB.

> [!NOTE]
> These are generic steps against TrueNAS Scale 25.10 — substitute your own pool
> name for `tank` throughout. The SMB share `Purpose` presets get reshuffled
> between releases, so the label may not match exactly; what matters is that the
> Time Machine share ends up with Time Machine support enabled and the Proxmox
> share does not.

## Common setup

1. Turn the SMB service on: `System > Services > SMB`, start it and set it to
   start automatically.
2. Create a dedicated user per consumer under `Credentials > Users` (e.g.
   `timemachine` and `proxmox`). Give each one a password and no shell. Keeping
   them separate means a compromised Proxmox host can't touch the Mac backups.

## Time Machine share

1. Create a dataset: `Datasets > Add Dataset`, name `tank/timemachine`, with
   `Dataset Preset: SMB`.
2. Set a quota on it. This is the only thing standing between Time Machine and
   the whole pool — it will happily grow until the disk is full otherwise.
3. `Shares > Windows (SMB) Shares > Add`:
   - Path: `/mnt/tank/timemachine`
   - Name: `timemachine`
   - Purpose: the Time Machine preset (`Multi-user time machine` on older
     releases)
4. Under the share's `Edit Filesystem ACL`, give the `timemachine` user full
   control and strip the other entries.
5. On the Mac: `Finder > Go > Connect to Server`, `smb://microserver/timemachine`,
   authenticate as the `timemachine` user and tick "Remember this password in my
   keychain". Then add it under `System Settings > General > Time Machine >
   Add Backup Disk`.

> [!TIP]
> The first backup over the network is slow regardless of link speed. Leave it
> running overnight; the 2.5GbE link mostly pays off on the incrementals and on
> restores.

## Proxmox backup share

1. Create a dataset `tank/proxmox` with `Dataset Preset: SMB`.
2. `Shares > Windows (SMB) Shares > Add`:
   - Path: `/mnt/tank/proxmox`
   - Name: `proxmox`
   - Purpose: the default/basic share preset — **not** the Time Machine one
3. Set the filesystem ACL so the `proxmox` user has full control.
4. On the Proxmox host, add it as a storage target under
   `Datacenter > Storage > Add > SMB/CIFS`:
   - ID: `microserver`
   - Server: the MicroServer's address
   - Username / Password: the `proxmox` user
   - Share: `proxmox`
   - Content: `VZDump backup file`
   - Set a retention policy so old dumps get pruned

   Or from the Proxmox shell:
```
pvesm add cifs microserver \
  --server 192.168.1.10 \
  --share proxmox \
  --username proxmox \
  --content backup \
  --prune-backups keep-daily=7,keep-weekly=4,keep-monthly=6
```
   Leave `--password` off entirely and `pvesm` prompts for it, which keeps it
   out of your shell history. It gets stored in `/etc/pve/priv/storage/`.
5. Schedule the backup job under `Datacenter > Backup`, targeting the new
   storage.

> [!NOTE]
> vzdump writes large sequential files, so this is the workload that most
> benefits from the 2.5GbE card. It's also the one most likely to saturate the
> spinning disks rather than the network.

# Apps

Only two apps run on this server, both installed from the TrueNAS app catalogue
onto the `apps` pool carved out of the OS SSD (see above):

- **Jellyfin** — movies and TV. Media lives on the 14TB pool and is mounted into
  the container as a host path.
- **Immich** — photos. Same arrangement for the library.

The Xeon E3-1265L v2 has Quick Sync, but it's Ivy Bridge-era and only useful for
a limited set of transcodes. Direct play wherever possible.

# TrueNAS configuration (ToDo)
- Hostname + custom domain
- Replication of the app datasets to the 2TB USB backup SSD
