# Architecture

## Components

- **DHCP** provides client network configuration and boot information.
- **TFTP** serves the small firmware-compatible bootstrap binary.
- **iPXE** provides a richer boot environment and HTTP support.
- **HTTP server** serves menus, WinPE images and deployment assets.
- **WinPE** provides the Windows-side deployment environment.

## Data flow

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
   | snponly.efi or equivalent
   v
iPXE
   |
   | menu + assets over HTTP
   v
HTTP Service
   |
   v
WinPE
```

## Design choice

The bootstrap remains small and firmware-friendly. Once iPXE is running, larger downloads move to HTTP to reduce transfer time and simplify diagnostics.
