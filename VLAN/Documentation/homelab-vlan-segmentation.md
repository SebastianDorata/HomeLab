# Home Lab Network Segmentation with VLANs

**Author:** Sebastian Dorata  
**Tags:** Networking, VLANs, Proxmox, Home Lab, Network Security

---

## What I Was Trying to Do

My home network had everything on the same flat network: my PC, laptops, consoles, Sonos, and my Proxmox server all sitting on the same broadcast domain. That means if anything on the network got compromised, it could potentially reach everything else. I wanted to segment things so that IoT devices and consoles are isolated from my servers and daily machines.

I'm also running Proxmox as a home lab server and didn't want it sitting exposed on the same network as everything else.

---

## Hardware

| Device | Role |
|---|---|
| TP-Link Archer AX23 | Main router (Layer 2/3)|
| TP-Link TL-SG108E | 8-port managed switch (Layer 2) with 802.1Q VLAN support |
| TP-Link TL-SG108 | 8-port unmanaged switch (Layer 1) downstream for consoles, TV, and speakers |
| 2017 MacBook Air (bare metal Proxmox VE) | Home lab server / hypervisor |

---

## VLAN Design

The first decision was to leave VLAN 1 completely unused. VLAN 1 is the default on every switch and is a known attack surface, it carries default trust assumptions baked into a lot of hardware. Leaving it empty is standard practice.

I ended up with four VLANs:

| VLAN ID | Name | What's on it |
|---|---|---|
| 10 | Management | Proxmox host itself |
| 20 | Servers | Proxmox VMs |
| 30 | Main_LAN | PC, laptops |
| 40 | IoT | Consoles, TV, speakers via TL-SG108 |

The idea is that VLAN 40 devices have no reason to talk to VLAN 10 or 20. When I set up pfSense as a VM on Proxmox, I'll write explicit firewall rules to control what can cross VLANs. For now the switch handles segmentation and the AX23 stays as the router because it doesn't support inter-VLAN routing on its own.

---

## Physical Port Assignments

| Port | Device | VLAN |
|---|---|---|
| Port 1 | PC (Plex) | VLAN 30 untagged |
| Port 2 | Xubuntu laptop | VLAN 30 untagged |
| Port 3 | Proxmox | VLAN 10 untagged, VLAN 20 tagged |
| Port 4 | TP-Link TL-SG108 (unmanaged) | VLAN 40 untagged |
| Port 5 | Router uplink | All VLANs tagged |
| Ports 6–8 | Unused | Not member |

Port 3 carries both VLAN 10 and 20 over a single cable. It's untagged on VLAN 10 so the Proxmox host itself lands on the management VLAN without needing to be VLAN-aware. It's tagged on VLAN 20 so I can assign VMs to the servers VLAN from within Proxmox.

Port 4 connects to the TP-Link TL-SG108, an unmanaged 8-port switch that acts as a Layer 1 device, it has no VLAN awareness and simply passes traffic through. The consoles, TV, and speakers all plug into it, and it connects back up to Port 4 on the TL-SG108E. Because the TL-SG108E port is untagged on VLAN 40, all traffic from those devices gets placed into VLAN 40 automatically without any configuration needed on the unmanaged switch itself.

Port 5 is a trunk port; it carries all VLANs tagged so the router sees all traffic. When pfSense eventually replaces the AX23, inter-VLAN routing rules will live there.

---

## Switch Configuration

I did everything through the TL-SG108E web UI under **VLAN then 802.1Q VLAN**. First I enabled 802.1Q mode and hit Apply, then created each VLAN one at a time. The final table:

```
VLAN ID   Name        Member Ports   Tagged Ports   Untagged Ports
1         Default     1-8                           1-8
10        Management  3,5            5              3
20        Servers     3,5            3,5
30        Main_LAN    1-2,5          5              1-2
40        IoT         4-5            5              4
```

lastly, I saved the configuration so it persists on reboot.
