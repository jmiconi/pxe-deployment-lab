# Troubleshooting PXE by layer

Treat the boot path as a sequence of independent dependencies.

## 1. Firmware

- Confirm UEFI network boot is enabled.
- Confirm the NIC is visible to firmware.
- Prefer a temporary boot menu over permanently changing boot order during testing.

## 2. DHCP

- Verify the client receives an address, gateway and DNS configuration.
- Confirm PXE boot information is appropriate for the client architecture.

## 3. Routing and firewall

Test reachability between the deployment network and the PXE server. Routed environments may require DHCP relay/helper configuration in addition to normal IP routing.

## 4. TFTP

Validate the bootstrap file independently before troubleshooting iPXE menus.

## 5. iPXE

Use the interactive shell to inspect addressing and test HTTP retrieval:

```text
ifstat
route
imgfetch http://192.0.2.10/pxe/boot/menu.ipxe
```

## 6. HTTP

Confirm the same URL can be retrieved from another host on the deployment network. HTTP server logs are often the fastest evidence for missing paths or access issues.

## 7. WinPE

Once WinPE loads, troubleshoot storage, network drivers and deployment scripts separately from PXE itself.
