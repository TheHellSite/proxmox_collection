# LXC GPU passthrough (run commands as root user)

The GPU passthrough guide below should work for all GPUs listed here: https://docs.mesa3d.org/systems.html

### 1. PVE Host: Get the render device ID.

  ```
  ls -l /dev/dri
  ```

  Example output:
  ```
  total 0
  drwxr-xr-x 2 root root         80 Oct  1 10:51 by-path
  crw-rw---- 1 root video  226,   0 Oct  1 10:51 card0
  crw-rw---- 1 root render 226, 128 Oct  1 10:51 renderD128
  
  --> In this case "226,128" is the render device ID.
  ```

### 2. PVE Host: Shutdown the LXC, run one of the commands below and start the LXC.

  **!!! Adjust the "LXC_ID" at the end of the command !!! (necessary)**\
  !!! Adjust the render device ID !!! *(if necessary)*\
  !!! Adjust the GID=989 in the "chown" command to match the GID of group "render" in your LXC !!! *(if necessary)*

  <details>
  <summary><b>Command explanation</b></summary>
    
    1. Grant the LXC access to the render device of the PVE host.
           lxc.cgroup2.devices.allow: c 226:128 rwm
    2. Mount the render device in the LXC.
           lxc.mount.entry: /dev/dri/renderD128 dev/dri/renderD128 none bind,optional,create=file
    3. Change UID and GID of the render device to root:render on the PVE host during each start of the LXC.
           lxc.hook.pre-start: sh -c "chown 0:100989 /dev/dri/renderD128"
  </details>

  ```
  # use the command below if your LXC is unprivileged
  { echo 'lxc.cgroup2.devices.allow: c 226:128 rwm' ; echo 'lxc.mount.entry: /dev/dri/renderD128 dev/dri/renderD128 none bind,optional,create=file' ; echo 'lxc.hook.pre-start: sh -c "chown 0:100989 /dev/dri/renderD128"' ; } | tee -a /etc/pve/lxc/LXC_ID.conf
  
  # use the command below if your LXC is privileged
  { echo 'lxc.cgroup2.devices.allow: c 226:128 rwm' ; echo 'lxc.mount.entry: /dev/dri/renderD128 dev/dri/renderD128 none bind,optional,create=file' ; echo 'lxc.hook.pre-start: sh -c "chown 0:989 /dev/dri/renderD128"' ; } | tee -a /etc/pve/lxc/LXC_ID.conf
  ```

### 3. LXC Guest: Final steps.

  1. Start the LXC.

  2. Add service account (f.e. jellyfin) to group "render".

  ```
  usermod -aG render jellyfin
  ```

  3. At this point it is most likely necessary to install additional drivers in the LXC.  
  Take a look at my Jellyfin VA-API guide for this: https://github.com/TheHellSite/archlinux_lxc/blob/main/jellyfin/README.md#jellyfin-va-api-hardware-transcoding-run-as-root-user
