# PXE Deployment Lab

A practical lab for **network-based Windows deployment** using PXE, iPXE, TFTP, HTTP and WinPE.

The goal is to provision or recover workstations **without removable media**, while keeping the deployment flow observable and easy to troubleshoot.

> Current validation status: the boot chain has been tested end-to-end through Windows PE. Full Windows image application is being validated separately and is not yet claimed as complete. See [`VALIDATION.md`](VALIDATION.md).

## What this project demonstrates

- PXE boot fundamentals
- UEFI network boot
- TFTP bootstrap with iPXE
- HTTP delivery for larger boot assets
- WinPE-based Windows deployment
- Linux service deployment and hardening
- Network-path troubleshooting across isolated or routed networks
- DHCP/iPXE chain handling without boot loops

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
        | HTTP menu + boot assets
        v
+----------------+
| nginx / HTTP   |
+-------+--------+
        |
        | wimboot + BCD + boot.sdi + boot.wim
        v
+----------------+
|     WinPE      |
+----------------+
```

## Why TFTP + HTTP

TFTP is useful for the small initial firmware-compatible bootstrap, but HTTP is faster and easier to inspect for larger assets. iPXE bridges both stages cleanly.

## Repository structure

```text
pxe-deployment-lab/
├── boot/
│   └── menu.ipxe
├── docs/
│   ├── architecture.md
│   └── troubleshooting.md
├── VALIDATION.md
└── README.md
```

## Tested server baseline

The current validation run used a clean **Ubuntu Server 24.04.4 LTS** virtual machine.

Packages required during the run:

```bash
sudo apt update
sudo apt install -y \
  nginx \
  tftpd-hpa \
  tftp-hpa \
  ipxe \
  ipxe-qemu \
  isc-dhcp-server
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

For validation, the PXE service can be placed on an isolated VMware standard switch with **no physical uplink**. Give the PXE server a dedicated second NIC and bind the lab DHCP service only to that interface.

This allows PXE/DHCP testing without interfering with an existing production DHCP or deployment environment.

## Operational flow

1. Client firmware starts UEFI network boot.
2. Lab DHCP provides an address, `next-server` and `snponly.efi`.
3. TFTP transfers the iPXE bootstrap.
4. iPXE performs DHCP again.
5. DHCP detects iPXE and returns the HTTP menu URL.
6. iPXE retrieves `menu.ipxe` over HTTP.
7. The WinPE option retrieves `wimboot`, `BCD`, `boot.sdi` and `boot.wim` over HTTP.
8. Windows PE starts.
9. If required, initialize WinPE networking with:

```text
wpeutil InitializeNetwork
```

Full disk partitioning and Windows image application are the next validation stage.

## Troubleshooting model

PXE failures are easier to isolate when treated as separate layers:

`Firmware → DHCP → Routing → TFTP → iPXE → HTTP → WinPE`

See [`docs/troubleshooting.md`](docs/troubleshooting.md) for findings from the clean lab run, including TFTP dynamic UDP ports, iPXE DHCP loops and WinPE network initialization.

## Security notes

- Do not expose a lab DHCP server to a production broadcast domain.
- Restrict network boot services to authorized deployment networks.
- Do not publish Windows installation images, product keys or organization-specific deployment material in this repository.
- Example infrastructure should remain sanitized and reproducible.

## Validation

The tested evidence and known scope are documented in [`VALIDATION.md`](VALIDATION.md).

Current status:

```text
PASS: UEFI -> DHCP -> TFTP -> iPXE -> HTTP -> wimboot -> WinPE
IN PROGRESS: disk partitioning -> DISM image apply -> bcdboot -> installed Windows
```

## Roadmap

- Complete Windows 11 image deployment validation
- Automated WinPE build
- Driver injection workflow
- Unattended Windows setup examples
- Post-install PowerShell automation
- Linux rescue/bootstrap option
