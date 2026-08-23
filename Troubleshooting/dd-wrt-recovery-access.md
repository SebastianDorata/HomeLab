# DD-WRT Router Recovery Access Guide

Reference for regaining access to a DD-WRT router (RT-AC3200) when locked out of the web GUI, including factory reset and cross-subnet access via a secondary IP.

## Table of Contents

- [Overview](#overview)<br>
- [Step 1: Diagnose the Lockout](#step-1)<br>
- [Step 2: Factory Reset (30/30/30)](#step-2)<br>
- [Windows: Access Router on Different Subnet](#windows-access)<br>
- [Mac: Access Router on Different Subnet](#mac-access)<br>
- [Step 3: Log Into DD-WRT GUI](#step-3)<br>
- [Cleanup: Remove Secondary IP](#cleanup)<br>
- [Help](#help)<br>


>Note: Do not do this when using RDP, the connection will drop.

---
<a id = "overview" ></a>
## Overview

If DD-WRT's web GUI (httpd) stops responding — often from repeated failed login attempts — and the router won't come back after a power cycle, a full 30/30/30 reset is required. This wipes the router's **configuration only** (passwords, static IP, wireless, DHCP, Samba, etc.) — the DD-WRT **firmware itself is untouched**.

After reset, the router returns to its default subnet (`192.168.1.1`), which may differ from your main LAN (`192.168.0.x`). To reach it without a direct cable, add a secondary IP address on `192.168.1.x` to your computer's active network interface.

---
<a id = "step-1"></a>
## Step 1: Diagnose the Lockout

Check whether the router is alive at all, and whether it's just the web service that's down.

```bash
ping <router-ip>
```

```bash
nmap -sn 192.168.0.0/24
```

```bash
sudo nmap -sn -PR 192.168.0.0/24
```

If ping succeeds but the GUI won't load, and SSH is refused:

```
ssh root@<router-ip>
```

---
<a id = "step-2"></a>
## Step 2: Factory Reset (30/30/30)

Only do this if the web GUI and SSH are both unreachable and credentials are unknown.

1. With the router powered ON, hold the reset button for **30 seconds**.
2. Keep holding, unplug the power, continue holding for another **30 seconds**.
3. Keep holding, plug power back in, hold for a final **30 seconds**.
4. Release and let the router boot fully (60–90 seconds).

Default DD-WRT credentials after reset:

```
Username: root
Password: admin
```

Default router IP after reset:

```
192.168.1.1
```

---
<a id = "windows-access"></a>
## Windows: Access Router on Different Subnet

Use when your PC is on `192.168.0.x` but the router is on `192.168.1.x`, and both are on the same physical switch.

**1. Confirm your adapter name:**

```
netsh interface ipv4 show interfaces
```

**2. Add a secondary IP address to the adapter:**

```
netsh interface ipv4 add address name="Ethernet" addr=192.168.1.2 mask=255.255.255.0
```

**3. Verify:**

```
ipconfig
```

**4. Test connectivity:**

```
ping 192.168.1.1
```

---
<a id = "mac-access"></a>
## Mac: Access Router on Different Subnet

Use when your Mac is on `192.168.0.x` but the router is on `192.168.1.x`, and both are on the same physical switch.

**1. Confirm your active interface (not always en0):**

```bash
ifconfig | grep "status: active" -B 5
```

```bash
route get default
```

**2. Add a secondary IP alias to the active interface** (replace `en5` with your actual active interface):

```bash
sudo ifconfig en5 alias 192.168.1.2 255.255.255.0
```

**3. Verify:**

```bash
ifconfig en5
```

**4. Test connectivity:**

```bash
ping 192.168.1.1
```

---
<a id = "step-3"></a>
## Step 3: Log Into DD-WRT GUI

```
http://192.168.1.1
```

```
Username: root
Password: admin
```

>DD-WRT will prompt for a new username/password on first login after reset.

---
<a id = "cleanup"></a>
## Cleanup: Remove Secondary IP

Once done reconfiguring the router, remove the temporary alias.

**Windows:**

```
netsh interface ipv4 delete address name="Ethernet" addr=192.168.1.2
```

**Mac:**

```bash
sudo ifconfig en5 -alias 192.168.1.2
```

---
<a id = "help" ></a>
# References:


1. 

|Commands	|links	|
|---		|---				|
| sudo | https://unix.stackexchange.com/questions/395548/what-does-sudo-mean-and-do|
|netsh | https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/netsh|
|ifconfig | https://www.geeksforgeeks.org/linux-unix/ifconfig-command-in-linux-with-examples/|
| en5 and other interfaces| https://unix.stackexchange.com/questions/603506/what-are-these-ifconfig-interfaces-on-macos|

2. 
|Tools	|Installation links	|
|---		|---				|
|nmap (homebrew) | https://formulae.brew.sh/formula/nmap|
|nmap (Windows) | https://nmap.org/download#windows|




