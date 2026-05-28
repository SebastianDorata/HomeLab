# Troubleshooting: LXC Boot Failure and Jellyfin Library Permissions

This document covers two issues encountered after a clean Proxmox reinstall: the Jellyfin LXC container failing to start on boot due to an LVM activation race condition, and Jellyfin being unable to read the media library due to incorrect filesystem permissions.

---

## Issue 1: LXC Container Fails to Start on Boot

### Symptom

When starting CT 100 from the Proxmox web UI or via `pct start 100`, the following error appears:

```
run_buffer: 569 Script exited with status 32
lxc_init: 1037 Failed to run lxc.hook.pre-start for container "100"
__lxc_start: 2208 Failed to initialize container "100"
TASK ERROR: startup for container '100' failed
```

### Root Cause

The 500GB media volume (`vm-100-disk-1`) is a thin-provisioned LVM logical volume. On boot, the LXC container attempts to mount the volume before LVM has finished activating it. Because the block device does not yet exist when the container's pre-start hook runs, the hook exits with status 32 and the container fails to initialise.

---

### Fix 1a: Temporary Workaround

Manually activate the logical volume, then start the container:

```bash
lvchange -ay pve/vm-100-disk-1 && pct start 100
```

`lvchange -ay` activates the specified logical volume, making the block device available at `/dev/pve/vm-100-disk-1`. Once the device exists, `pct start 100` can mount it and the container starts successfully. This workaround must be repeated every time the host reboots, so a permanent fix is required.

---

### Fix 1b: Permanent Fix via systemd Service

#### Step 1: Check if the service already exists

```bash
systemctl status lvm-activate-pve.service
```
Output:
```
Unit lvm-activate-pve.service could not be found.
```

#### Step 2: Create the service

The following heredoc writes a new systemd unit file in a single command. A heredoc is a shell technique for writing a multi-line block of text into a file without opening a text editor. `cat >` writes the content to the specified file path, and `<< 'EOF'` tells the shell to treat everything that follows as input until it encounters `EOF` on a line by itself. The single quotes around `'EOF'` prevent the shell from expanding variables inside the block.

```bash
cat > /etc/systemd/system/lvm-activate-pve.service << 'EOF'
[Unit]
Description=Activate PVE LVM Thin Volumes
DefaultDependencies=no
Before=local-fs.target
After=lvm2-activation.service

[Service]
Type=oneshot
ExecStart=/sbin/lvchange -ay pve/data
ExecStart=/sbin/lvchange -ay pve/vm-100-disk-1
RemainAfterExit=yes

[Install]
WantedBy=local-fs.target
EOF
```

#### Breakdown

**`[Unit]`**
The Unit section defines metadata and dependency relationships for the service, specifically how the service fits into the boot sequence relative to other units.

**`Description=Activate PVE LVM Thin Volumes`**
A human-readable label that appears in `systemctl status` output and journal logs to identify what this service does.

**`DefaultDependencies=no`**
By default, systemd adds implicit ordering dependencies to every unit to prevent units from running too early in the boot process. Disabling default dependencies is necessary here because this service needs to run before the normal dependency graph is fully resolved, specifically to activate the LVM volumes before any other unit attempts to mount them.

**`Before=local-fs.target`**
Declares that this service must complete before `local-fs.target` is reached. `local-fs.target` is the synchronisation point where all local filesystems are expected to be mounted. Declaring `Before` guarantees that the LVM volumes are active before anything attempts to mount them.

**`After=lvm2-activation.service`**
Declares that this service must run after `lvm2-activation.service`, which is the standard LVM startup unit. This ensures that basic LVM infrastructure is initialised before attempting to activate specific volumes.

**`[Service]`**
The Service section defines how the service actually runs.

**`Type=oneshot`**
Tells systemd that this service runs a command to completion and exits, rather than starting a long-running daemon. Systemd waits for the command to finish before considering the service started and moving on to dependent units.

**`ExecStart=/sbin/lvchange -ay pve/data`**
Activates the `data` thin pool. The thin pool must be activated first because individual thin volumes such as `vm-100-disk-1` depend on it. Attempting to activate a thin volume without its parent pool active will fail.

**`ExecStart=/sbin/lvchange -ay pve/vm-100-disk-1`**
Activates the specific 500GB volume that the LXC media mount depends on. Having two `ExecStart` lines in a `oneshot` service causes the commands to run sequentially. The second command only runs after the first completes successfully.

**`RemainAfterExit=yes`**
Keeps the service in an `active` state after the commands finish. Without this flag, systemd marks the service as `inactive (dead)` immediately after the commands complete, which could cause dependent units to incorrectly conclude that the service failed. With this flag set, `systemctl status lvm-activate-pve.service` shows `active (exited)`, confirming the service ran successfully.

**`[Install]`**
The Install section defines how the service is wired into the boot process when `systemctl enable` is run.

**`WantedBy=local-fs.target`**
When enabled, this creates a symlink inside `local-fs.target.wants/` pointing to this service file. This symlink is what causes systemd to run the service during boot. Without it, the service file exists on disk but is never triggered.

---

#### Step 3: Enable the service

Reload the systemd daemon to pick up the new unit file, then enable the service:

```bash
systemctl daemon-reload
```
```bash
systemctl enable lvm-activate-pve.service
```

Output:
```
Created symlink '/etc/systemd/system/local-fs.target.wants/lvm-activate-pve.service' 
→ '/etc/systemd/system/lvm-activate-pve.service'.
```

#### Step 4: Confirm the service is enabled

```bash
systemctl status lvm-activate-pve.service
```

Output:
```
lvm-activate-pve.service - Activate PVE LVM Thin Volumes
Loaded: loaded (/etc/systemd/system/lvm-activate-pve.service; enabled; preset: enabled)
Active: inactive (dead)
```

`inactive (dead)` is the correct state at this point. The service has not run yet in the current session, but it is loaded and enabled, meaning systemd will execute the service automatically on the next boot before any filesystem mounts are attempted.

#### Step 5: Configure CT 100 to start automatically on boot

```bash
pct set 100 --onboot 1
```

`pct set` modifies the container configuration. `--onboot 1` tells Proxmox to automatically start CT 100 after the host finishes booting, removing the need to start the container manually after every reboot.

#### Step 6: Reboot to verify

```bash
shutdown -r now
```

On the next boot, CT 100 started automatically without any manual intervention.

---

## Issue 2: Jellyfin Cannot Read the Media Library

### Symptom

The library path is set correctly in the Jellyfin web UI, but after scanning, no media appears. The Jellyfin journal shows repeated warnings:

```
[WRN] Library folder /mnt/media/Movies is inaccessible or empty, skipping
```

### Diagnosis

#### Step 1: Confirm the files are present

```bash
ls /mnt/media/Movies
```

The files are present, which rules out a mount or transfer issue and points to a permissions problem.

#### Step 2: Check the Jellyfin logs

```bash
journalctl -u jellyfin | grep -i "movies\|scan\|error" | tail -20
```
* `journalctl -u jellyfin` queries the systemd journal for all log entries produced by the Jellyfin service. 
* The output is filtered with `grep` for lines mentioning `movies`, `scan`, or `error`, and `tail -20` limits the output to the 20 most recent matching lines. 
* The repeated `inaccessible or empty` warning confirms that Jellyfin can resolve the path but cannot read the directory contents.

Output:
``` 
root@lxc-jellyfin:~# journalctl -u jellyfin | grep -i "movies\|scan\|error" | tail -20
May 27 07:11:29 lxc-jellyfin jellyfin[161]: [07:11:29] [INF] Scan Media Library Completed after 0 minute(s) and 0 seconds
May 27 07:20:02 lxc-jellyfin jellyfin[161]: [07:20:02] [WRN] Library folder /mnt/media/Movies is inaccessible or empty, skipping
May 27 07:20:02 lxc-jellyfin jellyfin[161]: [07:20:02] [WRN] Library folder /mnt/media/Movies is inaccessible or empty, skipping
May 27 07:20:02 lxc-jellyfin jellyfin[161]: [07:20:02] [INF] Scan Media Library Completed after 0 minute(s) and 0 seconds
May 27 07:20:05 lxc-jellyfin jellyfin[161]: [07:20:05] [WRN] Library folder /mnt/media/Movies is inaccessible or empty, skipping
May 27 07:20:05 lxc-jellyfin jellyfin[161]: [07:20:05] [WRN] Library folder /mnt/media/Movies is inaccessible or empty, skipping
May 27 07:20:05 lxc-jellyfin jellyfin[161]: [07:20:05] [INF] Scan Media Library Completed after 0 minute(s) and 0 seconds
May 27 07:20:28 lxc-jellyfin jellyfin[161]: [07:20:28] [ERR] Error processing request: The operation was canceled. URL GET /Sessions.
May 27 07:33:31 lxc-jellyfin jellyfin[161]: [07:33:31] [WRN] Library folder /mnt/media/Movies is inaccessible or empty, skipping
May 27 07:33:31 lxc-jellyfin jellyfin[161]: [07:33:31] [WRN] Library folder /mnt/media/Movies is inaccessible or empty, skipping
May 27 07:33:31 lxc-jellyfin jellyfin[161]: [07:33:31] [INF] Scan Media Library Completed after 0 minute(s) and 0 seconds
May 27 07:33:33 lxc-jellyfin jellyfin[161]: [07:33:33] [WRN] Library folder /mnt/media/Movies is inaccessible or empty, skipping
May 27 07:33:33 lxc-jellyfin jellyfin[161]: [07:33:33] [WRN] Library folder /mnt/media/Movies is inaccessible or empty, skipping
May 27 07:33:33 lxc-jellyfin jellyfin[161]: [07:33:33] [INF] Scan Media Library Completed after 0 minute(s) and 0 seconds
May 27 14:54:25 lxc-jellyfin jellyfin[144]: [14:54:25] [WRN] Library folder /mnt/media/Movies is inaccessible or empty, skipping
May 27 14:54:26 lxc-jellyfin jellyfin[144]: [14:54:26] [WRN] Library folder /mnt/media/Movies is inaccessible or empty, skipping
May 27 14:54:26 lxc-jellyfin jellyfin[144]: [14:54:26] [INF] Scan Media Library Completed after 0 minute(s) and 0 seconds
May 27 14:54:28 lxc-jellyfin jellyfin[144]: [14:54:28] [WRN] Library folder /mnt/media/Movies is inaccessible or empty, skipping
May 27 14:54:28 lxc-jellyfin jellyfin[144]: [14:54:28] [WRN] Library folder /mnt/media/Movies is inaccessible or empty, skipping
May 27 14:54:28 lxc-jellyfin jellyfin[144]: [14:54:28] [INF] Scan Media Library Completed after 0 minute(s) and 0 seconds
root@lxc-jellyfin:~#
```

#### Step 3: Inspect the filesystem permissions

```bash
ls -la /mnt/media/
```

Output:
```
drwxr-xr-x   5 root root  4096  .
drwxr-xr-x   3 root root  4096  ..
drwx------+  2 root root  4096  Movies
drwx------+ 13 root root  4096  TV Shows
drwx------   2 root root 16384  lost+found
```

### Root Cause

The `Movies` and `TV Shows` directories were transferred from macOS via SMB. macOS applied `drwx------` permissions during the transfer, restricting access to the `root` user only. The `jellyfin` service runs under its own dedicated system user account, not as `root`, so the Jellyfin process has no read or execute permission on either directory or any of the files inside them.

### Fix

#### Step 4: Fix the permissions

```bash
chmod -R 755 /mnt/media/Movies
```
```bash
chmod -R 755 /mnt/media/'TV Shows'
```

* `chmod -R` applies the permission change recursively to the specified directory and everything inside it. 
* `755` sets read, write, and execute for the owner (`root`), and read and execute for group and others. 
* The `jellyfin` user falls into the "others" category, so granting read and execute to others is what allows Jellyfin to traverse the directories and read the media files. 
* After running these commands, trigger a library scan from the Jellyfin web UI under **Dashboard > Libraries**.
---
# References:

* https://jellyfin.org/docs/general/faq/
* https://forum.proxmox.com/threads/lxc-does-not-start-exit-code-32.166611/
* https://forum.proxmox.com/threads/proxmox-virtual-environment-8-2-2-lxc-high-availibilty-after-upgrade-pve-v7-to-v8.148785/
* https://stackoverflow.com/questions/57496357/systemd-adding-service-into-multi-user-target-wants-folder-only-works-as-a-symli
* https://stackoverflow.com/questions/57496357/systemd-adding-service-into-multi-user-target-wants-folder-only-works-as-a-symli
