# Proxmox VE Homelab — Bare-Metal Installation and Host Configuration

## Project overview

This project documents the installation and initial configuration of **Proxmox Virtual Environment (VE)** on a dedicated desktop PC. The goal is to create a remotely managed virtualization host for hands-on practice with Linux, Windows Server, networking, and IT infrastructure administration.

**Current status:** Proxmox is installed, reachable over the local network, updated, and running its post-update kernel. **No virtual machines have been created yet.** VM deployment is planned for the next phase.

## Objectives

- Install a bare-metal Type 1 hypervisor on a dedicated PC.
- Configure a predictable management IP address and wired network connection.
- Administer the host through Proxmox's HTTPS web interface from other devices on the LAN.
- Configure appropriate APT repositories for a non-production homelab.
- Apply available system updates and verify the running kernel.
- Prepare the environment for future VMs and networking labs.

## Hardware and software

| Component | Configuration |
| --- | --- |
| Hypervisor | Proxmox VE 9.2.2 (version shown in the web UI during setup) |
| Host CPU | Intel Core i5-6500 @ 3.20 GHz |
| CPU resources | 4 physical cores / 4 threads |
| Memory | ~16 GB installed (15.497 GiB reported) |
| Storage | ~209 GiB total Proxmox-managed capacity reported in dashboard |
| Installation media | USB drive prepared with Rufus and the Proxmox VE installer ISO |
| Host connection | Ethernet to home router |
| Remote management clients | Desktop browser and Android phone browser on the local network |

### Default storage layout

| Storage | Approximate size shown | Intended use |
| --- | ---: | --- |
| `local` (directory) | 67.728 GiB | ISO images, templates, backups (where enabled) |
| `local-lvm` (LVM-thin) | 141.227 GiB | Virtual machine disks |

> These are Proxmox-reported storage pools; they are not necessarily separate physical drives. Actual free capacity changes with use.

## Network design

```text
                 Home router
               192.168.180.1
                /          \
        Ethernet            Wi-Fi
            |                 |
     Proxmox VE host     Phone / laptop
     192.168.180.10      Web browser
            |
        HTTPS :8006
```

| Setting | Value |
| --- | --- |
| Management interface | `nic0` (`e1000e` driver shown during installation) |
| Host IP address | `192.168.180.10/24` |
| Subnet mask | `255.255.255.0` |
| Default gateway | `192.168.180.1` |
| DNS server | `192.168.180.1` |
| Management URL | `https://192.168.180.10:8006` |

The address was configured statically in the Proxmox installer. The management web interface is accessed from devices on the same reachable LAN; a client device can use Wi-Fi while the server uses Ethernet.

> **Security note:** `192.168.180.10` is a private LAN address. This setup does **not** require port forwarding or exposing port 8006 to the public internet. The browser may warn about Proxmox's default self-signed HTTPS certificate; verify the address before proceeding. Keep administrator passwords, tokens, and certificates out of this repository.

## Implementation steps

### 1. Prepare installation media

1. Downloaded the Proxmox VE installer ISO.
2. Used **Rufus** to write the installer to a USB drive.
3. Booted the dedicated desktop from the USB installer using its boot menu.
4. Used the graphical Proxmox VE installation workflow.

**Issue encountered — PXE over IPv4:** At one point the PC displayed `Start PXE over IPv4` rather than the installer. This indicated that firmware was attempting a network boot when it did not boot from the intended local media. Booting explicitly from the installer USB allowed installation to proceed.

### 2. Enable virtualization and install Proxmox

- Checked the PC's firmware settings and corrected a virtualization-related configuration issue encountered during the earlier setup attempt.
- Ran the **Install Proxmox VE (Graphical)** installer.
- Selected the target installation disk and completed the installer prompts.
- Configured the wired management network as shown above.
- Set an administrator password during the installation workflow and used the built-in `root` account for first login.

**Hostname note:** The installation screenshot showed `changproxmox.com` in the hostname field. Hostname cleanup and local DNS design can be revisited later; the verified management access path for this project uses the IP address.

> **Data-loss warning for anyone following this guide:** Installing Proxmox onto a selected disk can overwrite its existing partitions and data. Confirm the target disk and back up anything important first.

### 3. Verify first boot and remote web access

After installation and reboot, the host console displayed:

```text
https://192.168.180.10:8006/
```

Verified access to the Proxmox interface from an Android phone on Wi-Fi, and later from a desktop browser. Logged in with:

- **User:** `root`
- **Realm:** `Linux PAM standard authentication`
- **Password:** installation-defined administrator password (not documented here)

The initial dashboard showed the `pve` node **online**, with four CPU threads and approximately 16 GB RAM. Both `local` and `local-lvm` storage were available.

**Issue encountered — mobile interface limitations:** The phone's mobile interface did not expose all repository controls conveniently, and the desktop-site view was difficult to operate. Switched to a full desktop browser for repository configuration.

### 4. Configure APT repositories

The initial repository screen warned that enterprise repositories were enabled without a subscription. The package-database refresh also previously reported an error.

In **`pve` → `Updates` → `Repositories`**:

1. Disabled the Proxmox VE enterprise repository (`pve-enterprise`).
2. Disabled the Ceph enterprise repository (Ceph storage is not configured for this lab).
3. Kept the Debian Trixie and security repositories enabled.
4. Added and enabled the **Proxmox VE no-subscription** repository.

The resulting Proxmox VE repository was:

```text
http://download.proxmox.com/debian/pve
Suite: trixie
Component: pve-no-subscription
```

> The no-subscription channel is suitable for a personal lab, but Proxmox warns that it is not recommended for production use. Repository configuration should be reviewed before using this build in a business-critical environment.

### 5. Refresh and upgrade packages

From **`pve` → `Updates`**:

1. Refreshed package information.
2. Confirmed the refresh task ended with **`TASK OK`**.
3. Started the host upgrade through the web interface.
4. During installation, a package changelog opened in a terminal pager; pressed **`q`** to exit the pager and allow the process to continue.
5. Confirmed the upgrade finished with:

```text
Your System is up-to-date
```

The upgrade also reported that a new kernel had been installed and recommended a reboot to activate it.

### 6. Reboot and verify the kernel

Rebooted the host and confirmed the Proxmox web interface was reachable again at the same LAN IP.

Opened **`pve` → `Shell`** and ran:

```bash
uname -r
```

Observed output:

```text
7.0.14-28-pve
```

This confirmed the host had booted into the updated Proxmox kernel.

## Validation checklist

- [x] Installation USB prepared and bootable
- [x] Proxmox VE installed to dedicated hardware
- [x] Ethernet management interface detected
- [x] Static management IP configured
- [x] Management URL accessible from another LAN device
- [x] Root login through Linux PAM verified
- [x] Host node `pve` online
- [x] `local` and `local-lvm` storage available
- [x] Enterprise APT repositories disabled
- [x] Proxmox VE no-subscription repository enabled
- [x] Package metadata refresh completed successfully
- [x] Host updates completed
- [x] Server rebooted successfully after kernel update
- [x] Running kernel verified with `uname -r`
- [ ] Create first VM (next phase)

## Troubleshooting and lessons learned

| Symptom | Cause / interpretation | Resolution |
| --- | --- | --- |
| `Start PXE over IPv4` on boot | Firmware attempted network boot instead of installer/local OS | Booted from the prepared Proxmox USB |
| Virtualization-related setup error during an earlier attempt | Firmware virtualization configuration needed attention | Corrected the relevant BIOS/UEFI setting before retrying |
| Package update error / enterprise repository warning | Enterprise repositories enabled with no subscription | Disabled enterprise entries and enabled no-subscription channel |
| Repository settings difficult to manage on Android | Full desktop management workflow not well suited to a narrow screen | Used desktop browser instead |
| Upgrade seemed paused at `(press q to quit)` | Changelog pager awaiting input | Pressed `q` to return to upgrade workflow |
| Kernel update requires activation | New kernel installed, old kernel still active until reboot | Rebooted and checked `uname -r` |

## Skills demonstrated

- Bare-metal hypervisor installation and initial configuration
- BIOS/UEFI boot troubleshooting and virtualization prerequisites
- IPv4 addressing, CIDR, gateways, DNS, and wired connectivity
- Remote web-based systems administration
- Linux APT repository configuration and patch management
- Command-line verification of running kernel version
- Reading administrative task logs and resolving setup issues

## Next phase: Deploy virtual machines

**Not completed in this phase.** Planned work includes:

1. Upload an Ubuntu Server LTS ISO to `local` storage.
2. Create an Ubuntu Server VM with a modest CPU/RAM allocation and a virtual disk on `local-lvm`.
3. Attach the VM to Proxmox's Linux bridge (`vmbr0`) and validate LAN connectivity.
4. Install and update Ubuntu Server; configure SSH access.
5. Explore additional Windows Server, Active Directory, DNS, SMB, and network services labs as resources allow.

---

**Project milestone:** A dedicated Proxmox VE host was installed, patched, rebooted, and verified for remote management. Virtual machine creation will be documented as a separate follow-on phase.
