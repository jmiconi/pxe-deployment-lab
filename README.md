# PXE Deployment Lab

A practical lab for **network-based Windows deployment** using PXE, iPXE, TFTP, HTTP and WinPE.

The goal is to provision or recover workstations **without removable media**, while keeping the deployment flow observable and easy to troubleshoot.

## What this project demonstrates

- PXE boot fundamentals
- UEFI network boot
- TFTP bootstrap with iPXE
- HTTP delivery for larger boot assets
- WinPE-based Windows deployment
- Linux service deployment and hardening
- Network-path troubleshooting across routed subnets

## Architecture

```text
Client workstation
       |
       | PXE / DHCP
       v
+----------------+
| TFTP bootstrap |
+-------+--------+
        |
        | iPXE EFI
        v
+----------------+
|   iPXE menu    |
+-------+--------+
        |
        | HTTP
        v
+----------------+
| WinPE / tools  |
+-------+--------+
        |
        v
 Windows deployment
```

## Why TFTP + HTTP

TFTP is useful for the small initial network bootstrap, but HTTP is faster and more reliable for larger assets. iPXE bridges both stages cleanly.

## Repository structure

```text
pxe-deployment-lab/
├── boot/
│   └── menu.ipxe
├── docs/
│   ├── architecture.md
│   └── troubleshooting.md
└── README.md
```

## Example iPXE menu

See [`boot/menu.ipxe`](boot/menu.ipxe) for a sanitized starter menu that chains boot assets over HTTP.

## Operational flow

1. Client firmware starts a network boot.
2. DHCP provides network configuration and the PXE bootstrap path.
3. TFTP loads an iPXE binary.
4. iPXE retrieves the menu and larger assets over HTTP.
5. WinPE starts and performs the deployment or recovery workflow.

## Troubleshooting model

PXE failures are easier to isolate when treated as separate layers:

`Firmware → DHCP → Routing → TFTP → iPXE → HTTP → WinPE`

The repository documents checks for each layer instead of treating PXE as one opaque service.

## Security notes

- Example addresses use documentation-only ranges.
- No production DHCP, DNS or internal infrastructure data is included.
- Network boot services should be restricted to authorized deployment networks.

## Roadmap

- Automated WinPE build
- Driver injection workflow
- Unattended Windows setup examples
- Post-install PowerShell automation
- Linux rescue/bootstrap option
