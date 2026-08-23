# Homelab Documentation

**Sebastian Dorata** — B.A., Hons. Information Technology Student, York University  
Expected Graduation: April 2027

[Skip to view my certificates](#certs)


## About

This repository documents my personal homelab, a hands-on environment I built and maintain to develop practical skills in network engineering, cybersecurity, and IT/OT infrastructure. 
It serves as a living record of my configurations, design decisions, and technical write-ups as I progress through my degree and work toward a career in network and security engineering.

My academic background focuses on network design, distributed systems, and security principles. The projects documented here extend my theoretical foundation into real hardware and real problems.


## What's in This Repo

### Troubleshooting
Real incidents I encountered and resolved, documented with root cause analysis and step-by-step recovery procedures.

- **Router Brick Recovery** — Recovery process after a failed firmware update rendered a router unresponsive, including serial console access and firmware restoration.


### VLAN — Network Segmentation
Design and implementation of a VLAN-segmented home network to isolate traffic between trusted devices, IoT equipment, and a management interface.

- **Documentation/** — Architecture overview, switch configuration rationale, and diagrams illustrating traffic flow and segmentation policy.

Key concepts covered: Layer 2/3 segmentation, inter-VLAN routing, firewall rules on managed switches, trunk and access port configuration.


### VPN — Router-Level VPN Client
Configuration of a ProtonVPN client running directly on the router, enforcing VPN tunneling for selected VLANs at the network perimeter.

- **Documentation/** Setup walkthrough, configuration decisions, and result screenshots showing tunnel verification and DNS leak testing.

Key concepts covered: OpenVPN client configuration, policy-based routing, kill switch implementation, DNS leak prevention.


## Skills Demonstrated

| Area | Details |
|---|---|
| Network Design | VLAN segmentation, inter-VLAN routing, firewall rule design |
| VPN & Tunneling | OpenVPN, ProtonVPN, policy routing, DNS leak testing |
| Security Practices | Network isolation, principle of least privilege, honeypot deployment |
| Virtualization | Proxmox, Linux VM management |
| Documentation | Architecture diagrams, configuration documentation, incident write-ups |


## Context

I am actively developing hands-on skills to complement my coursework in preparation for a career in IT/OT network engineering and cybersecurity. Areas of focus include:

- Network infrastructure design (LAN/WAN, VLANs, wireless, structured cabling concepts)
- Cybersecurity practices aligned with frameworks such as NIST and ISA/IEC 62443
- Physical and logical security system design
- IT/OT convergence and industrial network principles

I hold the **Google Cybersecurity Certificate** and have obtained **CompTIA's Security+** certification. I am also pursuing relevant experience through co-op placements begining in September 2026.


## Note on Repository Contents

VPN config files and switch configuration files containing credentials or sensitive network details are excluded from this repository via `.gitignore`. Only architecture documentation and write-ups are published here.


<a id= "certs"></a>
# My Certificates
![CompTIA Security+](/Certificates/imgs/compTIASecurity+.jpg)
![CompTIA Security+](/Certificates/imgs/GoogleCyberSecurity.jpg)
![CompTIA Security+](/Certificates/imgs/PacketTracer1.jpg)
![CompTIA Security+](/Certificates/imgs/PacketTracer2.jpg)

- [pdfs](/Certificates/)


