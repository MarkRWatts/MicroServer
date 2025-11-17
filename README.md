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

- [TrueNAS installation instructions](https://github.com/MarkRWatts/MicroServer/TrueNAS/README.md)
