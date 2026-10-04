# MicroServer
Notes on use of an HP ProLiant MicroServer in 2025/26 for self hosting.

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
- 1x 2.5GbE Intel I226-V NIC in the PCIe slot

## What it actually does

The original plan was a pile of self-hosted apps (Tailscale, Roon, Unifi Network,
and the two below). In practice it has settled down to a much smaller set:

- **Jellyfin** — movies and TV
- **Immich** — photos

Both run as containers under TrueNAS Apps, installed onto the `apps` pool that
lives on the spare space of the OS SSD (see the install notes).

Beyond the two apps, the box is a backup target over SMB:

- **Time Machine** for the Macs on the LAN
- **Proxmox Backup** (vzdump) for the Proxmox host

Roon, Unifi Network and Tailscale are no longer running here.

## Networking

The two onboard 1GbE NICs and the LACP link aggregation that used to bond them
have been retired. Networking is now a single 2.5GbE Intel I226-V card in the
PCIe slot, connected to a 2.5GbE switch port.

- [TrueNAS installation and configuration notes](https://github.com/MarkRWatts/MicroServer/blob/main/TrueNAS/README.md)
