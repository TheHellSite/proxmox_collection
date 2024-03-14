# Prologue - LXC device passthrough

## General Information
This tutorial should work for any kind of PVE host device that is intended to be shared with LXCs, no matter if the LXC is privileged or unprivileged. For the actual step-by-step part of this tutorial the renderD128 device is used as an example.

## What is the issue?
By default devices on the PVE host are not accessible inside LXCs, mainly due to permission problems but also because they are simply not available inside LXC environments. The latter can easily be fixed by adding the desired device to the LXC config file.

The permission problem is a bit more difficult to solve. This is because UIDs/GIDs of devices on the PVE host are not always equal inside LXCs.

For privileged LXCs those UIDs/GIDs are mostly unequal to the PVE host when using other Linux derivates inside the LXC (f.e. Arch Linux).\
As an example on the Debian based PVE host group render has the GID=103, but inside an Arch Linux LXC group render has the GID=989.

With unprivileged LXCs this is even more complicated because here each UID/GID is incremented by 100000.\
As an example on the Debian based PVE host group render has the GID=103 and inside an unprivileged Arch Linux LXC group render has the GID=989. However, for the PVE host the (unprivileged) GID=989 is actually the GID=100989, see the below Proxmox Wiki link for more information on this.\
https://pve.proxmox.com/wiki/Unprivileged_LXC_containers

## What is the solution?
1. Add the desired device to the LXC config file.
2. Choose a universal GID=111000 that can exist on the PVE host, in privileged LXCs and in unprivileged LXCs.
3. Assign the desired device to the UID=100000, which belongs to the root user inside unprivileged LXCs.
4. Assign the desired device to the universal GID=111000 so it can be used simultaneously on the PVE host and inside multiple privileged/unprivileged LXCs.
5. Create the group "lxc_gpu_shares" with GID=111000 inside the LXCs.
6. Assign LXC users, that need access to the device, to the group "lxc_gpu_shares"

## What are the benefits?
- much easier to set up and understand than ID mappings (https://pve.proxmox.com/wiki/Unprivileged_LXC_containers#Using_local_directory_bind_mount_points)
- a unified solution that works with both privileged and unprivileged LXCs
- the root user in privileged and unprivileged LXCs has full access to the device
- the GPU (renderD128 device) can be assigned and simultaneously used by as many LXCs as desired
- works for any kind of device, not just GPUs

---

# Tutorial - LXC device passthrough

## 1. PVE Host: Get the device ID.
       
```
root@pve:~# ls -l /dev/dri/renderD128
crw-rw---- 1 root render 226, 128 Feb 28 17:38 /dev/dri/renderD128
  
--> In this case "226, 128" is the renderD128 device ID.
```

### 1.1 (optional) PVE Host: Get the AMD tone mapping device ID.

This step is AMD specific and only necessary for tone mapping support.

```
root@pve:~# ls -l /dev/kfd
crw-rw---- 1 root render 235, 0 Feb 28 17:38 /dev/kfd

--> In this case "235, 0" is the kfd device ID.
```

## 2. PVE Host: Shutdown the LXC and run the command below.

> [!IMPORTANT]
> - Adjust the "LXC_ID" at the beginning of the command! (f.e. "/etc/pve/lxc/100.conf")
> - Make sure to copy and paste the whole command!
> - *Adjust the renderD128 device ID! (if necessary)*

<details>
  <summary>:warning: <b>Spoiler: command explanation</b></summary>
  
  ```
  1. Append the lines between EOF to the LXC config file.
          cat <<EOF >> /etc/pve/lxc/LXC_ID.conf
          ...
          EOF
  2. Grant the LXC access to the renderD128 device of the PVE host.
          lxc.cgroup2.devices.allow: c 226:128 rwm
  3. Mount the renderD128 device inside the LXC.
          lxc.mount.entry: /dev/dri/renderD128 dev/dri/renderD128 none bind,optional,create=file
  4. Change the UID/GID of the renderD128 device on the PVE host, right before actually starting the LXC, to the UID/GID matching root:lxc_gpu_shares inside the LXC.
          lxc.hook.pre-start: sh -c "chown 100000:111000 /dev/dri/renderD128"
  ```
</details>

```
cat <<EOF >> /etc/pve/lxc/LXC_ID.conf
lxc.cgroup2.devices.allow: c 226:128 rwm
lxc.mount.entry: /dev/dri/renderD128 dev/dri/renderD128 none bind,optional,create=file
lxc.hook.pre-start: sh -c "chown 100000:111000 /dev/dri/renderD128"
EOF
```

### 2.1 (optional) PVE Host: Add the AMD tone mapping device to the LXC config.

This step is AMD specific and only necessary for tone mapping support.

> [!IMPORTANT]
> - Adjust the "LXC_ID" at the beginning of the command! (f.e. "/etc/pve/lxc/100.conf")
> - Make sure to copy and paste the whole command!
> - *Adjust the kfd device ID! (if necessary)*

```
cat <<EOF >> /etc/pve/lxc/LXC_ID.conf
lxc.cgroup2.devices.allow: c 235:0 rwm
lxc.mount.entry: /dev/kfd dev/kfd none bind,optional,create=file
lxc.hook.pre-start: sh -c "chown 100000:111000 /dev/kfd"
EOF
```

## 3. LXC Guest: Final steps.

1. Start the LXC.

2. Create the group "lxc_gpu_shares" in the privileged or unprivileged LXC.
   
   - Privileged LXCs - Create the group "lxc_gpu_shares" with GID=111000.
      
      ```
      root@lxc:~# groupadd -g 111000 lxc_gpu_shares
      ```
   
   - Unprivileged LXCs - Create the group "lxc_gpu_shares" with GID=11000.
      
      ```
      root@lxc:~# groupadd -g 11000 lxc_gpu_shares
      ```

3. Add service users (f.e. jellyfin) to the group "lxc_gpu_shares".
   
   ```
   gpasswd -a jellyfin lxc_gpu_shares
   ```

4. At this point it is most likely necessary to install additional drivers inside the LXC.\
Take a look at my Jellyfin VA-API guide for this: https://github.com/TheHellSite/archlinux_lxc/tree/main/jellyfin#jellyfin-va-api-hardware-accelerated-video-transcoding-run-as-root-user
