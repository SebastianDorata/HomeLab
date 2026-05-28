Open WebUI and Ollama Installation
---

Install & create the Open WebUI LXC from Proxmox VE Scripts<sup>8</sup>

| CPU                           | GPU                    | Ram       | Storage |
|-------------------------------|------------------------|-----------|---------|
| Intel Core i5-4690K @ 3.50GHz | NVIDIA GeForce GTX 970 | 18432 MiB | 64 GB   |
| 4 Cores                       | 4GB VRAM               |           |         |

```bash
mode=generated var_ram="18432" var_disk="64" bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/ct/openwebui.sh)"
```
Installation Output:
```
   ____                      _       __     __    __  ______
  / __ \____  ___  ____     | |     / /__  / /_  / / / /  _/
 / / / / __ \/ _ \/ __ \    | | /| / / _ \/ __ \/ / / // /  
/ /_/ / /_/ /  __/ / / /    | |/ |/ /  __/ /_/ / /_/ // /   
\____/ .___/\___/_/ /_/     |__/|__/\___/_.___/\____/___/   
    /_/                                                     
  ⚙️  Using Generated Settings on node Proxmox
  💡  PVE Version 9.2.2 (Kernel: 7.0.2-6-pve)
  🆔  Container ID: 101
  🖥️  Operating System: debian (13)
  📦  Container Type: Unprivileged
  💾  Disk Size: 64 GB
  🧠  CPU Cores: 4
  🛠️  RAM Size: 18432 MiB
  🎮  GPU Passthrough: Enabled
  🚀  Creating an Open WebUI LXC using the above default settings
  
  ✔️   Storage local (Free: 82.6GB  Used: 6.6GB) [Template]
  ✔️   Storage local-lvm (Free: 430.5GB  Used: 363.9GB) [Container]
  ✔️   Storage 'local-lvm' (lvmthin) validated
  ✔️   Template storage 'local' validated
  ✔️   Template search completed
  ✔️   Template debian-13-standard_13.1-2_amd64.tar.zst [online]
  ✔️   Template download successful.
  ✔️   LXC Container 101 was successfully created.
 🎮 Detected NVIDIA GPU
 🎮 Found 6 NVIDIA device(s) for passthrough
  ✔️   Automatically configuring NVIDIA GPU passthrough
  ✔️   NVIDIA GPU passthrough configured (6 devices) - install drivers in container if needed
  ✔️   Started LXC Container
  ✔️   Network in LXC is reachable (ping)
  ✔️   GPU passthrough configured (video:44, render:992)
  ✔️   Customized LXC Container
  ✔️   Set up Container OS
  ✔️   Network Connected: (IP address) 
  ✔️   IPv4 Internet Connected
  ✖️   IPv6 Internet Not Connected
  ✔️   Git DNS: github.com:(✔️ ) raw.githubusercontent.com:(✔️ ) api.github.com:(✔️ ) git.community-scripts.org:(✔️ )
  ✔️   Updated Container OS
  ✔️   Installed Dependencies
  ✔️   Detected GPU: NVIDIA Corporation GM204 [GeForce GTX 970] (rev a1) (NVIDIA)

 🎮 NVIDIA GPU passthrough detected
  💡   No NVIDIA packages for version 580.x found in repos
  ✔️   NVIDIA GPU configured
  💡   nvidia-smi test failed - check driver version match
  ✔️   Setup Hardware Acceleration
  ✔️   Python 3.12 installed
  ✔️   Setup uv 0.11.16
  ✔️   Installed Open WebUI
      Would you like to add Ollama? <y/N> y
  ✔️   Set up Intel® Repositories
  ✔️   Installed Intel® Level Zero
  ✔️   Installed Intel® oneAPI Base Toolkit
  ✔️   Deployed: ollama-com (0.24.0)
  ✔️   Installed Ollama
  ✔️   Created Service
  ✔️   Customized Container
  ✔️   Cleaned
  ✔️   Completed successfully!

  🚀  Open WebUI setup has been successfully initialized!
  💡   Access it using the following URL:
    🌐  http://(IP address):8080
root@Proxmox:~# 
```
## Issue 1:
After creating an admin account I downloaded `CognitiveComputations/dolphin-llama3.1:latest    ed9503dedda9    4.7 GB`<sup>5</sup>
```
https://ollama.com/CognitiveComputations/dolphin-llama3.1
```
After running a test I was met with the error that there was not enough memory.\
To test if it would run under different conditions, I increased the swap to 5GB. It did run, but responses were generated slowly.\
To check GPU usage, I ran:

```bash
watch -n 1 nvidia-smi
```
And was met with
```
Every 1.0s: nvidia-smi                                                                                                                                                                                                                                                                                openwebui: Wed May 27 23:25:57 2026
Failed to initialize NVML: Driver/library version mismatch
NVML library version: 550.163
```
Which was confirmed by the installation output.


Because of the installation issues
`No NVIDIA packages for version 580.x found in repos`
and 
`nvidia-smi test failed - check driver version match`
 I followed the same steps from my Jellyfin mismatch documentation<sup>1</sup>

### 1. Push the .run installer into CT 101 and reinstall userspace-only:
### On the host:
```bash
pct push 101 NVIDIA-Linux-x86_64-580.159.04.run /root/NVIDIA-Linux-x86_64-580.159.04.run
```
### Inside the LXC:

```bash
# Remove old packages
apt remove --purge "*nvidia*" "*cuda*"
apt autoremove

# Run the installer userspace-only
bash /root/NVIDIA-Linux-x86_64-580.159.04.run --no-kernel-module --no-questions --ui=none

ldconfig
```

```bash
# To verify
nvidia-smi
```
Output:
```
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 580.159.04             Driver Version: 580.159.04     CUDA Version: 13.0     |
+-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA GeForce GTX 970         Off |   00000000:01:00.0 Off |                  N/A |
| 49%   37C    P0             33W /  250W |       0MiB /   4096MiB |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI              PID   Type   Process name                        GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|  No running processes found                                                             |
+-----------------------------------------------------------------------------------------+
```
Lastly, I restarted Ollama
```bash
systemctl restart ollama
```
I ran the same prompt again and monitored the GPU usage with:
```bash
watch -n 1 nvidia-smi
```
I saw the GPU % jump to 11% before I was met with `llama runner process has terminated: signal arrived during cgo execution`.

### This confirms the process loaded onto the 970 but I had to check if it was a VRAM issue or a CUDA context issue.

I first ran:
```bash
journalctl -u ollama -n 50 | grep -i "vram\|gpu\|memory\|error\|fatal\|signal"
```
And the output was
```
May 27 23:30:25 openwebui ollama[820]: time=2026-05-27T23:30:25.892-05:00 level=ERROR source=server.go:316 msg="llama runner terminated" error="exit status 2"
May 27 23:30:25 openwebui ollama[820]: time=2026-05-27T23:30:25.909-05:00 level=INFO source=sched.go:511 msg="Load failed" model=/root/.ollama/models/blobs/sha256-c4e04968e3ca697b947c4820d7d4e58873e9f93908a043e7280863b31019b7df 
    error="llama runner process has terminated: signal arrived during cgo execution"
```

## Root cause:
The model is 700MB too large to fit. It loads partially then runs out of memory and crashes.

## Next Steps: Find a model that fits within 4 GB of VRAM.

I downloaded `gemma4:e2b`<sup>2</sup> and `gemma4:e4b`<sup>3</sup> to observe how my system would handle them.
```bash
# Download gemma4:e2b
ollama pull gemma4:e2b

# Download gemma4:e4b
ollama pull gemma4:e4b
```
I gave them both the same prompt:
```
Write me a detailed 500 word essay on how the internet works, covering TCP/IP, DNS, routing, and physical infrastructure.
```
And again observed them both with:
```bash
# On Proxmox host
watch -n 1 nvidia-smi
```

- With `gemma4:e2b`, it thought for 23 seconds and GPU usage never exceeded 60% while generating.<sup>6</sup>
- With `gemma4:e4b`, it thought for 46 seconds and GPU usage also never exceeded 60% while generating.<sup>7</sup>
- Both reached 95%+ GPU utilization while suggesting followup questions.

```
gemma4:e2b
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 580.159.04             Driver Version: 580.159.04     CUDA Version: 13.0     |
+-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA GeForce GTX 970         Off |   00000000:01:00.0 Off |                  N/A |
| 56%   56C    P2            122W /  250W |    3605MiB /   4096MiB |     49%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI              PID   Type   Process name                        GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|    0   N/A  N/A           19742      C   /usr/lib/ollama/bin/ollama             3601MiB |
+-----------------------------------------------------------------------------------------+
```
```
gemma4:e4b
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 580.159.04             Driver Version: 580.159.04     CUDA Version: 13.0     |
+-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA GeForce GTX 970         Off |   00000000:01:00.0 Off |                  N/A |
| 57%   57C    P2            132W /  250W |    3605MiB /   4096MiB |     54%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI              PID   Type   Process name                        GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|    0   N/A  N/A           19742      C   /usr/lib/ollama/bin/ollama             3601MiB |
+-----------------------------------------------------------------------------------------+

```


# Secondary tasks:

With three models installed I wanted to check the remaining disk space allocated to the LXC.
```bash
free -h && echo "---" && nproc && echo "---" && nvidia-smi --query-gpu=name,memory.total,memory.free --format=csv,noheader && echo "---" && df -h / && echo "---" && cat /proc/cpuinfo | grep "model name" | head -1
```
Output:
```
total        used        free      shared  buff/cache   available
Mem:            18Gi       952Mi        12Gi        92Ki       4.5Gi        17Gi
Swap:          4.9Gi          0B       4.9Gi
---
4
---
NVIDIA GeForce GTX 970, 4096 MiB, 4027 MiB
---
Filesystem                        Size  Used Avail Use% Mounted on
/dev/mapper/pve-vm--101--disk--0   63G   30G   30G  50% /
---
model name      : Intel(R) Core(TM) i5-4690K CPU @ 3.50GHz
```

With 50% of the storage remaining, I checked the largest directories to see where the 30GB had gone.
```bash
du -h --max-depth=1 / 2>/dev/null | sort -rh | head -20
```
Output:
```
30G     /
15G     /opt
8.4G    /root
6.6G    /usr
295M    /var
3.5M    /etc
80K     /run
16K     /lost+found
4.0K    /tmp
4.0K    /srv
4.0K    /proc
4.0K    /mnt
4.0K    /media
4.0K    /home
4.0K    /dev
4.0K    /boot
0       /sys
```
And then to probe further:
```bash
du -h --max-depth=1 /opt 2>/dev/null | sort -rh | head -10
```
Output:
```
15G     /opt/intel
15G     /opt
```
```bash
du -h --max-depth=1 /root 2>/dev/null | sort -rh | head -10
```
Output:
```
8.4G    /root
7.1G    /root/.local
889M    /root/.cache
1.6M    /root/.open-webui
32K     /root/.ollama
8.0K    /root/.nv
4.0K    /root/.ssh
```
Since I am using an NVIDIA GPU, none of the Intel oneAPI libraries will be used. I removed them with:

```bash
rm -rf /opt/intel

# And to clean the cache
rm -rf /root/.cache/pip /root/.cache/uv
```

```bash
# lastly, to verify:
df -h /
```
Output:
```
Filesystem                        Size  Used Avail Use% Mounted on
/dev/mapper/pve-vm--101--disk--0   63G   16G   45G  26% /
```
---
# Potential future plans: 

I still have `dolphin-llama3` installed, but should I choose to remove it, I will run:
```bash
ollama rm CognitiveComputations/dolphin-llama3.1:latest
```

If I choose to force `dolphin-llama3` to run on the CPU so that I may use it, I will run:
```bash
OLLAMA_NUM_GPU=0 ollama serve
```
---
While checking GPU usage, I noticed `0  NVIDIA GeForce GTX 970 Off` which refers to `Persistence Mode`<sup>4</sup>.\
Persistence mode keeps the NVIDIA driver loaded in memory between jobs so there's no initialization delay.\
Should I choose to enable it in the future, I will run:
```bash
# On Proxmox host
nvidia-smi -pm 1
```
---

## References
* https://github.com/SebastianDorata/HomeLab/blob/main/Proxmox%20setup/Jellyfin/Troubleshooting/GPU_Passthrough_and_Hardware_Transcoding_Failure.md<sup>1</sup>
* https://ollama.com/library/gemma4:e2b<sup>2</sup>
* https://ollama.com/library/gemma4:e4b<sup>3</sup>
* https://docs.nvidia.com/deploy/driver-persistence/persistence-daemon.html<sup>4</sup>
* https://erichartford.com/uncensored-models<sup>5</sup>
* https://community-scripts.org/generator?script=openwebui<sup>8</sup>

---
# Prompt Generated Responses

## The Invisible Architecture: How the Internet Works — `gemma4:e2b`<sup>6</sup>

The internet, often perceived as an ethereal space of instant communication, is in reality a vast, intricate system built upon layers of physical reality and abstract protocols. Understanding how the internet functions requires examining the interplay between physical infrastructure, sophisticated addressing systems, and essential communication protocols like TCP/IP, DNS, and routing.

The foundation of the entire system is the **physical infrastructure**. This consists of the tangible hardware—fiber optic cables, copper wires, satellites, and the routers and servers that connect them. These physical lines act as the highways upon which data travels. Data is physically broken down into small, manageable units called packets, which are then transmitted across these wires.

To ensure these packets can travel reliably, a standard communication framework is required: **TCP/IP (Transmission Control Protocol/Internet Protocol)**. TCP/IP is the language of the internet. IP handles the addressing, assigning a unique numerical address (the IP address) to every device, much like a postal address. TCP ensures that the data sent is error-free and that all packets arrive in the correct order. Without TCP/IP, data would be unintelligible noise.

However, human users interact with names, not numbers. This is where the **Domain Name System (DNS)** steps in. When a user types a domain name (e.g., www.google.com), DNS acts as the internet’s massive phonebook, translating that easily memorable name into the corresponding numerical IP address that routers need to locate the destination. This translation is crucial for making the complex addressing system accessible to people.

Once the destination IP is known, the data must find the optimal path. This is the role of **routing**. Routers are the intelligent devices that make real-time decisions about packet forwarding. They analyze the destination IP address and determine the most efficient path—the sequence of network links—for the packet to travel across various interconnected networks to reach its final destination. This dynamic routing allows data to navigate complex geographical and network topography instantaneously.

In essence, the internet operates as a seamless choreography: Physical infrastructure provides the roads; TCP/IP defines the rules for traffic; DNS provides the addresses; and routing guides the packets along the optimal route. Together, these layers transform raw electrical signals into the instantaneous, global communication we rely on every moment.

***

## The Invisible Architecture: How the Internet Works — `gemma4:e4b`<sup>7</sup>

The internet, in its modern form, feels like a magical ether—an omnipresent repository of information accessible through a flick of a switch. Yet, beneath the seamless facade lies a deeply intricate, robust, and highly engineered architecture. Understanding how the internet works requires moving past the superficial idea of a single network and recognizing it instead as a sprawling, interconnected web of systems, protocols, and physical infrastructures.

At the foundational level is the **physical infrastructure**. The data we stream, download, and transmit must travel through something tangible. This plumbing consists primarily of massive undersea fiber optic cables, which connect continents, alongside terrestrial cable networks and countless local fiber lines. These cables transmit data as pulses of light. The signal moves from an end-user device, through a local access point, and out onto major network backbones — the backbone of global connectivity.

However, a physical cable only provides a pathway; it doesn't provide the addresses or the language. This is where the protocols begin.

For a human to interact with a website, they need a memorable name (e.g., google.com). But computers communicate only through numbers—the **IP address**. The gap between the two is bridged by the **Domain Name System (DNS)**. DNS acts as the internet's global phonebook. When you type a URL, your device first sends a query to a DNS server. This server then translates the human-readable domain name into its corresponding, numerical IP address, allowing the data journey to begin.

Once the destination (the IP address) is known, the data must be efficiently packaged and delivered. This is the job of the **TCP/IP protocol suite**. IP (Internet Protocol) is responsible for the *addressing*—it ensures that every packet of data is properly labeled with the sender’s and receiver’s addresses.

Meanwhile, TCP (Transmission Control Protocol) is responsible for the *reliability*. Since large files are broken down into smaller units called packets, TCP ensures that all these packets are sequenced correctly, that none are lost, and that if a packet is damaged, it is re-requested and resent. TCP establishes the foundational, reliable "conversation" layer that allows continuous streaming and guaranteed file transfer.

The packets, now addressed and packaged, must travel across potentially thousands of miles and through dozens of different corporate and governmental networks. This requires the role of the **router**. Routers are essentially highly sophisticated traffic cops. They examine the destination IP address of every incoming packet and consult constantly updated tables to determine the most efficient path—a process known as **routing**.

These routes are governed by complex rules established by protocols like Border Gateway Protocol (BGP). BGP allows vast, independent networks (called Autonomous Systems or AS) to negotiate and exchange path information. A packet might jump from a local ISP’s network to a continental carrier’s network, and then across a transatlantic cable, with each major handoff determined by a router making a split-second, algorithmic decision about the fastest, most stable path.

In synthesis, the internet is not a singular machine, but a stunningly complex collaboration. Physical cables provide the medium; DNS provides the translation; IP provides the addressing; TCP provides the guaranteed delivery; and routers, guided by BGP, provide the navigational intelligence. It is this layering of physical engineering and abstract protocol design that allows billions of devices to communicate instantaneously, making the internet one of humanity’s most profound feats of collective engineering.
