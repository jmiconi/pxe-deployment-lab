# Validation

## Environment

Validation was performed from a clean Ubuntu Server 24.04.4 LTS virtual-machine baseline in VMware.

Server-side components observed during the run:

- Ubuntu Server 24.04.4 LTS
- nginx 1.24.0
- tftpd-hpa 5.2
- ISC DHCP Server 4.4.3-P1
- iPXE package 1.21.1+git-20220113.fbbdc3926
- wimboot 2.9.0

The validation used an isolated VMware standard switch with no physical uplink. The PXE server had a dedicated lab interface and the test client was attached only to that isolated network, preventing interaction with production DHCP or PXE services.

## Validated flow

The following chain was tested end-to-end:

```text
UEFI client
  -> DHCP
  -> TFTP bootstrap
  -> snponly.efi
  -> iPXE
  -> second DHCP request identified as iPXE
  -> HTTP menu.ipxe
  -> wimboot
  -> BCD + boot.sdi + boot.wim
  -> Windows PE / Windows Setup
```

## Server bootstrap findings

A clean Ubuntu 24.04.4 host did not include nginx, TFTP or iPXE by default. The following packages were installed during validation:

```bash
sudo apt install -y nginx tftpd-hpa ipxe ipxe-qemu tftp-hpa isc-dhcp-server
```

Ubuntu provided the iPXE bootstrap binaries at:

```text
/usr/lib/ipxe/snponly.efi
/usr/lib/ipxe/undionly.kpxe
/usr/lib/ipxe/ipxe.efi
```

The default TFTP root created by `tftpd-hpa` was:

```text
/srv/tftp
```

TFTP and HTTP were validated independently before introducing DHCP.

## TFTP validation

A local TFTP test file transferred successfully. `snponly.efi` was then downloaded through TFTP and its SHA-256 hash matched the source file exactly.

A packet capture from the isolated PXE interface later confirmed the complete UEFI-client transfer: RRQ for `snponly.efi`, server data from a dynamic UDP port, and client acknowledgements through the final block.

This also confirmed an important troubleshooting detail: limiting packet capture to UDP/69 does not show the complete TFTP transfer because the data stream moves to a dynamic UDP port after the initial request.

## HTTP validation

nginx successfully served the iPXE menu from both localhost and another host.

The following WinPE assets returned HTTP 200:

```text
/pxe/wimboot
/pxe/winpe/BCD
/pxe/winpe/boot.sdi
/pxe/winpe/boot.wim
```

`wimboot` was not available as an Ubuntu package in the enabled repositories during this run. Version 2.9.0 was obtained from the upstream iPXE wimboot release and served over HTTP.

## Isolated DHCP validation

The PXE server used a dedicated isolated interface with a lab-only IPv4 subnet. ISC DHCP was explicitly bound to that interface.

The client successfully completed:

```text
DHCPDISCOVER
DHCPOFFER
DHCPREQUEST
DHCPACK
```

and received a lab address before requesting `snponly.efi` over TFTP.

## iPXE boot-loop finding

The first DHCP configuration always returned `snponly.efi`. This caused an iPXE loop: firmware loaded iPXE, iPXE performed DHCP again, and DHCP returned `snponly.efi` again.

The issue was resolved by detecting iPXE's DHCP user class and returning the HTTP menu on the second DHCP exchange:

```text
if exists user-class and option user-class = "iPXE" {
    filename "http://<pxe-server>/pxe/boot/menu.ipxe";
} else {
    filename "snponly.efi";
}
```

After this correction, the client displayed the `Network Deployment Lab` iPXE menu successfully.

## Portable menu finding

The original repository menu used the documentation-only address `192.0.2.10`. During validation it was changed to use the DHCP-provided next server:

```text
set base-url http://${next-server}/pxe
```

This makes the menu reusable across lab addresses without editing the script for each deployment.

## WinPE validation

A Windows 11 installation ISO was used for the boot assets. The tested `boot.wim` launched Windows Setup successfully through iPXE and wimboot.

The HTTP access log confirmed successful requests for wimboot, BCD, boot.sdi and boot.wim before Windows PE started.

WinPE initially had no usable network configuration. Running:

```text
wpeutil InitializeNetwork
```

initialized networking successfully; `ipconfig` then returned the DHCP configuration and the PXE server became reachable by ping. No virtual-NIC change or driver injection was required for this validation.

## Current scope

### PASS

The network-boot portion is validated end-to-end through Windows PE:

```text
UEFI -> DHCP -> TFTP -> iPXE -> HTTP -> wimboot -> WinPE
```

### In progress

Full operating-system deployment is the next stage. A Windows 11 ISO containing Windows 11 Pro at WIM index 6 was selected for the lab. The installation image is being exposed to WinPE so that disk partitioning, `DISM /Apply-Image` and `bcdboot` can be validated.

The repository must not claim full Windows installation automation until that stage has completed successfully.

## Revalidation policy

Clean-lab revalidation should be performed after changes affecting:

- DHCP/iPXE chain logic
- TFTP bootstrap paths
- HTTP asset layout
- iPXE menu syntax
- wimboot or WinPE boot assets
- WinPE network initialization
- unattended disk/image deployment logic
