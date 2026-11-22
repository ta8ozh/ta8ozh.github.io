---
layout: default
title: "AirPrint Server on Proxmox LXC"
date: 2026-11-22 00:00:00 +0000
---

# AirPrint Server on Proxmox LXC

**A reusable setup and troubleshooting guide for sharing an IPP network printer with iPhone, iPad, and macOS using CUPS and Avahi.**

> **Scope:** Printing only. Scanning is not covered. This guide uses a generic IPP-capable printer and a Debian/Ubuntu-based Proxmox LXC container. It includes real troubleshooting patterns from a completed deployment, with all deployment-specific identifiers replaced by examples.

## 1. What you are building

```text
                      Local network / same mDNS discovery domain

iPhone / iPad / Mac
    |  Bonjour (mDNS, UDP 5353) discovers the shared printer
    |  IPP (TCP 631) sends print jobs
    v
Proxmox LXC container
    CUPS + Avahi
    Shared queue: AirPrint_Queue
    |  IPP (TCP 631)
    v
Network printer
    Printer-supported formats, e.g. PWG Raster
```

CUPS accepts print jobs and converts documents to a format supported by the physical printer. Avahi advertises the shared CUPS queue via Bonjour/mDNS. `cups-browsed` is **not required** for this manually managed queue.

### Requirements

- Proxmox VE with a Debian/Ubuntu LXC container (commands below run **inside the LXC as root**, unless stated otherwise).
- IPP-capable network printer reachable from the container.
- iPhone/iPad and container on a network where mDNS multicast can pass between them (typically the same LAN/VLAN). If routed between VLANs, an appropriate mDNS reflector/gateway may be required.
- TCP 631 from clients to CUPS, TCP 631 from LXC to printer (for IPP), and UDP 5353 for local mDNS discovery, subject to firewall rules.
- Prefer DHCP reservations for the container and printer.

**No privileged LXC or nesting setting is required solely to share an IPP network printer.**

### Examples and substitutions

The following addresses are **reserved documentation values**, not functioning LAN addresses:

| Placeholder | Example | Meaning |
|---|---|---|
| `<LXC_IP>` | `192.0.2.54` | CUPS server IP |
| `<PRINTER_IP>` | `192.0.2.21` | Printer IP |
| `<PRINTER_URI>` | `ipp://192.0.2.21/ipp/print` | Verified printer IPP endpoint |
| `<QUEUE>` | `AirPrint_Queue` | CUPS queue name |

In shell commands, use your own IPs. For repeatability, define variables in the LXC shell before running the examples:

```bash
PRINTER_IP='192.0.2.21'   # REPLACE with the printer's real IP
LXC_IP='192.0.2.54'       # REPLACE with the LXC's real IP
QUEUE='AirPrint_Queue'
PRINTER_URI="ipp://${PRINTER_IP}/ipp/print"  # Verify endpoint first
```

The placeholder values above will **not** route on a normal private LAN. The IPP path `/ipp/print` is common but **not universal**.

## 2. Check the LXC network

On the **Proxmox host**, locate your container and inspect its network configuration:

```bash
pct list
pct config <CTID>     # Replace <CTID> with the container ID
```

Typically, the LXC NIC is attached to a LAN bridge such as `vmbr0`; this is configuration-dependent. Inside the **LXC**, inspect addressing and routing:

```bash
ip -4 addr show
ip route
```

Verify the printer is reachable:

```bash
ping -c 3 "$PRINTER_IP"
ip neigh show "$PRINTER_IP"
```

Interpret `ip neigh` carefully: `REACHABLE` indicates recently verified neighbor connectivity; `STALE` means a cached entry (not necessarily failure); `FAILED` can indicate failed address resolution. An ICMP failure is not conclusive by itself because some printers block ping. Always test the print protocol directly in Step 4.

## 3. Install CUPS, Avahi and diagnostic tools

```bash
apt update
apt install -y cups cups-filters cups-client avahi-daemon avahi-utils netcat-openbsd
systemctl enable --now cups avahi-daemon
```

Confirm services and syntax:

```bash
systemctl is-active cups avahi-daemon
cupsd -t
ss -lntp | grep ':631'
```

Expected: services `active`, valid CUPS config, a listener on TCP 631. Check the local HTTP endpoint:

```bash
curl -I --max-time 10 http://127.0.0.1:631/
```

**If TCP 631 is not listening**, first run `systemctl status cups --no-pager -l` and `journalctl -u cups -n 50 --no-pager`. Don't start configuring printers until CUPS is stable.

The administration UI is normally at `http://<LXC_IP>:631/` once CUPS listens on an accessible address and its access policy permits it. Expose CUPS only on your intended LAN.

## 4. Verify printer connectivity and the IPP endpoint

First check the printer's IPP port:

```bash
nc -zv -w 3 "$PRINTER_IP" 631
```

`open` means a TCP connection was accepted. A message such as `inverse host lookup failed` is usually only a reverse-DNS warning. If the connection fails, check the **current printer IP**, Wi-Fi connectivity, LXC bridge/VLAN/firewall and access restrictions before changing CUPS drivers.

Inspect printer IPP capabilities (after confirming the correct endpoint path):

```bash
ipptool -tv "$PRINTER_URI" \
  /usr/share/cups/ipptool/get-printer-attributes.test
```

Look for `PASS`, `successful-ok`, `printer-make-and-model`, `printer-uri-supported`, and `document-format-supported`. The verified deployment's printer supported PWG Raster (`image/pwg-raster`) but not native PDF; the CUPS server converted documents to the printer's supported format.

If the test fails, **do not assume `/ipp/print` is correct**. Discover the printer using its documentation, web interface, or Bonjour:

```bash
avahi-browse -rt _ipp._tcp
```

Read the printer's `hostname`, `port`, and `rp=` TXT record to construct the actual URI; use only the record belonging to the physical printer, not the CUPS-shared queue.

## 5. Create a direct CUPS print queue

After the IPP endpoint is confirmed:

```bash
lpadmin -p "$QUEUE" -E -v "$PRINTER_URI" -m everywhere
```

`-m everywhere` requests a driverless/IPP Everywhere setup. It worked for the printer in the observed deployment; other devices may require a compatible driver or a different supported print protocol.

Check the created queue:

```bash
lpstat -v
lpstat -p "$QUEUE"
```

Expected URI for the new queue: `ipp://<PRINTER_IP>/<verified-path>`. A queue using `implicitclass://...` is generally auto-managed by `cups-browsed`, **not** the direct IPP queue we want for this setup.

## 6. Enable CUPS sharing

```bash
lpadmin -p "$QUEUE" -o printer-is-shared=true
cupsctl --share-printers
```

Validate the configured share:

```bash
grep -A 20 "<Printer ${QUEUE}>" /etc/cups/printers.conf
```

Look for `Shared Yes` and the expected `DeviceURI`. `cupsctl` may update global CUPS sharing/listening settings; make sure remote clients can access the shared printer on TCP 631. A successful `cupsctl` command does **not** replace network/firewall checks.

If `lpadmin` or `cupsctl` reports `Connection reset by peer`, check whether CUPS is restarting, failing to start or waiting on a dependency; see **Troubleshooting A**.

## 7. Test local printing before AirPrint

```bash
printf 'AIRPRINT TEST\n' | lp -d "$QUEUE"
lpstat -o "$QUEUE"
lpstat -l -p "$QUEUE"
```

The `lp` command should return a job ID. **Confirm that a physical page comes out.** A job ID or a CUPS `completed` status is not by itself proof of physical output.

For a reproducible sample page:

```bash
time lp -d "$QUEUE" /usr/share/cups/data/testprint
```

**Timing distinction:** `time lp` times job submission to CUPS, not the delay until the page exits the printer. Measure physical output separately using a stopwatch.

If jobs remain queued, see **Troubleshooting B**. To list completed jobs:

```bash
lpstat -W completed -o "$QUEUE"
```

## 8. Confirm Avahi / AirPrint discovery

```bash
systemctl is-active avahi-daemon
avahi-browse -rt _ipp._tcp
```

Find the **CUPS-shared** service, typically displayed as `AirPrint_Queue @ <server-name>`, and check:

```text
address = [<LXC_IP>]
port = [631]
rp=printers/AirPrint_Queue
pdl=...image/urf...   # Examples of advertised CUPS-supported formats
URF=...                 # Apple raster capability advertisement
```

The physical printer might appear as a **separate** entry. It is normal for CUPS to advertise formats such as PDF/URF that it converts to the physical printer's accepted format.

**Do not add a custom Avahi service file unless the existing CUPS queue is not being advertised correctly.** This deployment required no custom file.

On an iPhone or iPad on the same discovery-capable network, open a document/photo, choose **Share → Print → Printer**, select the CUPS-shared queue, then print. If it is visible but jobs fail, troubleshoot the CUPS queue/IPP connection rather than mDNS discovery.

---

# Troubleshooting

Use the sections below **only when the matching symptom occurs**. The `colord` fix is not a mandatory step in every LXC deployment.

## A. CUPS web UI is slow, `cupsctl` resets, or CUPS repeatedly restarts

Symptoms seen in the reference deployment:

- Admin commands intermittently failed with `Connection reset by peer`.
- CUPS intermittently failed with `status=1/FAILURE`.
- CUPS startup occasionally took tens of seconds.
- The local web endpoint temporarily refused connections because no TCP 631 listener was active.

Check:

```bash
systemctl status cups cups.socket --no-pager -l
cupsd -t
ss -lntp | grep ':631'
journalctl -u cups -n 60 --no-pager -o short-precise
tail -80 /var/log/cups/error_log
```

**Interpretation:** Valid `cupsd -t` output proves the configuration parses; it does not prove `cupsd` is healthy. A `Connection reset by peer` message is a symptom, not proof of a specific root cause. Correlate timestamps with D-Bus and service errors, especially if print jobs also have fixed-duration stalls.

To measure the HTTP endpoint without browser overhead:

```bash
curl -s -o /dev/null --max-time 10 \
  -w 'HTTP=%{http_code} total=%{time_total}s\n' \
  "http://${LXC_IP}:631/"
```

A returned HTTP `000` means no HTTP response was received; inspect the underlying curl error and CUPS listener. In the observed deployment, the healthy response was HTTP `200` in a few milliseconds.

## B. Queue is idle/processing, no page appears, or destination is missing

Inspect:

```bash
lpstat -t
lpstat -v
lpstat -l -p "$QUEUE"
```

If the queue URI is `implicitclass://...` and you see:

```text
No suitable destination host found by cups-browsed.
```

that is an auto-discovered queue unable to find its backend destination. Prefer a separately configured **direct IPP** queue, once the printer endpoint has been verified. Do **not** delete the existing queue before testing the replacement.

If the direct queue displays `The printer may not exist or is unavailable at this time`, diagnose printer IP reachability and its actual IPP URI (Step 4) before changing print filters.

A printer can temporarily be `idle` after a job has finished or been cancelled; check the completed/not-completed job lists and verify physical output.

## C. Fixed ~50-second delay before printing: `colord` D-Bus activation timeout

This is the **confirmed root cause in the reference Proxmox LXC deployment**, not a universal cause of slow AirPrint.

### How it was identified

The debug CUPS job timeline showed:

| Relative time | Event |
|---|---|
| T+0 s | Job starts; filters and IPP backend launched |
| T+25 s | First D-Bus `NoReply` timeout |
| T+50 s | Second D-Bus `NoReply` timeout; IPP backend begins connecting |
| T+51 s | IPP connection established |
| Later | Printer reports completed job |

The physical printer and port 631 were reachable. CPU/memory were not persistently saturated. Disabling `cups-browsed` did **not** remove the print delay.

To enable richer CUPS logging while investigating:

```bash
cupsctl --debug-logging
```

Check the logs for a stalled job (replace `<JOB_ID>` with a real number):

```bash
grep "\[Job <JOB_ID>\]" /var/log/cups/error_log | \
  grep -E 'NoReply|Started filter|Started backend|Connecting to|Connected to|exited'
```

Check the D-Bus activation target and systemd status:

```bash
systemctl status dbus --no-pager -l
systemctl status colord --no-pager -l
```

The diagnostic signatures were:

```text
org.freedesktop.ColorManager
Failed to activate service ... timed out (service_start_timeout=25000ms)
colord.service: Failed to set up mount namespacing: Permission denied
colord.service: Failed at step NAMESPACE spawning ...
status=226/NAMESPACE
```

The service unit contained `PrivateTmp=yes`, which required a mount namespace operation disallowed by this LXC environment.

### Targeted workaround — only if the above errors match

Inspect the actual unit first:

```bash
systemctl cat colord.service
```

If the failure is `226/NAMESPACE` and the unit uses `PrivateTmp=yes`, create a **service-specific** drop-in:

```bash
mkdir -p /etc/systemd/system/colord.service.d
printf '[Service]\nPrivateTmp=no\n' > /etc/systemd/system/colord.service.d/lxc.conf
systemctl daemon-reload
systemctl show colord.service -p PrivateTmp
```

Expected: `PrivateTmp=no`. Then reset the failed state and try to start it:

```bash
systemctl reset-failed colord.service
systemctl start colord.service
systemctl is-active colord.service
```

Expected: `active` immediately after starting. `colord` is typically D-Bus activated and may be listed as `static` by `systemctl is-enabled`; that is **normal**.

Check that the drop-in is persistent:

```bash
systemctl cat colord.service
```

The effective configuration should include `/etc/systemd/system/colord.service.d/lxc.conf` and `PrivateTmp=no`.

**Observed outcome:** roughly **50 seconds → 5 seconds** for physical printing. The workaround changes only `colord`'s service-level temporary-directory isolation; no global LXC privilege change was needed. For a different LXC, verify the exact failure before applying this workaround.

Once diagnostics are complete, return CUPS logging to normal:

```bash
cupsctl --no-debug-logging
```

## D. Duplicate printer and `cups-browsed` automatically created `implicitclass://` queue

`cups-browsed` discovers and manages remote printers/queues. It is not required for a direct, manually created CUPS queue shared via Avahi.

If you do not use auto-created remote queues, optionally disable it:

```bash
systemctl disable --now cups-browsed
```

Re-enable it if automatic discovery is needed later:

```bash
systemctl enable --now cups-browsed
```

This was **not** the root-cause fix for the 50-second delay. Leave `avahi-daemon` running for Bonjour discovery. Remove unused auto-created queues only after confirming the direct queue works.

## E. iPhone cannot see the shared printer

Check in order:

1. `lpstat -v` confirms a working direct IPP queue.
2. CUPS printer configuration contains `Shared Yes` and sharing is enabled.
3. `systemctl is-active avahi-daemon` reports `active`.
4. `avahi-browse -rt _ipp._tcp` shows the **CUPS-shared** queue at the LXC IP, with correct `rp=` and capability records.
5. The iPhone can reach the same Bonjour discovery domain (avoid guest Wi-Fi/client isolation; account for VLAN/mDNS boundaries).
6. TCP 631 to CUPS and mDNS UDP 5353 are not blocked.

Seeing the physical printer's service does **not** prove the CUPS queue is advertised; distinguish the two service entries.

## F. Locale warning during package or service management

If the container prints `Setting locale failed` for `en_US.UTF-8`, the locale may not be generated. This warning was **not** the print-delay cause in the observed deployment.

If you want to fix it:

```bash
apt install -y locales
sed -i 's/^# *\(en_US.UTF-8 UTF-8\)/\1/' /etc/locale.gen
locale-gen
update-locale LANG=en_US.UTF-8
```

Open a fresh shell and inspect `locale -a`.

---

## Final verification checklist

```text
[ ] The LXC can reach the printer IP and IPP endpoint
[ ] CUPS and Avahi are running
[ ] CUPS remains stable and listens on TCP 631
[ ] A direct ipp:// or ipps:// queue exists (not an implicitclass:// fallback)
[ ] Printer queue is shared (Shared Yes)
[ ] Test document prints physically from the LXC
[ ] Avahi advertises the CUPS queue with the correct rp= path
[ ] iPhone finds the queue and physically prints a document
[ ] If D-Bus/colord timeout occurred, it is fixed and retested
[ ] If not needed, cups-browsed auto-creation is disabled
[ ] Debug logging is disabled after troubleshooting
[ ] Printer and LXC have stable addresses / DHCP reservations
[ ] A Proxmox backup or snapshot is available
```

Service startup status can be checked with:

```bash
systemctl is-enabled cups avahi-daemon colord.service cups-browsed
```

One successful reference configuration returned:

```text
enabled
enabled
static
disabled
```

That output is not mandatory for all environments; `cups-browsed` is optional and `colord` being `static` is expected for its D-Bus activation model.

### Minimal routine health check

```bash
systemctl is-active cups avahi-daemon
lpstat -p "$QUEUE"
lpstat -v
avahi-browse -rt _ipp._tcp
```

### Maintenance notes

- Use DHCP reservations to keep CUPS-to-printer URIs stable.
- Restrict CUPS to the intended LAN. Treat firewall/mDNS settings as network-specific.
- If restoring the container onto a different network, update the LXC IP, printer IP/URI, and discovery policies.
- Re-test with an iPhone after a container reboot; service enablement alone does not prove end-to-end printing.

---

## Appendix: what happened in the reference troubleshooting session

This section is included to help future maintainers recognize the **pattern**, not to prescribe every historical action as a setup step.

| Observed issue | Evidence | Conclusion |
|---|---|---|
| `cupsctl` / `lpadmin`: `Connection reset by peer` | CUPS temporarily restarting / not listening | Investigate service state and dependency failures; not automatically an AirPrint issue |
| No suitable destination host | Auto-generated `implicitclass://` queue | Use direct, verified IPP URI for a manually managed queue |
| Printer initially unreachable | `Destination Host Unreachable`, TCP 631 timeout | Network reachability needed checking; subsequent tests found ARP `REACHABLE` and TCP 631 `open` |
| Wrong/uncertain IPP endpoint | `ipptool` eventually returned `PASS` | Confirm IPP endpoint and supported formats before configuring queue |
| Queue completed but physical print needed confirmation | `lpstat` completed entry | Verify paper actually emerges |
| Web UI intermittently slow and CUPS restarts | CUPS startup failures and D-Bus logs | Diagnose service before tuning print quality or network |
| ~50-second startup-of-print delay | Two 25-second `org.freedesktop.ColorManager` timeouts | `colord` activation was blocking printing |
| `colord` cannot launch in LXC | `226/NAMESPACE`, `PrivateTmp=yes` | Apply a **targeted** `PrivateTmp=no` systemd drop-in after confirming the same failure |
| Delay remained with `cups-browsed` stopped | Retest still ~50 seconds | `cups-browsed` was not the primary delay cause |
| Print after `colord` fix | About five seconds in the observed test | Root cause effectively resolved |

**Privacy:** Device brands/models, container IDs, MAC addresses, UUIDs, hostnames, internal addresses, and original queue names are intentionally absent. The `192.0.2.0/24` examples are documentation-only. Replace them for your deployment.

**Guide version:** 1.0 · 2026-11-22
