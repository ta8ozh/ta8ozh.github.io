---
layout: default
title: "AirPrint Server on Proxmox LXC"
date: 2026-11-22 00:00:00 +0000
---


Use a Debian/Ubuntu LXC container to share an IPP network printer with iPhone, iPad, or Mac through **CUPS + Avahi**.

**Scope:** Printing only. No scanner setup. All addresses and names below are examples, not real device details.

## 1. Before you start

- The LXC, printer, and iPhone must be on a network where Bonjour/mDNS discovery works (usually the same LAN/VLAN).
- The printer must support IPP. TCP **631** must be reachable between the LXC and printer, and from the iPhone to the LXC. Bonjour uses UDP **5353**.
- Run commands **inside the LXC as root**, unless noted otherwise.
- An IPP network printer does **not** need privileged LXC or USB passthrough.

Set your own values first (replace the example IPs):

```bash
PRINTER_IP="192.0.2.21"   # REPLACE with your printer's IP
LXC_IP="192.0.2.54"       # REPLACE with your LXC's IP
QUEUE="AirPrint_Queue"
PRINTER_URI="ipp://${PRINTER_IP}/ipp/print"  # Verify this path; it varies by printer
```

> `192.0.2.x` addresses are documentation examples and will not work on your LAN.

## 2. Install the services

```bash
apt update
apt install -y cups cups-filters cups-client avahi-daemon avahi-utils netcat-openbsd
systemctl enable --now cups avahi-daemon
```

Check:

```bash
systemctl is-active cups avahi-daemon
```

Both should show `active`.

## 3. Test the printer connection

```bash
nc -zv -w 3 "$PRINTER_IP" 631
ipptool -tv "$PRINTER_URI" /usr/share/cups/ipptool/get-printer-attributes.test
```

Look for `open` and `PASS`. If the second command fails, check the actual IPP path with the printer manual or:

```bash
avahi-browse -rt _ipp._tcp
```

Find the **physical printer's** `rp=` value; do not assume every model uses `/ipp/print`.

## 4. Add and share the printer

```bash
lpadmin -p "$QUEUE" -E -v "$PRINTER_URI" -m everywhere
lpadmin -p "$QUEUE" -o printer-is-shared=true
cupsctl --share-printers
```

`-m everywhere` works for compatible driverless IPP printers; some printers require another driver.

Check:

```bash
lpstat -v
lpstat -p "$QUEUE"
```

The queue should point directly to your printer (`ipp://...`), not to an `implicitclass://...` URI.

## 5. Print a test page

```bash
lp -d "$QUEUE" /usr/share/cups/data/testprint
```

**Check that paper actually comes out.** A job ID alone is not proof of successful printing.

## 6. Test AirPrint from iPhone

Check that Avahi advertises the shared CUPS queue:

```bash
avahi-browse -rt _ipp._tcp
```

Look for the CUPS queue, typically `AirPrint_Queue @ server-name`, using the **LXC IP**, TCP port **631**, and `rp=printers/AirPrint_Queue`.

On the iPhone: **Open a photo/document → Share → Print → Select Printer**. Choose the CUPS queue and print.

**Done:** If the page prints, the basic setup is complete. You do not normally need a custom Avahi service file.

---

## Troubleshooting: only use the matching fix

### Printer cannot be reached

Check that `PRINTER_IP` is current, and the printer is connected to the network:

```bash
nc -zv -w 3 "$PRINTER_IP" 631
```

If it times out, investigate Wi-Fi, bridge/VLAN or firewall connectivity **before** changing CUPS drivers.

### `Connection reset by peer` or slow CUPS web interface

```bash
systemctl status cups --no-pager -l
cupsd -t
ss -lntp | grep ':631'
journalctl -u cups -n 30 --no-pager
```

CUPS must be running and listening on TCP 631. `cupsd -t` checks configuration syntax, not whether the service is healthy.

### A duplicate `implicitclass://` queue appears

This is often created by `cups-browsed`. If you only need the manually configured direct IPP queue, you can disable automatic queue creation:

```bash
systemctl disable --now cups-browsed
```

Do this **only after** your direct queue prints successfully. Keep Avahi running. Re-enable later with `systemctl enable --now cups-browsed` if needed.

### Printing takes about 50 seconds in a Proxmox LXC

In one tested LXC setup, CUPS waited **twice for 25 seconds** because the `colord` service could not start. **Apply this fix only if your logs match**; not every slow print job has this cause.

Check:

```bash
systemctl status colord --no-pager -l
systemctl status dbus --no-pager -l
```

Look for `org.freedesktop.ColorManager` timeouts and `colord` errors including **`226/NAMESPACE`** or **`Failed to set up mount namespacing: Permission denied`**.

If those errors appear and `systemctl cat colord.service` shows `PrivateTmp=yes`, use a service-only override:

```bash
mkdir -p /etc/systemd/system/colord.service.d
printf '[Service]\nPrivateTmp=no\n' > /etc/systemd/system/colord.service.d/lxc.conf
systemctl daemon-reload
systemctl reset-failed colord.service
systemctl start colord.service
systemctl is-active colord.service
```

Expected: `active`. The observed print delay improved from approximately **50 seconds to 5 seconds**. This fix persists across restarts.

### iPhone cannot find the queue

Verify the queue is shared, Avahi is active, and mDNS is not blocked by guest Wi-Fi, client isolation, or VLAN boundaries:

```bash
systemctl is-active avahi-daemon
grep -A 20 "<Printer ${QUEUE}>" /etc/cups/printers.conf
avahi-browse -rt _ipp._tcp
```

Look for `Shared Yes` and an advertisement for the **CUPS queue**, not only the physical printer.

---

## Final check

- [ ] A test page prints from the LXC.
- [ ] iPhone discovers the CUPS queue and can print.
- [ ] CUPS and Avahi start automatically.
- [ ] Printer and LXC IP addresses are stable (DHCP reservations recommended).
- [ ] Optional: take a Proxmox LXC backup.

```bash
systemctl is-enabled cups avahi-daemon
```

Both should normally show `enabled`.
