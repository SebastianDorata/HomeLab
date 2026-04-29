# Recovering a Bricked TP-Link Archer AX23

**Author:** Sebastian Dorata  
**Tags:** Networking, Troubleshooting, Firmware Recovery, TP-Link, Home Lab

---

## What Happened

After a firmware issue, my router powered on but was completely unresponsive. No admin panel, no DHCP, no internet. The only sign of life was a single orange LED in the middle.

---

## Troubleshooting

Before touching any software I wanted to rule out hardware first. The symptoms could just as easily have been caused by a failing power adapter, and flashing firmware on a device with an unstable power supply would likely cause another failed flash.

I swapped the power adapter with a known working one. Same symptoms, so hardware was ruled out.

With that cleared, the LED behavior matched what TP-Link's documentation describes for a corrupted firmware state, likely caused by a failed or interrupted upgrade. I found the recovery procedure on the TP-Link community forum and confirmed it against the official FAQ: [https://www.tp-link.com/us/support/faq/1482/](https://www.tp-link.com/us/support/faq/1482/)

---

## Recovery Process

The AX23 has a rescue mode that lets you push firmware directly to the router over ethernet, completely bypassing the normal boot process.

### 1. Download the firmware

I went to the TP-Link Download Center, found the Archer AX23 page, matched the hardware version from the label on the bottom of the router, and downloaded the latest firmware zip. Inside the zip is a `.bin` file, that is what gets flashed.

Worth noting: the firmware has to be from the correct regional site. Using firmware from another region can cause further problems.

### 2. Enter rescue mode

- Powered the router off
- Held the WPS button on the back
- Powered it on while still holding WPS
- Held for about 5 seconds until only the middle LED lit orange
- Released the button

The router was now in rescue mode.

### 3. Connect directly via ethernet

I connected my Mac to one of the yellow LAN ports using an ethernet cable.

### 4. Set a static IP on the Mac

The router in rescue mode has no DHCP server running, so the Mac will not get an IP automatically. I had to set one manually on the same subnet as the router's rescue mode address so the two devices could talk.

On macOS: **System Preferences > Network > Ethernet > Configure IPv4: Manually**

I set my IP to  `192.168.0.10` with a subnet mask of `255.255.255.0`.

I also disabled Wi-Fi during this so traffic would not accidentally route the wrong way.

### 5. Access the rescue page

Opened a browser and navigated to `http://192.168.0.10`. The TP-Link firmware recovery page loaded.

### 6. Upload firmware and recover

- Clicked Browse and selected the `.bin` file
- Clicked Upgrade
- Waited without touching the browser or the router's power

The router flashed the firmware and rebooted on its own. The whole process took about 3 minutes.

### 7. Clean up

Once all LEDs stabilized I went back to **System Preferences > Network** and set the ethernet adapter back to DHCP. Then reconfigured my router.

