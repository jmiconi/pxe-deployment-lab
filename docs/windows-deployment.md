# Windows deployment from WinPE

This document records the manual deployment path validated in the isolated PXE lab.

It assumes the client has already booted into WinPE through the tested PXE/iPXE/wimboot chain.

> **Warning:** the disk commands below are destructive. Identify the intended disposable target disk before using `clean`.

## 1. Initialize WinPE networking

If WinPE does not immediately show an IPv4 configuration:

```text
wpeutil InitializeNetwork
ipconfig
```

Confirm that the PXE/deployment server is reachable before continuing.

## 2. Map the Windows image source

The validated lab exposed the Windows image through a read-only SMB share.

```text
net use Z: \\<pxe-server>\windows
dir Z:\sources
```

Confirm that `install.wim` is visible.

The tested media contained Windows 11 Pro at WIM index 6. Always inspect your own image metadata and select the correct index for your authorized installation media.

## 3. Identify the target disk

```text
diskpart
list disk
list volume
```

Do not assume that the target is Disk 0 in another environment.

The validated test VM contained one disposable 60 GB GPT disk.

## 4. Create a clean UEFI/GPT layout

After identifying the correct target disk, the tested layout was created with:

```text
select disk 0
clean
convert gpt

create partition efi size=260
format quick fs=fat32 label="System"
assign letter=S

create partition msr size=16

create partition primary
format quick fs=ntfs label="Windows"
assign letter=W

list volume
exit
```

Resulting layout:

```text
EFI System Partition   260 MB FAT32   S:
Microsoft Reserved      16 MB
Windows                  remainder   W:
```

## 5. Apply Windows

The validated Windows 11 Pro image was applied with:

```text
dism /Apply-Image /ImageFile:Z:\sources\install.wim /Index:6 /ApplyDir:W:\
```

Wait for DISM to reach 100% and report a successful operation before continuing.

## 6. Create UEFI boot files

```text
bcdboot W:\Windows /s S: /f UEFI
```

The tested run successfully created the boot files.

## 7. Reboot to local disk

Exit WinPE or restart the VM and boot from the local virtual disk instead of network boot.

The validated test client progressed through Windows preparation and reached Windows 11 Pro OOBE.

## What this proves

The validated path is:

```text
WinPE
-> network initialization
-> SMB image source
-> GPT partitioning
-> DISM image application
-> bcdboot
-> local-disk boot
-> Windows OOBE
```

This is a **manual** deployment procedure. It is intentionally separate from future zero-touch automation so the repository does not claim unattended behavior that has not yet been tested.
