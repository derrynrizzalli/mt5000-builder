# mt5000-builder

Custom OpenWrt firmware build for the GL.iNet GL-MT5000 (Brume 3), built via
GitHub Actions from [dmsza/openwrt](https://github.com/dmsza/openwrt) branch
`openwrt-main-mt5000`, pinned at `102a02f03e` (2026-09-27).

That tree is current OpenWrt main plus GL-MT5000 support from
[openwrt/openwrt#24237](https://github.com/openwrt/openwrt/pull/24237),
including the RTL8366UB DSA switch. It replaces the February 2026
`nathanli1211/openwrt_mt5000` fork this workflow used to compile.

Run the build manually from the Actions tab -> "Build OpenWrt for GL.iNet
GL-MT5000 (Brume 3)" -> Run workflow.

## What's baked in

- LuCI web UI, unbound (recursive DNS, with cache persisted across
  reboots/power loss), Prometheus node exporter + unbound stats exporter,
  vnstat2, full firewall/NAT stack, cron.
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
  - You don't need to give the whole USB drive to boot files - partition it
    (`fdisk`/`parted`, both included) and only mount a small ext4 partition
    at `/mnt/pxe`. The rest is free for other uses.
- **Samba (luci-app-samba4)**: SMB/CIFS file sharing for any USB storage
  space not used for PXE - configure shares via LuCI (Services > Network
  Shares) after flashing.
- **USB 3**: this source already connects the SuperSpeed PHY
  (`mediatek,u3p-dis-msk = <0>` and both USB2 and USB3 phys on `ssusb`).
  The old devicetree append, which targeted the February fork, is not
  applied.

## Self-hosted package repo

Each build also publishes a matching opkg/apk package repo to GitHub Pages
at derrynrizzalli.github.io/mt5000-builder, baked into the firmware's
`/etc/apk/repositories.d/distfeeds.list` so `opkg`/`apk` on the router
always points at packages that match the exact build it's running.

## Upgrading from the February image

Do not flash this image with **Keep settings** ticked. The February
`nathanli1211/openwrt_mt5000` tree and this one are not the same network
layout. A kept backup would be written back: this pin does include
`glinet,gl-mt5000` in `platform_copy_config()`, and that calls
`emmc_copy_config`, so eMMC sysupgrade no longer silently drops a saved
config. The old config is the problem. It is not a compatible config.

The live setup that has to survive is the UCI you have now: the uplink
named `dns` (DHCP, 10.10.1.2 on device `eth1`), firewall, dhcp (including
PXE/TFTP), unbound, the unbound cache dump, Prometheus
(`listen_interface='*'` on port 9100), Samba shares, mount points,
dropbear host keys, and root's crontab. A fresh flash recreates the
stock extras (unbound-cache init, metrics CGI, Prometheus uci-defaults,
TFTP root `/mnt/pxe`, distfeeds). It does not recreate that live UCI.

### Port map

The February board script is `ucidef_set_interfaces_lan_wan eth0 eth1`.
This tree is `ucidef_set_interfaces_lan_wan "lan1 lan2" eth1`.

| February image | This image |
| --- | --- |
| `eth0` LAN (both LAN jacks as one interface, 192.168.1.1) | DSA ports `lan1` and `lan2`, bridged as `br-lan`. Do not put the LAN address on `eth0`. On this image `eth0` is gmac0, the switch CPU port. |
| `eth1` uplink, UCI interface name `dns`, DHCP from upstream (10.10.1.2) | Same PHY: `eth1` is gmac1. A fresh flash calls that interface `wan`. Rename it to `dns` so restored firewall zones still match. |
| swconfig `switch0`, if present | Not used. The RTL8366UB is a DSA switch. |

There is no automatic translation of an arbitrary old `network` file.
Back it up, do not restore it, and do the one rename below.

### 1. Back up on the live router, before the flash

Paste this over SSH to `10.10.1.2` (the uplink that is up today).

```sh
cat > /tmp/mt5000-backup-live << 'EOF'
#!/bin/sh
# Backup the live GL-MT5000 setup before a cross-tree flash.
# Copy the tarball off the router before you flash. /tmp is wiped.
OUT="${1:-/tmp/mt5000-backup.tar.gz}"
STAGE=$(mktemp -d) || exit 1
trap 'rm -rf "$STAGE"' EXIT INT TERM

sysupgrade -b "$STAGE/sysupgrade.tar.gz" || exit 1
sysupgrade -l > "$STAGE/sysupgrade-list.txt" || true

copy_one() {
	src="$1"
	[ -e "$src" ] || return 0
	dest="$STAGE/files$src"
	mkdir -p "$(dirname "$dest")"
	cp -a "$src" "$dest"
}

copy_tree() {
	src="$1"
	[ -d "$src" ] || return 0
	mkdir -p "$STAGE/files$src"
	cp -a "$src/." "$STAGE/files$src/"
}

copy_tree /etc/config
copy_tree /etc/crontabs
copy_tree /etc/dropbear
copy_tree /etc/unbound
copy_one /etc/sysupgrade.conf
copy_one /etc/rc.local
copy_one /www/cgi-bin/metrics.cgi
copy_one /etc/init.d/unbound-cache

tar -czf "$OUT" -C "$STAGE" . || exit 1
echo "Wrote $OUT"
echo "Copy it off the router before flashing:"
echo "  scp root@10.10.1.2:$OUT ."
if [ -d /mnt/pxe ]; then
	cp -f "$OUT" /mnt/pxe/mt5000-backup.tar.gz && echo "Also copied to /mnt/pxe/mt5000-backup.tar.gz"
fi
EOF
chmod +x /tmp/mt5000-backup-live
/tmp/mt5000-backup-live /tmp/mt5000-backup.tar.gz
```

Copy the tarball off the box before you flash. `/tmp` does not survive
sysupgrade. The USB stick does, if it is mounted.

```sh
scp root@10.10.1.2:/tmp/mt5000-backup.tar.gz .
```

### 2. Flash without keeping settings

Use the sysupgrade image from the build artifacts. In LuCI, leave
**Keep settings** unchecked. From SSH:

```sh
sysupgrade -n /tmp/openwrt-mediatek-filogic-glinet_gl-mt5000-squashfs-sysupgrade.bin
```

`-n` means do not restore the config. After it boots, a computer on
either LAN port should reach `192.168.1.1`. The WAN jack (`eth1`) will
DHCP as interface `wan`. Do the rest of this from the LAN port, not
from 10.10.1.2, because the firewall zone is about to change name.

The router has not been flashed from here.

### 3. Rename the uplink, then restore everything except network

```sh
uci rename network.wan=dns
uci commit network
/etc/init.d/network reload
```

`eth1` should DHCP again (the upstream lease that has been 10.10.1.2).
`br-lan` stays `192.168.1.1` on `lan1` and `lan2`.

Copy the tarball back and paste this restore script. It is not installed
in the image.

```sh
cat > /tmp/mt5000-restore-live << 'EOF'
#!/bin/sh
# Restore a mt5000-backup-live tarball, or a raw `sysupgrade -b` archive.
# Does not apply /etc/config/network, distfeeds, or the old vnstat config.
TAR="$1"
[ -n "$TAR" ] && [ -f "$TAR" ] || {
	echo "usage: mt5000-restore-live /tmp/mt5000-backup.tar.gz" >&2
	exit 1
}
STAGE=$(mktemp -d) || exit 1
trap 'rm -rf "$STAGE"' EXIT INT TERM
tar -xzf "$TAR" -C "$STAGE" || exit 1

if [ -d "$STAGE/files/etc" ]; then
	SRC="$STAGE/files"
elif [ -d "$STAGE/etc" ]; then
	SRC="$STAGE"
elif [ -f "$STAGE/sysupgrade.tar.gz" ]; then
	mkdir -p "$STAGE/nested"
	tar -xzf "$STAGE/sysupgrade.tar.gz" -C "$STAGE/nested" || exit 1
	SRC="$STAGE/nested"
else
	echo "unrecognized backup layout" >&2
	find "$STAGE" -maxdepth 3 -type d >&2
	exit 1
fi

install_file() {
	rel="$1"
	[ -e "$SRC/$rel" ] || return 0
	mkdir -p "$(dirname "/$rel")"
	cp -a "$SRC/$rel" "/$rel"
	echo "restored /$rel"
}

if [ -d "$SRC/etc/config" ]; then
	for f in "$SRC/etc/config"/*; do
		[ -e "$f" ] || continue
		base=$(basename "$f")
		case "$base" in
			network|vnstat)
				echo "skipped /etc/config/$base"
				;;
			*)
				install_file "etc/config/$base"
				;;
		esac
	done
fi

if [ -f "$SRC/etc/config/network" ]; then
	cp -a "$SRC/etc/config/network" /etc/config/network.from-backup
	echo "saved backed-up network as /etc/config/network.from-backup (not applied)"
fi

for rel in etc/crontabs etc/dropbear etc/unbound; do
	[ -d "$SRC/$rel" ] || continue
	mkdir -p "/$rel"
	cp -a "$SRC/$rel/." "/$rel/"
	echo "restored /$rel/"
done

install_file www/cgi-bin/metrics.cgi
[ -f /www/cgi-bin/metrics.cgi ] && chmod +x /www/cgi-bin/metrics.cgi
install_file etc/init.d/unbound-cache
[ -f /etc/init.d/unbound-cache ] && chmod +x /etc/init.d/unbound-cache

if ! grep -q '/etc/unbound/' /etc/sysupgrade.conf 2>/dev/null \
	&& ! grep -q '/etc/unbound/' /lib/upgrade/keep.d/* 2>/dev/null; then
	mkdir -p /etc
	touch /etc/sysupgrade.conf
	cat >> /etc/sysupgrade.conf << 'KEEP'
/etc/unbound/
/etc/init.d/unbound-cache
/www/cgi-bin/metrics.cgi
KEEP
	echo "added unbound cache and metrics.cgi to /etc/sysupgrade.conf"
fi

for svc in firewall dnsmasq unbound cron samba4; do
	[ -x "/etc/init.d/$svc" ] && /etc/init.d/"$svc" restart
done
if [ -x /etc/init.d/unbound-cache ]; then
	/etc/init.d/unbound-cache enable
	/etc/init.d/unbound-cache start
fi

echo
echo "Network was not restored."
echo "If the uplink interface is still named wan, rename it before you rely on firewall zones that say dns:"
echo "  uci rename network.wan=dns"
echo "  uci commit network"
echo "  /etc/init.d/network reload"
echo "  /etc/init.d/firewall restart"
EOF
chmod +x /tmp/mt5000-restore-live
/tmp/mt5000-restore-live /tmp/mt5000-backup.tar.gz
```

That restores `/etc/config` except `network` and the old `vnstat` file
(the package on this tree is `vnstat2`), plus `/etc/crontabs`,
`/etc/dropbear`, `/etc/unbound` (including `cache.dump` if it was
there), and `/www/cgi-bin/metrics.cgi`. It writes the old network file
to `/etc/config/network.from-backup` and does not apply it. It does
not restore `/etc/apk/repositories.d/distfeeds.list`. The image's own
copy must stay, or `apk` points at the wrong build.

If `/etc/config/network.from-backup` shows a LAN address other than
`192.168.1.1`, set only that address. Do not copy `option device` or
`list ports` from that file.

```sh
uci set network.lan.ipaddr='<address from network.from-backup>'
uci set network.lan.netmask='<netmask from network.from-backup>'
uci commit network
/etc/init.d/network reload
```

### 4. Check

```sh
ip addr show br-lan
ip addr show eth1
uci get network.dns.proto
uci get prometheus-node-exporter-lua.main.listen_interface
uci get dhcp.@dnsmasq[0].enable_tftp
uci get dhcp.@dnsmasq[0].tftp_root
```

Expect `192.168.1.1` on `br-lan`, a DHCP address on `eth1`, `dns` proto
`dhcp`, Prometheus listen interface `*`, TFTP enabled with root
`/mnt/pxe`. Re-set the TFTP lines if the restored dhcp config did not
have them:

```sh
uci set dhcp.@dnsmasq[0].enable_tftp='1'
uci set dhcp.@dnsmasq[0].tftp_root='/mnt/pxe'
uci commit dhcp
/etc/init.d/dnsmasq restart
```

Plug the USB drive back and confirm the `/mnt/pxe` mount. Samba shares
come back from `/etc/config/samba4`.

### What is safe to keep

Safe to restore from the tarball: firewall, dhcp, unbound, samba4,
fstab, prometheus-node-exporter-lua, dropbear, crontabs, `/etc/unbound`,
the metrics CGI, `rc.local`.

Not safe to restore as-is: `/etc/config/network`, any swconfig
`switch` section, `/etc/apk/repositories.d/distfeeds.list`,
`/etc/config/vnstat` (old package name).

### If Keep settings was ticked anyway

This image does not rewrite that file. Keep settings puts the February
network back, `br-lan` stays on `eth0`, and the LAN jacks can come up
with no address. Do not tick it. The downloaded tarball is the copy you
restore by hand.

### Later upgrades, once you are on this tree

Keep settings is appropriate for a later sysupgrade of this same
image. `platform_copy_config()` writes the backup back to eMMC.
The restore script appends `/etc/unbound/`, `/etc/init.d/unbound-cache`,
and `/www/cgi-bin/metrics.cgi` to `/etc/sysupgrade.conf`, which a
default sysupgrade list can miss. `distfeeds.list` stays excluded on
purpose.
