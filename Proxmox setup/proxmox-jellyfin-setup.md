# Jellyfin Media Server on Proxmox LXC with NVIDIA GPU Passthrough and Samba Share

## Overview

This documents the complete setup of a Jellyfin media server running inside an unprivileged LXC container on a bare-metal Proxmox VE host, with a dedicated 500GB media volume, NVIDIA GPU hardware transcoding passthrough, and a Samba share exposed to the local network for file transfers.

**Host:** Proxmox VE
**Container:** CT 100 — `lxc-jellyfin` (Ubuntu 22.04 LTS, 2 vCPU, 2GB RAM)
**Media volume:** `local-lvm:vm-100-disk-1` (500GB, ext4), mounted at `/mnt/media` inside the LXC
**GPU:** NVIDIA GeForce GTX 970, driver 580.159.04
**Hardware transcoding:** NVENC (H.264, HEVC, AV1)

---

## Architecture

```
LXC CT 100 (lxc-jellyfin, Ubuntu 22.04)
  ├── /mnt/media  ◄── 500GB LVM volume (vm-100-disk-1)
  ├── /dev/nvidia* ◄── GPU devices passed through via lxc.mount.entry
  ├── Jellyfin reads media library from /mnt/media
  └── Samba serves /mnt/media over the network
```

Files dropped via SMB are immediately visible to Jellyfin. Samba runs inside the LXC — not on the Proxmox host — so there is only one mount point for the media volume, avoiding fstab boot issues.<sup>1</sup>

---

## Hardware

| Component | Details                                                 |
|-----------|---------------------------------------------------------|
| CPU       | Intel Core i5-4690K @ 3.50GHz                           |
| GPU       | NVIDIA GeForce GTX 970 (4GB VRAM, Maxwell architecture) |
| RAM       | 24GB                                                    |
| Storage   | 1x 1TB HDD                                              |

---

## Important fstab Note

After installation, verify `/etc/fstab` on the Proxmox host. The root entry must use `defaults`, not `errors=remount-ro`:

```
/dev/pve/root / ext4 defaults 0 1
```

* `errors=remount-ro` is a mount option that tells the kernel to flip the filesystem to read-only if it encounters any error during boot — intended as a data safety measure, but in practice it causes `pve-cluster` to fail on startup because it can't write its lockfile, which cascades into the web UI being unreachable. 
* Using `defaults` lets the system boot normally and handle errors through standard recovery paths instead.<sup>1</sup>

---

## Part 1 — Proxmox Host: Initial Setup

### Step 1.1 — Update the system

```bash
apt update && apt upgrade -y
```

* `apt update` refreshes the local package index against the configured repositories — it doesn't install anything, it just tells apt what versions are currently available. 
* `apt upgrade -y` then installs all available updates, with `-y` auto-confirming prompts. 
* Running this first ensures build tools and headers pulled in later are the latest versions and reduces the chance of dependency conflicts.

### Step 1.2 — Download the Ubuntu 22.04 LXC template <sup>2</sup>

```
Datacenter > Proxmox > loacl(Proxmox) > CT Templates > Templates 
```
| Type    | Package                 | Version | Description                    |
|---------|-------------------------|---------|--------------------------------|
| lxc     | ubuntu-22.04-standard   | 22.04-1 | Ubuntu 22.04 Jammy (standard)  |

## Part 2 — NVIDIA Driver Installation on the Proxmox Host

### Why?

Proxmox VE uses a Linux 6.14 kernel. All NVIDIA drivers below version 580 fail to compile against this kernel due to breaking changes in internal kernel APIs.\
The Debian Trixie repository only ships driver 550, which does not support kernel 6.14.\
The solution is to install driver 580 via the `.run` installer directly from NVIDIA.<sup>3</sup>

### Step 2.1 — Install build dependencies first

```bash
apt install -y gcc make dkms pve-headers
```

* The NVIDIA `.run` installer compiles a kernel module from source at install time, so it needs a working C toolchain and the kernel headers to compile against. 
* `gcc` is the C compiler. 
* `make` is the build system. 
* `dkms` (Dynamic Kernel Module Support) registers the module so it automatically recompiles when the kernel is updated. 
* `pve-headers` provides the Proxmox-patched kernel headers that match the running PVE kernel.\
Installing these before running the installer is essential. Skipping this step causes the installer to fail immediately with a missing `cc` error.

### Step 2.2 — Download the NVIDIA 580 driver <sup>3</sup>

```bash
wget https://download.nvidia.com/XFree86/Linux-x86_64/580.159.04/NVIDIA-Linux-x86_64-580.159.04.run
```

* `wget` downloads the file from NVIDIA's direct download server. 
* The `.run` file is a self-extracting installer that bundles the kernel module source, userspace libraries, and the installer logic all in one archive.

### Step 2.3 — Make it executable and run it

```bash
chmod u+x NVIDIA-Linux-x86_64-580.159.04.run
./NVIDIA-Linux-x86_64-580.159.04.run --dkms --no-questions --ui=none
```

* Downloaded files don't have execute permission by default — `chmod u+x` grants it for the current user (`u`) only, which is enough to run the installer. 
* `--dkms` tells the installer to register the kernel module with DKMS rather than just doing a one-time compile, meaning future kernel updates won't break the driver. 
* `--no-questions` skips interactive prompts. 
* `--ui=none` disables the ncurses UI so it runs as plain text output — useful in a headless environment.

If the installer detects that Nouveau (the open-source NVIDIA driver) is still loaded, it will offer to blacklist it and rebuild the initramfs. Allow it to do both, then reboot before re-running the installer. Nouveau and the proprietary driver cannot coexist.

### Step 2.4 — Reboot and verify

```bash
reboot
nvidia-smi
```
``` 
Expected result
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 580.159.04             Driver Version: 580.159.04     CUDA Version: 13.0     |
+-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA GeForce GTX 970         Off |   00000000:01:00.0 Off |                  N/A |
| 51%   39C    P0             33W /  250W |       0MiB /   4096MiB |      0%      Default |
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

After the reboot, `nvidia-smi` (NVIDIA System Management Interface) queries the driver and GPU hardware and prints a status table. If it shows the GTX 970 with driver 580.159.04, the kernel module loaded correctly and the driver is functional.

### Step 2.5 — Note device major numbers for LXC passthrough

```bash
ls -la /dev/nvidia*
```

This lists the NVIDIA device nodes created by the driver, along with their permissions and device numbers. The output looks like:

```
crw-rw-rw- 1 root root 195,   0  /dev/nvidia0
crw-rw-rw- 1 root root 195, 255  /dev/nvidiactl
crw-rw-rw- 1 root root 234,   0  /dev/nvidia-uvm
crw-rw-rw- 1 root root 234,   1  /dev/nvidia-uvm-tools
/dev/nvidia-caps/nvidia-cap1  (major 237)
/dev/nvidia-caps/nvidia-cap2  (major 237)
```

* The numbers before and after the comma (e.g. `195, 0`) are the major and minor device numbers. 
* The major number identifies which kernel driver owns the device.
* All devices with major 195 belong to the core NVIDIA driver.
* 234 to the unified memory driver.
* 237 to the capability interface. \
These major numbers are required for the LXC cgroup rules in Part 4. Verify them before editing the container config.

---

## Part 3 — Create the Jellyfin LXC

### Step 3.1 — Create the container

```bash
pct create 100 local:vztmpl/ubuntu-22.04-standard_22.04-1_amd64.tar.zst \
  --hostname lxc-jellyfin \
  --cores 2 \
  --memory 2048 \
  --swap 512 \
  --rootfs local-lvm:20 \
  --net0 name=eth0,bridge=vmbr0,firewall=1,ip=dhcp,ip6=dhcp,type=veth \
  --ostype ubuntu \
  --features fuse=1,mount=nfs \
  --unprivileged 1
```

`pct create` provisions a new LXC container. The first argument (`100`) is the container ID — Proxmox uses this to reference the container in all subsequent commands. The template path tells Proxmox where to pull the root filesystem from. The remaining flags configure the container:

* `--hostname` sets the container's hostname, visible inside the container and in the Proxmox UI
* `--cores 2` allocates 2 CPU cores. Since NVENC handles transcoding on the GPU, the CPU load stays low and 2 cores is sufficient
* `--memory 2048` gives the container 2GB of RAM. Jellyfin's memory usage is modest with GPU transcoding
* `--swap 512` provides 512MB of swap as overflow if RAM is exhausted — acts as a safety net against OOM crashes
* `--rootfs local-lvm:20` creates a 20GB root disk on the `local-lvm` storage pool for the OS, Jellyfin, and its metadata cache. The actual media lives on a separate volume
* `--net0` attaches a virtual network interface bridged to `vmbr0` (the default Proxmox bridge connected to the physical NIC), with the Proxmox firewall enabled and DHCP for IP assignment
* `--features fuse=1,mount=nfs` unlocks kernel capabilities the container needs: `fuse` allows userspace filesystem operations required by Jellyfin, `mount=nfs` allows mounting NFS shares if needed later
* `--unprivileged 1` runs the container in unprivileged mode, where the container's root user maps to an unprivileged user on the host — better isolation than privileged containers

### Step 3.2 — Enable nesting

```bash
pct set 100 --features fuse=1,mount=nfs,nesting=1
```

* `pct set` modifies a container's configuration after creation. 
* `nesting=1` enables Linux namespace nesting, which allows the container to run its own instance of systemd properly.\
Without it, systemd detects it is inside a container and restricts certain operations, causing services like Jellyfin to fail to start or stop cleanly.\
This is added separately because Proxmox may warn about it at container creation time.

### Step 3.3 — Set a static IP

Rather than hardcoding a static IP in the container config, I assigned a static DHCP lease in my router using the container's MAC address. This keeps IP management in one place  rather than split across individual container configs.

Find the container's MAC address with:

```bash
pct config 100 | grep net0
```

* `pct config` prints the full configuration of a container. 
*  Piping through `grep net0` filters to just the network interface line, which contains the `hwaddr` (MAC address) field.

---

## Part 4 — GPU Passthrough Configuration

### Step 4.1 — Navigate to the LXC config directory

```bash
cd /etc/pve/lxc
```

Proxmox stores each container's configuration as a plain text file at `/etc/pve/lxc/<id>.conf`. This directory is part of the Proxmox cluster filesystem (`pmxcfs`), which is why it persists across reboots and is visible in the web UI.

### Step 4.2 — Edit the container config

```bash
nano 100.conf
```

Using the major device numbers confirmed in Step 2.5, I added the following lines at the bottom of the file:

```
lxc.cgroup2.devices.allow: c 195:* rwm
lxc.cgroup2.devices.allow: c 234:* rwm
lxc.cgroup2.devices.allow: c 237:* rwm
lxc.mount.entry: /dev/nvidia0 dev/nvidia0 none bind,optional,create=file
lxc.mount.entry: /dev/nvidiactl dev/nvidiactl none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-uvm dev/nvidia-uvm none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-uvm-tools dev/nvidia-uvm-tools none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-caps/nvidia-cap1 dev/nvidia-caps/nvidia-cap1 none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-caps/nvidia-cap2 dev/nvidia-caps/nvidia-cap2 none bind,optional,create=file
```

**What these lines do:**

* The `lxc.cgroup2.devices.allow` lines grant the container permission to access character devices (`c`) with the specified major numbers.\
  *By default, LXC containers are denied access to all host devices through cgroup device controls — these rules punch specific holes for the NVIDIA devices.*
* The `rwm` flags allow read, write, and mknod operations on those devices.


* The `lxc.mount.entry` lines bind-mount each NVIDIA device node from the host into the container at the same relative path. 
* `bind` means it's a direct mirror of the host device rather than a copy. `optional` prevents the container from failing to start if a device node doesn't exist (e.g. if `nvidia-caps` isn't present on some systems). `create=file` tells LXC to create an empty file at the destination path inside the container if one doesn't already exist, so the bind mount has somewhere to attach to.

Together, the cgroup rules grant permission and the mount entries make the devices visible — both are required for GPU passthrough to work.

---

## Part 5 — Media Volume
*I currently do not have a NAS, so I have chosen to allocate 500GB of the HDD to store a copy of my media, and 
I am not following the 3-2-1 rule as I do not have an offsite location, but I now have three total copies.*
### Step 5.1 — Allocate the volume

```bash
pvesm alloc local-lvm 100 vm-100-disk-1 500G
```

`pvesm alloc` creates a new logical volume in a Proxmox storage pool. `local-lvm` is the target storage pool (backed by LVM thin provisioning on the host), `100` is the container ID this volume is associated with, `vm-100-disk-1` is the volume name (following Proxmox's naming convention), and `500G` is the size. The volume is thin-provisioned, meaning it doesn't immediately consume 500GB of physical disk — it grows as data is written.

### Step 5.2 — Format the volume

```bash
mkfs.ext4 /dev/pve/vm-100-disk-1
```

`mkfs.ext4` writes an ext4 filesystem onto the raw block device. This must be done before attaching the volume to the container — if skipped, run `pct set` on an unformatted volume, Proxmox will probe the block device, find no recognizable filesystem, and exit with error code 32. Formatting creates the inode table, journal, and superblock that the OS needs to mount and use the volume.

### Step 5.3 — Attach the volume to the LXC

```bash
pct set 100 -mp0 /dev/pve/vm-100-disk-1,mp=/mnt/media
```

`-mp0` defines mount point 0 for the container. The argument specifies the host-side block device path (`/dev/pve/vm-100-disk-1`) and the path inside the container where it should be mounted (`mp=/mnt/media`). Proxmox handles the actual mounting when the container starts. fstab entry on the host is not needed.

### Step 5.4 — Verify the config

```bash
cat /etc/pve/lxc/100.conf
```

> **Note on `lost+found`:** After formatting, ext4 automatically creates a `lost+found` directory at the root of the volume. This is used by `fsck` — if the filesystem checker finds orphaned inodes or file fragments during a repair, it deposits them here rather than deleting them outright. It has no impact on Jellyfin or Samba and can be ignored.

---

## Part 6 — Jellyfin Installation (inside LXC)

### Step 6.1 — Start and enter the container

```bash
pct start 100
pct enter 100
```

`pct start` boots the container. `pct enter` opens an interactive shell session inside it by attaching to the container's init process. The prompt will change from `root@Proxmox` to `root@lxc-jellyfin`.
### Step 6.2 — Install Jellyfin [^4]

```bash
apt update && apt install -y curl
curl -s https://repo.jellyfin.org/install-debuntu.sh | bash
```

* First installs `curl` if not already present, since the container template is minimal. 
* The second command downloads the official Jellyfin install script and pipes it directly into `bash` for execution. 
* The `-s` flag runs curl in silent mode, suppressing download progress output. 
* The script auto-detects the Ubuntu 22.04 (Jammy) release, adds the official Jellyfin apt repository, installs both `jellyfin` and `jellyfin-ffmpeg` (the bundled FFmpeg build with hardware acceleration support), and starts the systemd service.

### Step 6.3 — Verify Jellyfin is running

```bash
systemctl status jellyfin
```

Queries systemd for the current state of the `jellyfin` service unit. You want to see `active (running)`. 
If it shows `failed` or `activating`, check `journalctl -u jellyfin` for error details.

### Step 6.4 — Verify NVENC encoders are available

```bash
/usr/lib/jellyfin-ffmpeg/ffmpeg -encoders 2>/dev/null | grep nvenc
```

Runs Jellyfin's bundled FFmpeg binary with `-encoders` to list all compiled-in encoders, redirects stderr to `/dev/null` to suppress noise, then filters the output for `nvenc`. If the GPU passthrough is working, it will show `h264_nvenc`, `hevc_nvenc`, and `av1_nvenc` in the output. If this comes back empty, the GPU devices are not accessible inside the container and the passthrough config needs to be checked.

Expected output:
```
V....D av1_nvenc            NVIDIA NVENC av1 encoder (codec av1)
V....D h264_nvenc           NVIDIA NVENC H.264 encoder (codec h264)
V....D hevc_nvenc           NVIDIA NVENC hevc encoder (codec hevc)
```

### Step 6.5 — Install NVIDIA userspace utilities

```bash
apt install -y nvidia-utils-580 libnvidia-encode-580
```

The NVIDIA kernel module is provided by the host driver, but the userspace libraries that applications use to talk to it need to be present inside the container separately. `nvidia-utils-580` provides `nvidia-smi` and related tools, while `libnvidia-encode-580` provides the NVENC encoding library that Jellyfin's FFmpeg links against at runtime.

> **Note on version mismatch:** The host driver installed via `.run` is `580.159.04`, while Ubuntu's packaged `nvidia-utils-580` is `580.159.03`. This is a packaging revision difference only — the driver ABI is identical between the two. `nvidia-smi` inside the container will report a "Driver/library version mismatch" warning, but NVENC transcoding functions correctly and the warning can be ignored.

---

## Part 7 — Samba Share Setup (inside LXC)

Samba is installed inside the LXC rather than on the Proxmox host. This means the media volume has only one mount point — inside the container via `mp0` — and requires no fstab entry on the host. Previously having Samba on the host required a separate host-side fstab mount of the same volume, which created a boot-time race condition between fstab and LVM activation.<sup>1</sup>

### Step 7.1 — Install Samba

```bash
apt install -y samba
```

Installs the Samba suite, which includes `smbd` (the SMB file sharing daemon), `nmbd` (the NetBIOS name service daemon for network discovery), and supporting libraries. After installation, a default `smb.conf` is created at `/etc/samba/smb.conf`.

### Step 7.2 — Configure the share

```bash
nano /etc/samba/smb.conf
```

Open the config file and scroll to the bottom to add the share definition. The existing content (global settings, printer shares) can be left as-is.

```ini
[media]
   path = /mnt/media
   browseable = yes
   writable = yes
   valid users = mediauser
   create mask = 0777
   directory mask = 0777
   force user = root
   force group = root
   vfs objects = catia fruit streams_xattr
   fruit:metadata = stream
   fruit:model = MacSamba
   fruit:posix_rename = yes
   fruit:veto_appledouble = yes
   fruit:wipe_intentionally_left_blank_rfork = yes
   fruit:delete_empty_adfiles = yes
```

**What each option does:**

- `[media]` — the share name. This is what clients connect to.
- `path` — the filesystem path being shared. Points to the 500GB volume mount inside the LXC
- `browseable` — controls whether the share appears when a client browses the server. If set to `no`, clients must know the share name and type it manually
- `writable` — allows clients to write files to the share. Without this, the share is read-only
- `valid users` — restricts connections to the listed accounts. Only `mediauser` can authenticate; prevents anonymous or guest access
- `create mask / directory mask` — permission bits applied to newly created files and directories. `0777` gives full read/write/execute to owner, group, and others, ensuring Jellyfin can read anything written via SMB regardless of which user wrote it
- `force user / force group` — overrides the effective user and group for all filesystem operations to `root`, regardless of who authenticated. This prevents permission mismatches between files written by Samba and files read by Jellyfin
- `vfs objects = catia fruit streams_xattr` — loads three VFS (Virtual Filesystem) plugin modules: `fruit` handles macOS-specific SMB extensions, `catia` translates characters that are legal in macOS filenames but illegal in Linux filenames, and `streams_xattr` stores NTFS alternate data streams as extended attributes on the Linux filesystem
- `fruit:metadata = stream` — stores macOS extended metadata (resource forks, Finder info) as data streams rather than separate `._` files
- `fruit:model = MacSamba` — advertises the server as a Mac in the Bonjour/mDNS service record, giving it a Mac icon in Finder's sidebar
- `fruit:posix_rename = yes` — enables atomic POSIX rename semantics for macOS clients, allowing Finder to rename files properly without intermediate errors
- `fruit:veto_appledouble = yes` — prevents macOS from creating `._filename` AppleDouble resource fork files, which would otherwise appear in the share and confuse non-Apple clients like Jellyfin and smart TVs
- `fruit:wipe_intentionally_left_blank_rfork = yes` — automatically removes resource fork entries that macOS creates but leaves empty
- `fruit:delete_empty_adfiles = yes` — deletes any empty AppleDouble sidecar files that slip through

### Step 7.3 — Create a Samba user

```bash
useradd -M -s /sbin/nologin mediauser
smbpasswd -a mediauser
```

* `useradd -M` creates a system user without a home directory (`-M`).
* `-s /sbin/nologin` sets the login shell to a no-op binary that immediately exits — this user exists only for Samba authentication and cannot be used to log into the system interactively.
* `smbpasswd -a` adds the user to Samba's separate password database (Samba maintains its own credential store independent of `/etc/shadow`) and prompts you to set a password.

### Step 7.4 — Start and enable Samba

```bash
systemctl restart smbd nmbd
systemctl enable smbd nmbd
```

* `systemctl restart` stops and restarts both daemons to pick up the new config. 
* `smbd` handles actual file sharing over TCP port 445, while `nmbd` handles NetBIOS name resolution and network browsing over UDP 137/138 — this is what makes the server discoverable by name on the local network. 
* `systemctl enable` creates the systemd symlinks so both services start automatically when the container boots.

### Step 7.5 — Verify

```bash
cat /etc/samba/smb.conf
```

Prints the full config file to confirm the `[media]` block was written correctly.

---

## Part 8 — Configure Jellyfin Hardware Transcoding

1. Open Jellyfin.
2. Complete the initial setup wizard, setting your media library path to `/mnt/media/Movies` (or the appropriate subfolder — point to the folder containing the actual media files, not the root of the volume)
3. Go to **Dashboard > Playback > Transcoding**
4. Set **Hardware acceleration** to `Nvidia NVENC`
5. Under **Enable hardware decoding for**: enable H.264, HEVC, AV1
6. Click **Save**

With NVENC selected, Jellyfin offloads encode and decode operations to the GTX 970's dedicated hardware video engine rather than using the CPU.

---

## Part 9 — NVENC Stream Limit Patch (Unresolved)

NVIDIA artificially limits consumer GPUs to a small number of simultaneous NVENC encode sessions in the driver. The keylase patch removes this restriction by patching the relevant check out of the driver binary.<sup>5</sup>

**Current status:** The keylase patch does not yet support driver version `580.159.04`. Version `580.159.03` is listed as supported, but the script reads the running driver version via `nvidia-smi` and the NVML version mismatch inside the container prevents detection. Forcing the version with `VERSION=580.159.03 bash ./patch.sh` was also unsuccessful as the script ignores the override when `nvidia-smi` fails.

Since this is a single-user setup, the default stream limit is not a practical constraint. Revisit when the patch adds support for `580.159.04`.

To attempt the patch in future:
```bash
cd /opt
wget https://raw.githubusercontent.com/keylase/nvidia-patch/master/patch.sh
bash ./patch.sh
```

---

## Troubleshooting Reference

| Symptom                                                     | Cause                                                           | Fix                                                                                             |
|-------------------------------------------------------------|-----------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Proxmox boots to emergency mode, `vm-100-disk-1` timeout    | Thin LV not activated before fstab mount                        | Add `x-systemd.requires=lvm2-activation.service` to fstab entry                                 |
| `pve-cluster` fails, web UI unreachable after boot          | Root filesystem remounted read-only                             | `mount -o remount,rw /` then `systemctl start pve-cluster`                                      |
| Root filesystem keeps going read-only on every boot         | `errors=remount-ro` in fstab triggering on boot hiccups         | Change root fstab entry from `errors=remount-ro` to `defaults`                                  |
| CT 100 fails to start after reboot                          | Thin LV not activated by boot time                              | `lvchange -ay pve/vm-100-disk-1` then `pct start 100`                                           |
| NVIDIA 550 driver fails to compile                          | Linux 6.14 kernel API incompatibility                           | Use NVIDIA 580+ driver from `.run` installer — do not use apt                                   |
| `pct set` mount fails with exit code 32                     | No filesystem on volume                                         | Run `mkfs.ext4` on the block device before running `pct set`                                    |
| `nvidia-smi` in LXC shows version mismatch                  | Host `.run` driver (580.159.04) vs LXC apt package (580.159.03) | Safe to ignore — ABI is identical, transcoding works correctly                                  |
| Jellyfin library shows no movies after scan                 | Library path points to volume root instead of subfolder         | Set library path to `/mnt/media/Movies`, not `/mnt/media`                                       |
| macOS can't connect to SMB share                            | Guest SMB blocked by macOS Ventura/Sonoma                       | Connect with `mediauser` credentials                                                            |
| Files copied from Mac cause errors on other clients         | macOS `._` AppleDouble resource fork files polluting the share  | `fruit` VFS module prevents new ones; clean existing with `find /mnt/media -name "._*" -delete` |
| keylase patch fails: "Can not detect nvidia driver version" | NVML mismatch blocks `nvidia-smi` inside the container          | Patch unsupported for 580.159.04 — revisit when patch is updated                                |

---

## References

<sup>1</sup>: Root cause analysis documented from a multi-session recovery. The original setup had Samba on the Proxmox host with `/dev/pve/vm-101-disk-1` mounted via fstab at `/mnt/lxc-media`. Without `x-systemd.requires=lvm2-activation.service` in the fstab entry, systemd attempted the mount before LVM activated the thin volume, causing `local-fs.target` to fail and dropping the host into emergency mode. Additionally, `errors=remount-ro` on the root partition caused every unclean shutdown to remount root read-only on next boot, breaking `pve-cluster`. Moving Samba into the LXC eliminates the host-side fstab entry entirely.

<sup>2</sup>: Ubuntu 22.04 LXC template: http://download.proxmox.com/images/system/ubuntu-22.04-standard_22.04-1_amd64.tar.zst

<sup>3</sup>: NVIDIA driver download page: https://www.nvidia.com/en-us/drivers/ — Driver 580.159.04 direct link: https://download.nvidia.com/XFree86/Linux-x86_64/580.159.04/NVIDIA-Linux-x86_64-580.159.04.run

<sup>4</sup>: Jellyfin official installation script for Debian/Ubuntu: https://jellyfin.org/downloads/linux

<sup>5</sup>: keylase NVENC stream limit patch: https://github.com/keylase/nvidia-patch/blob/master/patch.sh

<sup>6</sup>: https://forum.proxmox.com/threads/2025-proxmox-pcie-gpu-passthrough-with-nvidia.169543/

<sup>7</sup>: https://docs.nvidia.com/deploy/nvidia-smi/

<sup>8</sup>: https://www.reddit.com/r/unRAID/comments/dm2phn/unraid_680rc1_wnvidia_remove_2_stream_limit/

<sup>9</sup>: https://www.youtube.com/watch?v=7mzNr_rfk4k&t=527s

<sup>10</sup>: https://www.youtube.com/watch?v=gHBSrENzeqk&t=430s