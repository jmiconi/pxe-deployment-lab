# Architecture

## Components

- **DHCP** provides client network configuration and boot information.
- **TFTP** serves the small firmware-compatible bootstrap binary.
- **iPXE** provides a richer boot environment and HTTP support.
- **HTTP server** serves menus, wimboot and WinPE boot assets.
- **WinPE** provides the Windows-side deployment environment.
- **SMB image source** exposes the Windows installation image to WinPE without loading the full WIM into memory.
- **DISM** applies the selected Windows image to the target volume.
- **bcdboot** creates the UEFI boot files required for local startup.

## Validated data flow

```text
UEFI Client
   |
   | DHCP discovery
   v
DHCP Service
   |
   | boot filename / next server
   v
TFTP Service
   |
   | snponly.efi
   v
iPXE
   |
   | menu + WinPE assets over HTTP
   v
HTTP Service
   |
   | wimboot + BCD + boot.sdi + boot.wim
   v
WinPE
   |
   | read-only SMB access
   v
Windows Image Source
   |
   | install.wim
   v
DiskPart + DISM + bcdboot
   |
   v
Local UEFI boot
   |
   v
Windows OOBE
```

## Design choices

The bootstrap remains small and firmware-friendly. Once iPXE is running, larger boot downloads move to HTTP to reduce transfer time and simplify diagnostics.

The Windows installation image remains outside the repository and was exposed to WinPE through a read-only SMB share. This avoids loading a multi-gigabyte `install.wim` into WinPE memory and keeps boot assets separate from operating-system installation media.

The validation environment used an isolated VMware standard switch without a physical uplink. DHCP was bound only to the PXE lab interface, allowing the full workflow to be tested without interacting with an existing production DHCP/PXE environment.
