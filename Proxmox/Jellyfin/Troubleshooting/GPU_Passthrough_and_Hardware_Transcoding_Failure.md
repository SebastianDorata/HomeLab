# Troubleshooting: NVIDIA GPU Passthrough and Hardware Transcoding Failure

This document covers five issues encountered after a clean Proxmox reinstall that prevented Jellyfin from using the NVIDIA GTX 970 for hardware transcoding: an NVML version mismatch between the host driver and the LXC userspace libraries, incorrect device node permissions on the GPU character devices, FFmpeg exiting with code 187 on every transcode attempt, the `nvidia-uvm` module not loading on boot leaving device nodes absent after a reboot, and the `nvidia-uvm` major device number changing between reboots causing the LXC cgroup rule to deny access.

---

## Issue 1: NVML Version Mismatch in LXC

### Symptom

Running `nvidia-smi` inside the LXC produces the following error:

```
Failed to initialize NVML: Driver/library version mismatch
NVML library version: 580.159
```

### Root Cause

The Proxmox host installed the NVIDIA driver via the `.run` installer directly from NVIDIA, which produced driver version `580.159.04`. The LXC was running Ubuntu 22.04 and had the NVIDIA userspace libraries installed via `apt` from Ubuntu's package repository, which only carries version `580.159.03`. The LXC shares the host kernel and therefore uses the host's kernel module — when the userspace libraries inside the container report a different version string than the loaded kernel module, NVML refuses to initialize.

### Diagnosis

#### Step 1: Check the host kernel module version

```bash
cat /proc/driver/nvidia/version
```

Output:
```
NVRM version: NVIDIA UNIX x86_64 Kernel Module  580.159.04  Wed Apr 29 17:32:45 UTC 2026
GCC version:  gcc version 14.2.0 (Debian 14.2.0-19)
```

#### Step 2: Check what NVIDIA packages are installed in the LXC

```bash
dpkg -l | grep -i nvidia
```

Output:
```
ii  libnvidia-compute-580:amd64    580.159.03-0ubuntu0.22.04.1
ii  libnvidia-decode-580:amd64     580.159.03-0ubuntu0.22.04.1
ii  libnvidia-encode-580:amd64     580.159.03-0ubuntu0.22.04.1
ii  nvidia-firmware-580-580.159.03 580.159.03-0ubuntu0.22.04.1
ii  nvidia-kernel-common-580       580.159.03-0ubuntu0.22.04.1
ii  nvidia-utils-580               580.159.03-0ubuntu0.22.04.1
```

The `.03` suffix in the LXC packages against the `.04` host kernel module confirms the mismatch.

#### Step 3: Confirm the host used the .run installer

```bash
dpkg -l | grep nvidia
```

Output on the host:
```
ii  pve-nvidia-vgpu-helper  0.3.1  all  Proxmox Nvidia vGPU helper script and systemd service
```

No userspace NVIDIA packages are present on the host — the driver was installed entirely via the `.run` installer, which places libraries outside of apt's control. Ubuntu's apt repository does not carry the `.04` build, so the LXC packages can never match the host via `apt` alone.

---

### Fix: Replace LXC apt packages with the .run installer (userspace only)

#### Step 1: Push the .run installer into the LXC from the Proxmox host

```bash
pct push 100 NVIDIA-Linux-x86_64-580.159.04.run /root/NVIDIA-Linux-x86_64-580.159.04.run
```

* `pct push` copies a file from the Proxmox host filesystem into the specified container. The first path is the source on the host and the second is the destination inside the container.
* If the `.run` file is no longer present on the host, re-download it first:

```bash
wget https://download.nvidia.com/XFree86/Linux-x86_64/580.159.04/NVIDIA-Linux-x86_64-580.159.04.run
```

#### Step 2: Remove the conflicting apt packages inside the LXC

```bash
apt remove --purge libnvidia-compute-580 nvidia-utils-580 libnvidia-decode-580 libnvidia-encode-580 nvidia-kernel-common-580 nvidia-firmware-580-580.159.03
apt autoremove
```

* `apt remove --purge` uninstalls the packages and deletes their configuration files. Leaving the old `.so` files on disk would cause the `.run` installer to detect a conflict or the dynamic linker to load the wrong version.
* `apt autoremove` cleans up any dependency packages that were pulled in by the removed packages and are no longer needed.

#### Step 3: Run the .run installer with --no-kernel-module

```bash
bash /root/NVIDIA-Linux-x86_64-580.159.04.run --no-kernel-module --no-questions --ui=none
```

* `--no-kernel-module` tells the installer to skip compiling and loading a kernel module entirely. The LXC has no kernel of its own — it uses the host kernel module directly. Attempting to install a kernel module inside an LXC would fail and is unnecessary.
* `--no-questions` suppresses interactive prompts.
* `--ui=none` disables the ncurses interface so output is plain text, which is appropriate for a headless container environment.

#### Step 4: Refresh the dynamic linker cache

```bash
ldconfig
```

`ldconfig` rebuilds the cache that maps shared library names to their paths on disk. After the `.run` installer drops new `.so` files, running `ldconfig` ensures the system finds the new versions rather than any stale cache entries.

#### Step 5: Verify the library version

```bash
ldconfig -p | grep libcuda
find /usr /lib -name "libcuda.so*" 2>/dev/null
```

Expected output:
```
libcuda.so.1 (libc6,x86-64) => /lib/x86_64-linux-gnu/libcuda.so.1
libcuda.so   (libc6,x86-64) => /lib/x86_64-linux-gnu/libcuda.so
/usr/lib/x86_64-linux-gnu/libcuda.so.580.159.04
/usr/lib/x86_64-linux-gnu/libcuda.so.1
/usr/lib/x86_64-linux-gnu/libcuda.so
```

`libcuda.so.580.159.04` must be present and must be the only version on disk. If `.03` files appear alongside it, the purge in Step 2 did not complete cleanly and the stale files need to be removed manually.

---

## Issue 2: GPU Device Nodes Have No Permissions

### Symptom

After fixing the library mismatch, FFmpeg still fails to initialize CUDA:

```
[AVHWDeviceContext @ 0x...] cu->cuInit(0) failed -> CUDA_ERROR_UNKNOWN: unknown error
Failed to set value 'cuda=cu:0' for option 'init_hw_device': Generic error in an external library
```

### Diagnosis

#### Step 1: Inspect the device node permissions inside the LXC

```bash
ls -la /dev/nvidia*
```

Output:
```
---------- 1 root root        0 May 28 01:42 /dev/nvidia-uvm
---------- 1 root root        0 May 28 01:42 /dev/nvidia-uvm-tools
---------- 1 root root        0 May 28 01:42 /dev/nvidia0
crw-rw-rw- 1 root root 195, 255 May 28 01:47 /dev/nvidiactl
```

Three of the four device nodes show `----------` — no read, write, or execute permission for any user, including root. `nvidiactl` has correct permissions (`crw-rw-rw-`) but the other three do not. The LXC inherits these device nodes via `lxc.mount.entry` bind mounts from the host, so whatever permissions the host assigns are what the container sees.

### Root Cause

After the Proxmox reinstall, the host did not have a udev rule to set permissions on the NVIDIA device nodes. When the `nvidia-uvm` kernel module initialises at boot it creates `/dev/nvidia-uvm` and `/dev/nvidia-uvm-tools` with mode `0600` restricted to root, and `/dev/nvidia0` is created similarly. Without a udev rule overriding this, the devices are inaccessible to any process not running as root — including the Jellyfin ffmpeg subprocess, which runs as the `jellyfin` service user.

---

### Fix

#### Step 1: Apply permissions immediately inside the LXC

```bash
chmod 666 /dev/nvidia0 /dev/nvidia-uvm /dev/nvidia-uvm-tools
```

* `chmod 666` grants read and write permission to owner, group, and others.
* This change takes effect immediately but does not survive a container restart because the device nodes are bind-mounted from the host and reset to their host permissions on each mount.

#### Step 2: Make the fix permanent via a udev rule on the Proxmox host

```bash
cat > /etc/udev/rules.d/99-nvidia-lxc.rules << 'EOF'
KERNEL=="nvidia[0-9]*", MODE="0666"
KERNEL=="nvidia-uvm", MODE="0666"
KERNEL=="nvidia-uvm-tools", MODE="0666"
KERNEL=="nvidiactl", MODE="0666"
EOF
```

* udev rules are evaluated by the kernel's device manager when devices are created or modified. Placing a rule in `/etc/udev/rules.d/` with a high numeric prefix (`99-`) ensures it runs after the default rules and overrides any earlier mode assignments.
* `KERNEL=="nvidia[0-9]*"` matches `nvidia0`, `nvidia1`, and so on for multi-GPU setups.
* `MODE="0666"` sets the device node to world-readable and world-writable, which is appropriate for a single-user homelab server where no untrusted users have shell access to the host.

#### Step 3: Reload udev rules on the host

```bash
udevadm control --reload-rules && udevadm trigger
```

* `udevadm control --reload-rules` tells the running udev daemon to re-read all rule files from disk without requiring a reboot.
* `udevadm trigger` re-evaluates the rules against all currently present devices, applying the new `MODE="0666"` to the NVIDIA device nodes immediately.

---

## Issue 3: FFmpeg Exits with Code 187, Transcoding Fails

### Symptom

Playback works in a browser (direct stream, no transcode required) but fails immediately on the Roku. The Jellyfin journal shows FFmpeg exiting with code 187 on every transcode attempt:

```
[ERR] FFmpeg exited with code 187
[ERR] Error processing request. URL GET /videos/.../hls1/main/0.ts
MediaBrowser.Common.FfmpegException: FFmpeg exited with code 187
```

The FFmpeg command Jellyfin is issuing uses `-init_hw_device cuda=cu:0` and `-codec:v:0 h264_nvenc`, meaning every transcode is routed through NVENC. Exit code 187 is the code FFmpeg returns when hardware device initialisation fails at startup — the process exits before reading a single frame of input.

### Root Cause

Exit code 187 is a direct consequence of Issues 1 and 2 above. FFmpeg calls `cuInit()` to open the CUDA device. With either the wrong library version or inaccessible device nodes, `cuInit()` returns `CUDA_ERROR_UNKNOWN` and FFmpeg exits immediately. The Roku requires a transcode because it only supports H.264 in a limited set of containers, while the browser client can receive the source file directly. This is why the browser worked while the Roku did not.

### Fix

Resolving Issues 1 and 2 above resolves this issue entirely. No additional configuration is required in Jellyfin.

### Verification

After applying both fixes, confirm CUDA initialises cleanly using Jellyfin's own bundled ffmpeg binary:

```bash
/usr/lib/jellyfin-ffmpeg/ffmpeg -init_hw_device cuda=cu:0 -f lavfi -i nullsrc -t 1 -f null - 2>&1 | grep -i "cuda\|nvenc\|error"
```

Expected output — the configuration line only, with no error messages:
```
  configuration: --prefix=/usr/lib/jellyfin-ffmpeg ... --enable-cuda --enable-nvenc
```

If the command returns only the configuration line and exits cleanly, CUDA initialised successfully and Jellyfin's hardware transcoding pipeline is fully operational.

---

## Issue 4: Device Nodes Absent After Reboot

### Symptom

After a full Proxmox host reboot, the NVIDIA device nodes no longer exist:

```
root@Proxmox:~# ls -la /dev/nvidia*
ls: cannot access '/dev/nvidia*': No such file or directory
```

FFmpeg inside the LXC fails again with `CUDA_ERROR_UNKNOWN` and transcoding stops working.

### Root Cause

The `nvidia-uvm` kernel module is not loaded automatically on boot. The `nvidia` module loads because it is registered with DKMS, but `nvidia-uvm` — which is responsible for creating `/dev/nvidia-uvm` and `/dev/nvidia-uvm-tools` — has no equivalent autoload entry. Without the module loaded, the device nodes are never created, so the udev permission rules have nothing to act on and the LXC bind mounts produce empty placeholder files.

Additionally, even when the modules are loaded, the device nodes are only created when something first accesses the GPU. Running `nvidia-smi` triggers this initialisation as a side effect.

### Diagnosis

#### Step 1: Confirm the modules are loaded but devices are missing

```bash
lsmod | grep nvidia
```

Output:
```
nvidia_uvm           1986560  0
nvidia_drm            131072  0
nvidia_modeset       1683456  1 nvidia_drm
nvidia              105308160  3 nvidia_uvm,nvidia_drm,nvidia_modeset
```

The modules are present but `/dev/nvidia*` does not exist — the device nodes were never created because nothing triggered GPU initialisation.

---

### Fix

#### Step 1: Configure nvidia-uvm to load at boot

```bash
echo "nvidia-uvm" >> /etc/modules-load.d/nvidia.conf
```

`/etc/modules-load.d/` contains files listing kernel modules that systemd should load automatically during boot. Adding `nvidia-uvm` here ensures the module is loaded before any service or container that depends on the GPU devices.

#### Step 2: Create a systemd service to initialize the device nodes

Even with the module loaded, the device nodes are not created until something accesses the GPU. A dedicated oneshot service running `nvidia-smi` at boot triggers this initialisation before the LXC container starts.

```bash
cat > /etc/systemd/system/nvidia-device-init.service << 'EOF'
[Unit]
Description=Initialize NVIDIA device nodes
Before=lxc.service
After=systemd-udev-settle.service

[Service]
Type=oneshot
ExecStart=/usr/bin/nvidia-smi
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF
```

**`[Unit]`**

**`Before=lxc.service`**
Declares that this service must complete before the LXC service starts. This guarantees the device nodes exist and have correct permissions before the container's bind mounts are evaluated.

**`After=systemd-udev-settle.service`**
Ensures all udev events from module loading have been processed before `nvidia-smi` runs, so the device nodes udev would normally create are already present.

**`[Service]`**

**`Type=oneshot`**
Tells systemd this service runs a single command to completion rather than starting a long-running daemon. Systemd waits for the command to finish before considering the service started and moving on to dependent units.

**`ExecStart=/usr/bin/nvidia-smi`**
Runs `nvidia-smi`, which initialises the NVIDIA driver and triggers creation of all GPU device nodes as a side effect. This is the mechanism that makes `/dev/nvidia0`, `/dev/nvidia-uvm`, and `/dev/nvidia-uvm-tools` appear.

**`RemainAfterExit=yes`**
Keeps the service in an `active` state after the command exits so that dependent units can correctly determine the service has run.

#### Step 3: Enable the service

```bash
systemctl daemon-reload
systemctl enable nvidia-device-init.service
```

Output:
```
Created symlink '/etc/systemd/system/multi-user.target.wants/nvidia-device-init.service'
→ '/etc/systemd/system/nvidia-device-init.service'.
```

---

## Issue 5: nvidia-uvm Major Device Number Changes Between Reboots

### Symptom

After a reboot, the LXC container starts but CUDA initialisation fails again with `CUDA_ERROR_UNKNOWN`. The device nodes exist on the host with correct permissions, but the ffmpeg test inside the LXC still fails.

### Root Cause

The Linux kernel assigns major device numbers to dynamically registered devices at module load time. The `nvidia-uvm` module registers under the `misc` subsystem, which means its major number is allocated from a shared pool and is not guaranteed to be the same across reboots. The LXC container configuration in `/etc/pve/lxc/100.conf` contains a `lxc.cgroup2.devices.allow` entry that whitelists a specific major number for the `nvidia-uvm` device. When the major number changes, the cgroup rule no longer matches the actual device and the container is denied access at the kernel level, even though the bind mount exists and the permissions look correct.

### Diagnosis

#### Step 1: Check the current major number on the host

```bash
ls -la /dev/nvidia-uvm
```

Output after a reboot:
```
crw-rw-rw- 1 root root 235, 0 May 27 21:11 /dev/nvidia-uvm
```

#### Step 2: Compare against the cgroup rule in the container config

```bash
grep cgroup2 /etc/pve/lxc/100.conf
```

Output:
```
lxc.cgroup2.devices.allow: c 195:* rwm
lxc.cgroup2.devices.allow: c 234:* rwm
lxc.cgroup2.devices.allow: c 237:* rwm
```

The config still references major number `234` from the previous boot, but the device is now on `235`. The kernel silently denies all access to the device from inside the container.

---

### Fix

#### Step 1: Update the cgroup rule to the current major number

```bash
sed -i 's/lxc.cgroup2.devices.allow: c 234/lxc.cgroup2.devices.allow: c 235/' /etc/pve/lxc/100.conf
```

Verify the change:

```bash
grep cgroup2 /etc/pve/lxc/100.conf
```

Output:
```
lxc.cgroup2.devices.allow: c 195:* rwm
lxc.cgroup2.devices.allow: c 235:* rwm
lxc.cgroup2.devices.allow: c 237:* rwm
```

#### Step 2: Restart the container

```bash
pct stop 100 && pct start 100
```

#### Step 3: Prevent the major number from changing between reboots

Add a `static_node` udev rule that pins the major number for `nvidia-uvm` so it is allocated consistently across reboots:

```bash
cat >> /etc/udev/rules.d/99-nvidia-lxc.rules << 'EOF'
SUBSYSTEM=="misc", KERNEL=="nvidia-uvm", GROUP="video", MODE="0666", OPTIONS+="static_node=nvidia-uvm"
SUBSYSTEM=="misc", KERNEL=="nvidia-uvm-tools", GROUP="video", MODE="0666", OPTIONS+="static_node=nvidia-uvm-tools"
EOF

udevadm control --reload-rules && udevadm trigger
```

* `OPTIONS+="static_node=nvidia-uvm"` instructs udev to create a static device node entry for `nvidia-uvm` in the initramfs device database. This causes the kernel to reserve the same major number for the device on every boot rather than allocating it dynamically from the shared pool.
* Once the major number is stable, the `lxc.cgroup2.devices.allow` entry in `100.conf` will remain correct across reboots without manual intervention.

#### Step 4: Verify the major number is stable across reboots

After adding the static_node rule, reboot and confirm the major number is unchanged:

```bash
ls -la /dev/nvidia-uvm
```

If the number matches what was in the cgroup rule before the reboot, the fix is working. The setup is fully persistent when all of the following are true after a cold boot:

* All four device nodes exist on the host with `crw-rw-rw-` permissions
* The major number of `nvidia-uvm` matches the `lxc.cgroup2.devices.allow` entry in `100.conf`
* The ffmpeg CUDA test inside the LXC returns no errors

---

## Troubleshooting Reference

| Symptom | Cause | Fix |
|---------|-------|-----|
| `nvidia-smi` in LXC: `Driver/library version mismatch` | Host `.run` driver (580.159.04) vs LXC apt packages (580.159.03) | Purge apt NVIDIA packages in LXC; reinstall via `.run` with `--no-kernel-module` |
| `cuInit(0) failed -> CUDA_ERROR_UNKNOWN` in ffmpeg | GPU device nodes have `----------` permissions | `chmod 666 /dev/nvidia0 /dev/nvidia-uvm /dev/nvidia-uvm-tools`; add udev rule for persistence |
| FFmpeg exits with code 187 on every transcode | CUDA device initialisation failure caused by library mismatch or device permissions | Resolve Issues 1 and 2 above |
| Browser playback works, Roku playback fails immediately | Browser direct-streams; Roku requires transcode which hits the NVENC failure | Resolve Issues 1 and 2 above |
| `/dev/nvidia*` absent after reboot | `nvidia-uvm` module not loading on boot; no service triggering device node creation | Add `nvidia-uvm` to `/etc/modules-load.d/nvidia.conf`; create `nvidia-device-init.service` |
| `CUDA_ERROR_UNKNOWN` after reboot despite devices existing | `nvidia-uvm` major number changed; cgroup rule in `100.conf` no longer matches | Update `lxc.cgroup2.devices.allow` to new major number; add `static_node` udev rule to pin it |
| Device node permissions reset after container restart | udev rule missing on host; bind-mounted nodes inherit host permissions on each mount | Add `/etc/udev/rules.d/99-nvidia-lxc.rules` on Proxmox host |

---

## References

* https://download.nvidia.com/XFree86/Linux-x86_64/580.159.04/NVIDIA-Linux-x86_64-580.159.04.run
* https://www.freedesktop.org/software/systemd/man/udev_rules.html
* https://jellyfin.org/docs/general/administration/hardware-acceleration/nvidia/
* https://forum.proxmox.com/threads/2025-proxmox-pcie-gpu-passthrough-with-nvidia.169543/
