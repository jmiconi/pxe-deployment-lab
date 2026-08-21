# Troubleshooting PXE by layer

Treat the boot path as a sequence of independent dependencies.

## 1. Firmware

- Confirm UEFI network boot is enabled.
- Confirm the NIC is visible to firmware.
- Prefer a temporary boot menu over permanently changing boot order during testing.
- Disable Secure Boot during initial iPXE lab validation if firmware refuses the bootstrap image.

## 2. DHCP

- Verify the client receives an address, gateway and DNS configuration as appropriate for the lab.
- Confirm PXE boot information is appropriate for the client architecture.
- In an isolated iPXE lab, distinguish firmware PXE from iPXE itself to avoid a boot loop.

Example ISC DHCP logic:

```text
next-server 10.10.50.10;

if exists user-class and option user-class = "iPXE" {
    filename "http://10.10.50.10/pxe/boot/menu.ipxe";
} else {
    filename "snponly.efi";
}
```

Without this distinction, iPXE may request DHCP and be handed `snponly.efi` again indefinitely.

## 3. Routing and firewall

Test reachability between the deployment network and the PXE server. Routed environments may require DHCP relay/helper configuration in addition to normal IP routing.

For a safe validation environment, an isolated virtual switch with no physical uplink avoids interfering with production DHCP or PXE services.

## 4. TFTP

Validate the bootstrap file independently before troubleshooting iPXE menus.

TFTP uses UDP/69 only for the initial request. The server then transfers data from a dynamic UDP port. When troubleshooting, a capture limited to port 69 may therefore look incomplete.

Useful capture:

```bash
sudo tcpdump -ni <pxe-interface> udp
```

A healthy transfer should show an RRQ for the bootstrap followed by server data and client acknowledgements.

## 5. iPXE

Use the interactive shell to inspect addressing and test HTTP retrieval:

```text
ifstat
route
imgfetch http://192.0.2.10/pxe/boot/menu.ipxe
```

The repository menu uses `${next-server}` for the HTTP base URL so the same file can be reused across lab addresses.

## 6. HTTP

Confirm the same URL can be retrieved from another host on the deployment network. HTTP server logs are often the fastest evidence for missing paths or access issues.

For WinPE boot via `wimboot`, validate all expected requests:

```text
/pxe/wimboot
/pxe/winpe/BCD
/pxe/winpe/boot.sdi
/pxe/winpe/boot.wim
```

## 7. WinPE

Once WinPE loads, troubleshoot storage, network drivers and deployment scripts separately from PXE itself.

If WinPE boots but `ipconfig` shows no usable network configuration, initialize networking explicitly:

```text
wpeutil InitializeNetwork
ipconfig
ping <pxe-server>
```

This was required during clean lab validation even though iPXE networking had already worked correctly. iPXE network success does not prove that WinPE has initialized its own network stack.
