# PXE Deployment Lab

A practical lab for **network-based Windows deployment** using PXE, iPXE, TFTP, HTTP, WinPE, SMB and DISM.

The goal is to provision or recover workstations **without removable media**, while keeping the deployment flow observable, reproducible and easy to troubleshoot.

> Current validation status: **PASS — Windows 11 Pro was deployed end-to-end from network boot through first OOBE startup.** See [`VALIDATION.md`](VALIDATION.md).

## What this project demonstrates

- PXE boot fundamentals
- UEFI network boot
- TFTP bootstrap with iPXE
- HTTP delivery for larger boot assets
- DHCP/iPXE chain handling without boot loops
- WinPE networking
- SMB delivery of a Windows installation image
- GPT/UEFI disk preparation
- Windows image application with DISM
- UEFI bootloader creation with `bcdboot`
- Linux service deployment and hardening
- Network-path troubleshooting across isolated or routed networks

## Validated architecture

```text
UEFI client
     |
     | DHCP
     v
+------------------+
| Lab DHCP service |
+--------+---------+
         |
         | next-server + snponly.efi
         v
+----------------+
| TFTP bootstrap |
+-------+--------+
        |
        | iPXE EFI
        v
+----------------+
|      iPXE      |
+-------+--------+
        |
        | HTTP menu + WinPE assets
        v
+----------------+
| nginx / HTTP   |
+-------+--------+
        |
        | wimboot + BCD + boot.sdi + boot.wim
        v
+----------------+
|     WinPE      |
+-------+--------+
        |
        | SMB read-only install.wim
        v
+----------------+
| Windows image  |
|     source     |
+-------+--------+
        |
        | diskpart + DISM + bcdboot
        v
+----------------+
| Windows 11 Pro |
|     OOBE       |
+----------------+
```

## Why TFTP + HTTP + SMB

TFTP is useful for the small initial firmware-compatible bootstrap, but HTTP is faster and easier to inspect for larger boot assets. iPXE bridges those stages cleanly.

For the multi-gigabyte Windows installation image, the validated lab exposed `install.wim` to WinPE through a read-only SMB share rather than loading the full image into WinPE memory.

## Repository structure

```text
pxe-deployment-lab/
├── boot/
│   └── menu.ipxe
├── docs/
│   ├── architecture.md
│   ├── troubleshooting.md
│   └── windows-deployment.md
├── VALIDATION.md
└── README.md
```

## Tested server baseline

The validation run used a clean **Ubuntu Server 24.04.4 LTS** virtual machine.

Packages required during the run:

```bash
sudo apt update
sudo apt install -y \
  nginx \
  tftpd-hpa \
  tftp-hpa \
  ipxe \
  ipxe-qemu \
  isc-dhcp-server \
  samba
```

Ubuntu provided the iPXE binaries at:

```text
/usr/lib/ipxe/snponly.efi
/usr/lib/ipxe/undionly.kpxe
/usr/lib/ipxe/ipxe.efi
```

The default TFTP root from `tftpd-hpa` was:

```text
/srv/tftp
```

## HTTP asset layout used by the lab

```text
/var/www/html/pxe/
├── boot/
│   └── menu.ipxe
├── wimboot
└── winpe/
    ├── BCD
    ├── boot.sdi
    └── boot.wim
```

The WinPE assets can be sourced from an authorized Windows installation or WinPE image. Installation media itself is not stored in this repository.

## Windows image source

The validated deployment exposed the Windows image through a read-only SMB share:

```text
/srv/windows/
└── sources/
    └── install.wim
```

Example Samba pattern:

```ini
[windows]
   path = /srv/windows
   browseable = yes
   read only = yes
   guest ok = yes
   force user = nobody
```

For real environments, replace guest access with authenticated read-only access and network restrictions appropriate to the deployment segment.

## Portable iPXE menu

The repository menu derives the HTTP server from DHCP rather than hard-coding a lab address:

```text
set base-url http://${next-server}/pxe
```

See [`boot/menu.ipxe`](boot/menu.ipxe).

## Avoiding the iPXE DHCP loop

Firmware PXE and iPXE both perform DHCP. If DHCP always returns `snponly.efi`, iPXE will load itself repeatedly.

A validated ISC DHCP pattern is:

```text
next-server <pxe-server>;

if exists user-class and option user-class = "iPXE" {
    filename "http://<pxe-server>/pxe/boot/menu.ipxe";
} else {
    filename "snponly.efi";
}
```

Use lab-specific addressing rather than copying example addresses directly.

## Safe isolation model

For validation, the PXE service was placed on an isolated VMware standard switch with **no physical uplink**. The PXE server used a dedicated second NIC and the lab DHCP service was bound only to that interface.

This allowed DHCP/PXE testing without interfering with an existing production deployment environment.

## Validated deployment flow

1. Client firmware starts UEFI network boot.
2. Lab DHCP provides an address, `next-server` and `snponly.efi`.
3. TFTP transfers the iPXE bootstrap.
4. iPXE performs DHCP again.
5. DHCP detects iPXE and returns the HTTP menu URL.
6. iPXE retrieves `menu.ipxe` over HTTP.
7. The WinPE option retrieves `wimboot`, `BCD`, `boot.sdi` and `boot.wim` over HTTP.
8. Windows PE starts.
9. WinPE networking is initialized when needed with:

```text
wpeutil InitializeNetwork
```

10. The read-only Windows image share is mapped from WinPE.
11. The target disk is recreated as GPT with EFI, MSR and Windows partitions.
12. Windows 11 Pro is applied with `DISM /Apply-Image`.
13. `bcdboot` creates the UEFI boot files.
14. The client reboots from local disk and reaches Windows OOBE.

The exact tested commands are documented in [`docs/windows-deployment.md`](docs/windows-deployment.md).

## Troubleshooting model

PXE failures are easier to isolate when treated as separate layers:

`Firmware → DHCP → TFTP → iPXE → HTTP → WinPE → SMB → DISM → UEFI boot`

See [`docs/troubleshooting.md`](docs/troubleshooting.md) for findings from the clean lab run, including TFTP dynamic UDP ports, iPXE DHCP loops and WinPE network initialization.

## Security notes

- Do not expose a lab DHCP server to a production broadcast domain.
- Restrict network boot services to authorized deployment networks.
- Do not publish Windows installation images, product keys or organization-specific deployment material in this repository.
- Use authenticated read-only SMB access outside an isolated disposable lab.
- Review destructive disk operations before adapting them to any real environment.
- Example infrastructure should remain sanitized and reproducible.

## Validation

The tested evidence and known scope are documented in [`VALIDATION.md`](VALIDATION.md).

Current status:

```text
PASS: UEFI -> DHCP -> TFTP -> iPXE -> HTTP -> wimboot -> WinPE
      -> SMB -> diskpart -> DISM -> bcdboot -> Windows 11 Pro OOBE
```

## Roadmap

- Automate WinPE deployment commands
- Automated WinPE build
- Driver injection workflow
- Unattended Windows setup examples
- Post-install PowerShell automation
- Linux rescue/bootstrap option
