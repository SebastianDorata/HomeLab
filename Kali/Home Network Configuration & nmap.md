# Home Network Configuration & nmap


**Topology:**
```
ISP → Main house modem/router → TP-Link router  → devices
```
This is a double NAT setup. All devices below are isolated.

---

## Tools Used

- `nmap 7.99` on macOS (Apple Silicon)
- TP-Link router admin panel
- TL-SG108E managed switch web UI

---

## Network Discovery with Nmap

### Ping sweep to find live hosts
```bash
nmap -sn (IP address)/24
```
Identified two live hosts: My router and switch.

### Port scan router
```bash
nmap --open (IP address)
```
```
PORT     STATE SERVICE
53/tcp   open  domain
80/tcp   open  http
443/tcp  open  https
1900/tcp open  upnp
```

### OS/service fingerprint on unknown host
```bash
nmap -A (IP address)
```
Returned a TP-Link login page on port 80 — confirmed as the TL-SG108E managed switch.

---

## DHCP Reservation (Static IP via MAC)

To ensure the media server always receives the same IP:

**Router admin panel → Advanced → Network → DHCP Server → Address Reservation**



The router assigns this IP every time the device connects, based on its MAC address.

---

## UPnP

UPnP was found enabled on the router with an active mapping from the media server:

| Service           | Client IP    | Internal Port | External Port | Protocol |
|-------------------|--------------|---------------|---------------|----------|
| Plex Media Server | (IP address) | 32400         | 10415         | TCP      |

Since Plex is only used locally (not accessed remotely), UPnP was disabled and no port forwarding rules are required.

**Router admin panel → Advanced → NAT Forwarding → UPnP → Disabled**

---

## Notes

- Port `1900` (UPnP/SSDP) was open on the router during initial scan. UPnP disabled after confirming it was not needed for local-only Plex usage.
- Double NAT means external port forwarding on the TP-Link router alone would not expose services to the internet — the upstream house router would also need to forward ports.
- When I migrate from Plex to **Jellyfin**, no network reconfiguration is needed. Jellyfin defaults to port `8096`. DHCP reservation carries over as it is tied to the MAC address.

---

