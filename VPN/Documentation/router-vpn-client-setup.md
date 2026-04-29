# Router-Level VPN Setup with ProtonVPN

**Author:** Sebastian Dorata  
**Tags:** Networking, VPN, OpenVPN, ProtonVPN, TP-Link, Privacy, Home Lab, TCP, UDP

---

## What I Was Trying to Do

I wanted VPN protection on devices that cannot run a VPN app natively, mainly consoles and smart devices. Instead of installing ProtonVPN on each machine individually, I configured it at the router level so any device I choose gets tunneled automatically.

I also wanted split tunneling so some devices go through the VPN and others do not. My Proxmox server for example should not go through a VPN since it needs stable inbound connectivity for SSH and Tailscale.

---

## Hardware and Software

| Item | Detail |
|---|---|
| Router | TP-Link Archer AX23 |
| VPN Provider | ProtonVPN (Plus) |
| Protocol | OpenVPN |
| Server Location | United States |

---

## Step 1: Get the OpenVPN Credentials from ProtonVPN

ProtonVPN uses separate credentials for OpenVPN, they are not the same as your Proton account login. I found them under **Account > OpenVPN/IKEv2 username** in the ProtonVPN dashboard.

---

## Step 2: Generate the Config File

In the ProtonVPN dashboard under **Downloads > OpenVPN configuration files**, I selected:
- Platform: Router
- Protocol: UDP
- Country: United States

This downloaded a `.ovpn` file which contains the server address, port, and the certificates needed to establish the tunnel.

## Why I Chose UDP Over TCP

I went with UDP for this setup because my main use case is streaming.
UDP is faster than TCP since it does not wait for packet acknowledgment before sending the next one.
That lower overhead makes a noticeable difference for video streaming where a dropped packet is better tolerated than the delay caused by retransmission.

TCP on the other hand guarantees delivery and reorders packets, which sounds better on paper but introduces latency that works against real-time or continuous playback.
For a use case like browsing or file transfers TCP would make more sense, but for consoles and smart TVs pushing video, UDP was the right call.

---

## Step 3: Configure the Router

I logged into the AX23 admin panel and went to **VPN Client**. After enabling it and hitting Apply, I added a new server entry with:
- VPN Type: OpenVPN
- Credentials: my ProtonVPN OpenVPN username and password
- Config file: the `.ovpn` file from Step 2

Once saved, I toggled the server on and the status changed from Connecting to Connected.

---

## Step 4: Assign Devices (Split Tunneling)

Under the **Device List** section I added only the devices I wanted routed through ProtonVPN. Anything not on the list uses the regular ISP connection.

I deliberately left Proxmox and anything related to the home lab off the list. Sending that traffic through a VPN would break SSH access and interfere with Tailscale.

---
