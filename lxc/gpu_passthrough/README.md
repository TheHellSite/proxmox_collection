# LXC GPU passthrough (run commands as root user)

### 1. PVE Host: Get the renderD128 device ID.

  ```
  ls -l /dev/dri
  ```

  Example output:
  ```
  total 0
  drwxr-xr-x 2 root root         80 Oct  1 10:51 by-path
  crw-rw---- 1 root video  226,   0 Oct  1 10:51 card0
  crw-rw---- 1 root render 226, 128 Oct  1 10:51 renderD128
  
  --> In this case "226,128" is the renderD128 device ID.
  ```

### 2. PVE Host: Shutdown the LXC, run one of the commands below and start the LXC.

  **!!! Adjust the "LXC_ID" at the end of the command !!! (necessary)**\
  !!! Adjust the renderD128 device ID !!! *(if necessary)*\
  !!! Adjust the GID=989 in the "chown" command to match the GID of group "render" in your LXC !!! *(if necessary)*
  !!! For unprivileged LXCs the ID must be incremented by 100000 !!!

  <details>
  <summary><b>Command explanation</b></summary>
    
    1. Grant the LXC access to the renderD128 device of the PVE host.
           lxc.cgroup2.devices.allow: c 226:128 rwm
    2. Mount the renderD128 device in the LXC.
           lxc.mount.entry: /dev/dri/renderD128 dev/dri/renderD128 none bind,optional,create=file
    3. Change the UID/GID of the renderD128 device on the PVE host, right before actually starting the LXC, to the UID/GID matching root:render inside the LXC.
           lxc.hook.pre-start: sh -c "chown 100000:100989 /dev/dri/renderD128"
  </details>

  ```
  # use the command below if your LXC is unprivileged
  { echo 'lxc.cgroup2.devices.allow: c 226:128 rwm' ; echo 'lxc.mount.entry: /dev/dri/renderD128 dev/dri/renderD128 none bind,optional,create=file' ; echo 'lxc.hook.pre-start: sh -c "chown 100000:100989 /dev/dri/renderD128"' ; } | tee -a /etc/pve/lxc/LXC_ID.conf
  
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
  Take a look at my Jellyfin VA-API guide for this: https://github.com/TheHellSite/archlinux_lxc/tree/main/jellyfin#jellyfin-va-api-hardware-accelerated-video-transcoding-run-as-root-user
