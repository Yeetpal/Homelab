# Pi-hole on Raspberry Pi — Setup & Integration Guide

> A practical reference for deploying Pi-hole on a Raspberry Pi and integrating it with a Ubiquiti UniFi network as the primary DNS resolver. Covers OS selection, installation, UniFi DHCP configuration, local DNS records, and NTP fixes specific to hardware-clock-less Pi hardware.

---

> ### 🔑 Placeholder Key
> Wherever you see text in `ALL_CAPS_WITH_UNDERSCORES`, replace it with your own value before running the command.
>
> | Placeholder | What to enter |
> |---|---|
> | `YOUR_PIHOLE_IP` | Local IP of your Raspberry Pi e.g. `192.168.0.x` |
> | `YOUR_GATEWAY_IP` | Local IP of your Ubiquiti gateway e.g. `192.168.0.1` |
> | `YOUR_LOCAL_DOMAIN` | Your local domain suffix e.g. `home` or `lan` |
> | `YOUR_SERVER_IP` | Local IP of a server you want a DNS record for e.g. `192.168.0.x` |
> | `YOUR_SERVER_NAME` | Friendly name for that server e.g. `mediaserver` |

---

## Table of Contents

1. [Phase 1 — OS Selection and Flashing](#phase-1--os-selection-and-flashing)
2. [Phase 2 — Pi-hole Installation](#phase-2--pihole-installation)
3. [Phase 3 — UniFi Integration](#phase-3--unifi-integration)
4. [Phase 4 — Local DNS Records](#phase-4--local-dns-records)
5. [Phase 5 — Conditional Forwarding](#phase-5--conditional-forwarding)
6. [Troubleshooting](#troubleshooting)

---

## Phase 1 — OS Selection and Flashing

### 1.1 Choose the right image

Use **Raspberry Pi OS Lite (32-bit)**. There is no need for a desktop environment — Pi-hole runs headlessly and the Lite image conserves the RAM and CPU the Pi needs for DNS processing.

> ℹ️ On older hardware like the Pi 2B, the 32-bit image is required. The 64-bit image is only supported on Pi 3 and newer.

### 1.2 Flash the SD card

Download [Raspberry Pi Imager](https://www.raspberrypi.com/software/) and flash the card with the following settings in the **Edit Settings** (gear icon) dialog before writing:

| Setting | Value |
|---|---|
| Hostname | `pihole` (or your preference) |
| Username / Password | Set and record securely |
| SSH | Enable (password authentication) |

> ⚠️ Enable SSH before writing the image. Without it you will need a monitor and keyboard to access the Pi on first boot.

### 1.3 Assign a static IP in UniFi

Reserve the Pi's MAC address with a fixed IP in UniFi as soon as it appears on the network. This prevents the IP from changing later and breaking DNS for your entire network.

1. Plug the Pi in and let it boot (~2 minutes)
2. UniFi Dashboard → **Client Devices** → find `pihole`
3. Click the client → **Settings** → toggle **Fixed IP** → enter `YOUR_PIHOLE_IP`
4. Apply

> ⚠️ If the Pi's IP ever changes after you have configured your network to use it for DNS, every device on the network loses internet access until the DHCP record is corrected. Assign the fixed IP before you point anything at it.

---

## Phase 2 — Pi-hole Installation

### 2.1 Connect via SSH

```bash
ssh YOUR_USERNAME@YOUR_PIHOLE_IP
```

### 2.2 Run the automated installer

```bash
curl -sSL https://install.pi-hole.net | bash
```

Work through the blue on-screen prompts:

| Prompt | Recommended choice |
|---|---|
| Static IP | Confirm the IP already reserved in UniFi |
| Upstream DNS | Cloudflare (`1.1.1.1`) or Quad9 (`9.9.9.9`) — see note below |
| Web interface | Yes |
| Query logging | **Show everything** — required for troubleshooting blocked services |

> **Upstream DNS:** Cloudflare (`1.1.1.1`) is the fastest general-purpose option and handles the rapid-fire API calls made by services like Sonarr and Radarr without latency. Quad9 (`9.9.9.9`) adds malware domain blocking at the DNS level if you prefer that. Either is a significant improvement over ISP-provided DNS.

> ⚠️ The final screen of the installer shows your admin dashboard password. Copy it before the screen clears.

### 2.3 Run Update Gravity

After installation, force Pi-hole to download its blocklists:

1. Open `http://YOUR_PIHOLE_IP/admin` in a browser
2. **Tools → Update Gravity → Update**

A successful run shows a domain count (typically 100,000+) in the dashboard. If the count shows `Error (-2)`, the Pi cannot reach the internet — see the Gravity Update Failure entry in the Troubleshooting section.

Once gravity completes, add the Hagezi Light blocklist via **Lists → Add Blocklist**, then run Update Gravity again to apply it:

```
https://raw.githubusercontent.com/hagezi/dns-blocklists/main/adblock/light.txt
```

> ℹ️ Hagezi is actively maintained and tiered by aggressiveness. Light is a solid default with minimal false positives. Higher tiers (Normal, Pro) are available if you want broader coverage at the cost of occasionally needing to whitelist legitimate domains.

---

## Phase 3 — UniFi Integration

### 3.1 Disable UniFi's built-in ad blocking

Having two DNS-level blockers active simultaneously causes unpredictable behaviour and makes troubleshooting much harder.

UniFi Dashboard → **Settings → Security → Ad Blocking → Off**

### 3.2 Set Pi-hole as the DHCP DNS server

This is the cleanest integration method. Devices receive Pi-hole's IP directly from DHCP and query it first, which means Pi-hole can attribute queries to individual clients rather than seeing everything come from the gateway.

1. UniFi Dashboard → **Settings → Networks** → click your primary network
2. Scroll to **DHCP Service → DHCP DNS Server**
3. Switch from **Auto** to **Manual**
4. **DNS Server 1:** `YOUR_PIHOLE_IP`
5. **DNS Server 2:** leave blank
6. **Apply Changes**

> ⚠️ Do not put a fallback like `8.8.8.8` in the second slot. Devices will round-robin between the two and bypass Pi-hole roughly half the time, breaking local DNS records and ad blocking.

### 3.3 Force devices to pick up the new settings

Devices hold their existing DHCP lease and will not switch to Pi-hole until the lease renews (up to 24 hours by default in UniFi). To force an immediate switch:

- **Windows:** `ipconfig /release && ipconfig /renew`
- **Phones / other devices:** toggle Wi-Fi or Ethernet off and back on

### 3.4 Verify the change

```bash
nslookup google.com
```

The response should show `Server: YOUR_PIHOLE_IP`. If it still shows your gateway IP, the DHCP lease has not renewed yet — repeat the step above.

---

## Phase 4 — Local DNS Records

Pi-hole can serve as an internal DNS server, mapping friendly names to local IPs. This is useful for accessing services by name rather than IP and port.

### 4.1 Add an A record

**Pi-hole Dashboard → Local DNS → DNS Records → Add**

| Field | Value |
|---|---|
| Domain | `YOUR_SERVER_NAME.YOUR_LOCAL_DOMAIN` |
| IP Address | `YOUR_SERVER_IP` |

Example: `mediaserver.home` → `192.168.0.96`

> ⚠️ Enter only the IP address here — no `http://`, no port numbers. DNS maps names to IPs only. To access a specific service, append the port in the browser: `http://mediaserver.home:8989`

### 4.2 Add CNAME records for multiple services on one machine

If several services run on the same IP, create one A record and use CNAMEs to alias additional names to it. This way, if the server IP changes, you update one A record and all CNAMEs follow automatically.

**Pi-hole Dashboard → Local DNS → CNAME Records → Add**

| Domain (alias) | Target domain |
|---|---|
| `service1.YOUR_LOCAL_DOMAIN` | `YOUR_SERVER_NAME.YOUR_LOCAL_DOMAIN` |
| `service2.YOUR_LOCAL_DOMAIN` | `YOUR_SERVER_NAME.YOUR_LOCAL_DOMAIN` |

---

## Phase 5 — Conditional Forwarding

By default, Pi-hole logs client queries by IP address only. Conditional forwarding instructs Pi-hole to ask your gateway to resolve those IPs back to hostnames (e.g. `192.168.0.25` → `johns-iphone`), making the query log readable.

**Pi-hole Dashboard → Settings → DNS → Conditional Forwarding**

| Field | Value |
|---|---|
| Use conditional forwarding | ✅ Enabled |
| IP address of router | `YOUR_GATEWAY_IP` |
| Local domain name | `YOUR_LOCAL_DOMAIN` |

Save and the client list in the dashboard will begin showing device names.

---

## Troubleshooting

---

### ✅ NTP — Time Sync Fails on Boot (No Hardware Clock)

**Symptoms:**
- Pi-hole diagnosis page showing `Warning in NTP client: No valid NTP replies received`
- `Error in NTP client: Cannot resolve NTP server address: Try again`

**Cause:** Two separate NTP clients are involved and both need fixing.

The Raspberry Pi 2B has no battery-backed hardware clock. On every boot, `systemd-timesyncd` must sync the system time from an NTP server before Pi-hole's DNS service is fully ready — a bootstrap loop where NTP needs DNS to resolve the time server hostname, but DNS isn't up yet.

Separately, Pi-hole v6 runs its own NTP client inside FTL, independent of the OS. Even after `systemd-timesyncd` is correctly configured, FTL's NTP client will continue generating warnings in the Pi-hole diagnosis page if it cannot independently reach a time server.

**Fix — Part 1: OS-level time sync (`systemd-timesyncd`)**

1. Manually set the system time to break the bootstrap loop:

```bash
sudo timedatectl set-ntp false
sudo timedatectl set-time "YYYY-MM-DD HH:MM:SS"
sudo timedatectl set-ntp true
```

2. Configure `systemd-timesyncd` to use direct IP addresses instead of hostnames, removing the DNS dependency:

```bash
sudo nano /etc/systemd/timesyncd.conf
```

Find the `#NTP=` line, remove the `#`, and set it to direct IPs for Google and Cloudflare time servers:

```
NTP=216.239.35.0 162.159.200.1
```

3. Restart the service and verify:

```bash
sudo systemctl restart systemd-timesyncd
timedatectl status
```

Look for `System clock synchronized: yes` before continuing.

**Fix — Part 2: Pi-hole FTL NTP client**

Pi-hole's FTL component runs its own NTP client. Disable it and let the OS handle time sync entirely.

The recommended method is via the dashboard: **Settings → All Settings → search "ntp" → toggle Expert mode → disable `ntp.ipv4.active`**.

Alternatively, edit `/etc/pihole/pihole.toml` directly and restart FTL with `sudo systemctl restart pihole-FTL`. The relevant keys are under the `[ntp]` section — refer to the Pi-hole v6 documentation for the exact structure if editing by hand, as key names may vary between versions.

Clear the warnings from the Pi-hole diagnosis page. They should not return.

**Note:** Both parts are required. Fixing only `systemd-timesyncd` keeps the OS clock accurate but leaves FTL's NTP client running and generating warnings. Fixing only FTL silences the warnings but does not resolve the underlying boot-time DNS dependency for the system clock.

---

### ✅ Gravity Update Failure — Error (-2) on Dashboard

**Symptoms:** Dashboard shows `Error (-2)` for domains on blocklists after installation, and Update Gravity fails or shows network errors.

**Cause:** The Pi cannot reach the internet to download blocklists. Most commonly caused by the Pi's own DNS resolver pointing at itself (`127.0.0.1`) before Pi-hole is fully initialised, or a DHCP lease that hasn't resolved correctly.

**Fix:**

Check what DNS the Pi is using for its own resolution:

```bash
cat /etc/resolv.conf
```

If it shows `nameserver 127.0.0.1` only, the Pi is asking itself for DNS. Set a direct upstream temporarily so the Pi can reach the internet independently of its own service:

```bash
sudo nmcli con mod "$(nmcli -t -f NAME con show --active | head -1)" ipv4.dns "1.1.1.1 1.0.0.1"
sudo nmcli con mod "$(nmcli -t -f NAME con show --active | head -1)" ipv4.ignore-auto-dns yes
sudo nmcli con up "$(nmcli -t -f NAME con show --active | head -1)"
```

Then re-run Update Gravity from the dashboard. Once gravity updates successfully, the Pi-hole service itself handles upstream resolution and this setting ensures the Pi's own processes (NTP, apt, etc.) can always reach the internet independently.

---

### ✅ DNS Not Switching After UniFi Change (Stale Lease)

**Symptoms:** After configuring Pi-hole as the DHCP DNS server in UniFi, `nslookup` still shows the gateway IP as the DNS server on some devices.

**Cause:** The device's DHCP lease has not expired yet. UniFi's default lease time is 24 hours.

**Fix:** Force a DHCP renewal on the affected device:

- **Windows:** `ipconfig /release && ipconfig /renew`
- **macOS / Linux:** disconnect and reconnect the network interface
- **Mobile:** toggle Wi-Fi off and on

Verify with `nslookup google.com` — the Server field should now show `YOUR_PIHOLE_IP`.

---

## Useful Commands Reference

```bash
# Check Pi-hole service status
pihole status

# Update blocklists (gravity) from CLI
pihole -g

# View DNS query log live
pihole -t

# Temporarily disable Pi-hole for 5 minutes (useful for testing)
pihole disable 5m

# Re-enable Pi-hole
pihole enable

# Check NTP sync status
timedatectl status

# Check which DNS server the Pi itself is using
cat /etc/resolv.conf

# Test a specific domain resolution
nslookup google.com YOUR_PIHOLE_IP
```
