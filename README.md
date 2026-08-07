# mt5000-builder

Custom OpenWrt firmware build for the GL.iNet GL-MT5000 (Brume 3), built via
GitHub Actions from the [nathanli1211/openwrt_mt5000](https://github.com/nathanli1211/openwrt_mt5000)
fork (`glinet-mt5000` branch, OpenWrt 25.12). Run the build manually from the
Actions tab -> "Build OpenWrt for GL.iNet GL-MT5000 (Brume 3)" -> Run workflow.

## What's baked in

- LuCI web UI, unbound (recursive DNS, with cache persisted across
  reboots/power loss), Prometheus node exporter + unbound stats exporter,
  vnstat, full firewall/NAT stack, cron.
- **Network boot (PXE) support**: dnsmasq-full (DHCP + TFTP server), USB3/
  USB-storage drivers, ext4/vfat/exFAT filesystem support + mkfs/fsck tools,
  and an NFS server (for diskless-client root filesystems). TFTP is enabled
  by default, rooted at `/mnt/pxe`.
  - After flashing, plug in a USB drive and set its mount point to
    `/mnt/pxe` in LuCI (System > Mount Points), then drop boot files there
    (`pxelinux.0` / `grubnetx64.efi` / `ipxe.efi` / kernels / images -
    these aren't produced by this build and need to be sourced separately).
  - The DHCP "boot filename" is left unset in the firmware since it depends
    on what you're booting - set it via LuCI (Network > DHCP and DNS) or
    `uci set dhcp.@dnsmasq[0].dhcp_boot='<file>'` once decided.

## Self-hosted package repo

Each build also publishes a matching opkg/apk package repo to GitHub Pages
at derrynrizzalli.github.io/mt5000-builder, baked into the firmware's
`/etc/apk/repositories.d/distfeeds.list` so `opkg`/`apk` on the router
always points at packages that match the exact build it's running.