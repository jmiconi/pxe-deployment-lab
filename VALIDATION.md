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
- Samba installed for the Windows image source

The validation used an isolated VMware standard switch with no physical uplink. The PXE server had a dedicated lab interface and the test client was attached only to that isolated network, preventing interaction with production DHCP or PXE services.

## Final validated flow

The following chain was tested end-to-end through first Windows startup:

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
  -> Windows PE
  -> WinPE network initialization
  -> SMB read-only install.wim
  -> GPT disk partitioning
  -> DISM /Apply-Image
  -> bcdboot UEFI files
  -> local-disk boot
  -> Windows 11 Pro OOBE
```

## Server bootstrap findings

A clean Ubuntu 24.04.4 host did not include nginx, TFTP or iPXE by default. The following packages were installed during validation:

```bash
sudo apt install -y nginx tftpd-hpa ipxe ipxe-qemu tftp-hpa isc-dhcp-server samba
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

## Windows image source validation

The Windows installation source was a Windows 11 `install.wim` containing Windows 11 Pro at index 6.

The image was stored outside the repository and exposed to WinPE using a read-only SMB share. From WinPE:

```text
net use Z: \\<pxe-server>\windows
dir Z:\sources
```

completed successfully and exposed the multi-gigabyte `install.wim` without copying the image into WinPE memory.

## Disk and image deployment validation

The test VM contained a single disposable 60 GB GPT disk. The target disk was identified before destructive operations.

A clean UEFI layout was created with:

```text
EFI System Partition   260 MB FAT32
Microsoft Reserved     16 MB
Windows                 remaining NTFS space
```

Windows 11 Pro was then applied using WIM index 6:

```text
dism /Apply-Image /ImageFile:Z:\sources\install.wim /Index:6 /ApplyDir:W:\
```

The UEFI boot files were created with:

```text
bcdboot W:\Windows /s S: /f UEFI
```

The VM rebooted from its local disk, displayed the Windows preparation phase and successfully reached Windows 11 OOBE (country/region selection).

## Final status

### PASS

The full tested workflow completed successfully:

```text
UEFI
-> DHCP
-> TFTP
-> iPXE
-> HTTP
-> wimboot
-> WinPE
-> SMB
-> GPT partitioning
-> DISM image application
-> bcdboot
-> local Windows boot
-> Windows 11 Pro OOBE
```

This validation proves the manual deployment path. It does **not** yet claim unattended zero-touch deployment; automation of the WinPE disk/image commands remains a future iteration.

## Revalidation policy

Clean-lab revalidation should be performed after changes affecting:

- DHCP/iPXE chain logic
- TFTP bootstrap paths
- HTTP asset layout
- iPXE menu syntax
- wimboot or WinPE boot assets
- WinPE network initialization
- SMB image delivery
- disk partitioning logic
- DISM image-selection logic
- UEFI bootloader creation
- unattended deployment automation
