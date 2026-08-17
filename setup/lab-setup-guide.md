# Cybersecurity Home Lab — From-Scratch Setup Guide

> An isolated attack/defense lab on a Windows 11 Pro host (64 GB RAM) using VMware
> Workstation Pro. Every step is checkable — tick the boxes as you go.

---

## Architecture

```
[HOST — Windows 11 Pro]
   │
   ├─ VMware Workstation Pro
   │     ├─ Kali VM  (ATTACKER)
   │     │     ├─ Adapter 1 → NAT        (internet: apt, tool updates)
   │     │     └─ Adapter 2 → Host-Only  (to talk to the victim)
   │     │
   │     └─ Metasploitable VM  (VICTIM — intentionally vulnerable)
   │           └─ Adapter → Host-Only    (FULLY ISOLATED: no internet, no home LAN)
   │
   └─ (physical ethernet / WiFi — home network)
         │
         └─ [OLD WINDOWS LAPTOP]  (realistic enumeration target — you own it)
```

**The two targets do different jobs:**
- **Metasploitable VM** → intentionally vulnerable playground; exploitation practice. Host-Only means it poses zero risk to your host.
- **Physical Windows** → realistic, patched target; teaches reconnaissance and enumeration discipline. Not quick wins — surface mapping.

---

## Part 0 — Prerequisites

- [ ] Confirm at least **60–80 GB free disk** on the host (two VMs + snapshots).
- [ ] Make **file extensions visible** in Windows: File Explorer → View → tick "File name extensions". (Needed to find the `.vmx` file.)
- [ ] Install **7-Zip** (to extract archives): 7-zip.org.
- [ ] Note: WSL is not installed on this machine, so VMware will not conflict with anything. No Windows Features changes needed.

---

## Part 1 — Install VMware Workstation Pro

It's now free — no license key required for personal/commercial/education use.

- [ ] Create a free account (or sign in) at **support.broadcom.com**.
- [ ] My Dashboard → My Downloads → "Free Software Downloads" → search **VMware Workstation Pro**.
- [ ] Download the latest build (calendar naming: **26H1** = first half of 2026).
- [ ] Run the installer. No license key is requested; if prompted, choose **"Personal Use"** / the free option.
- [ ] Launch VMware when done.

> The free edition has no official Broadcom support; use community forums for issues. Fine for a lab.

---

## Part 2 — Configure virtual networks (BEFORE the VMs)

Getting networking right up front saves rework later. Goal: a **Host-Only** network ready with DHCP.

- [ ] VMware → **Edit → Virtual Network Editor** (may need "Change Settings" for admin rights).
- [ ] Check whether **VMnet1 (Host-only)** exists. If not: "Add Network" → VMnet1 → **Host-only**.
- [ ] For VMnet1:
  - [ ] Type: **Host-only**
  - [ ] "Connect a host virtual adapter to this network" **checked**
  - [ ] "Use local DHCP service to distribute IP addresses" **checked**
  - [ ] Note the Subnet IP (e.g. `192.168.171.0` — yours may differ; use your own value).
- [ ] **VMnet8 (NAT)** should already exist; internet comes through it. Leave it alone.
- [ ] Save with OK.

**Tell the three network modes apart:**

| Mode | VM can reach | Reachable from outside | Use for |
|---|---|---|---|
| **Bridged** | Whole home LAN + internet | Yes | Only Kali's optional link to the physical Windows target |
| **NAT** | Internet (outbound) | No | Kali's internet adapter |
| **Host-Only** | Host + same-segment VMs only | No | Kali's victim adapter + Metasploitable |

---

## Part 3 — Kali attacker VM

We use the prebuilt VMware image (not a manual ISO install — much faster, VMware Tools baked in).

### 3.1 Download and verify
- [ ] **kali.org/get-kali → Virtual Machines** tab → **VMware** button; download the `.7z` image (~3.2 GB, filename like `kali-linux-2026.2-vmware-amd64.7z`).
- [ ] Verify the checksum (PowerShell):
  ```powershell
  certutil -hashfile kali-linux-2026.2-vmware-amd64.7z sha256
  ```
- [ ] Compare the SHA256 to the value on kali.org. **If it doesn't match, re-download.**

### 3.2 Extract and import
- [ ] Right-click the `.7z` → 7-Zip → Extract Here. (On Win11 it may be under "Show more options".)
- [ ] Find the **`.vmx`** file in the extracted folder.
- [ ] VMware → **Open a Virtual Machine** → select that `.vmx`.
- [ ] On launch, answer "moved or copied?" with **"I Copied It"**.

### 3.3 Resources
- [ ] VM → **Edit virtual machine settings**:
  - [ ] Memory: **8 GB**
  - [ ] Processors: **4 cores**
- [ ] (Network adapters are configured in Part 5 — leave them for now.)

### 3.4 First boot
- [ ] Start the VM. Login: user **`kali`**, password **`kali`**.
- [ ] Change the password immediately:
  ```bash
  passwd
  ```
- [ ] Update:
  ```bash
  sudo apt update && sudo apt full-upgrade -y
  ```
- [ ] Handy extra tools (optional):
  ```bash
  sudo apt install -y enum4linux-ng crackmapexec seclists
  ```

### 3.5 Snapshot
- [ ] VM → Snapshot → **Take Snapshot** → "kali-clean". (Revert here if you break something.)

---

## Part 4 — Metasploitable victim VM (isolated)

Intentionally vulnerable Linux. **Never expose it to the internet or an untrusted network** — Host-Only only.

### 4.1 Download
- [ ] **sourceforge.net/projects/metasploitable** → `metasploitable-linux-2.0.0.zip` (~800 MB).
- [ ] Extract the `.zip`. It contains `.vmdk` + `.vmx` files.

### 4.2 Import
- [ ] VMware → **Open a Virtual Machine** → select the **`.vmx`** in the Metasploitable folder.
- [ ] On launch, "moved or copied?" → **"I Copied It"**.

### 4.3 Isolate the network (critical step)
- [ ] VM → **Edit virtual machine settings** → Network Adapter:
  - [ ] **Custom: Specific virtual network** → select **VMnet1 (Host-only)**.
  - [ ] Remove any other adapter — keep a single Host-Only adapter.
- [ ] Memory: 512 MB – 1 GB is enough.

### 4.4 First boot
- [ ] Start it. Login: **`msfadmin`** / **`msfadmin`**.
- [ ] Find its IP:
  ```bash
  ifconfig
  ```
  (It gets a `192.168.x.y` from Host-Only DHCP — this is **VICTIM_IP**. Write it down.)

### 4.5 Snapshot
- [ ] Take a snapshot → "msf-clean".

---

## Part 5 — Give Kali two adapters (NAT + Host-Only)

Kali needs both internet (tool updates) and reach to the isolated victim. Solution: two adapters.

- [ ] **Power off** the Kali VM (to add an adapter).
- [ ] VM → **Edit virtual machine settings** → **Add → Network Adapter** (a second one).
- [ ] Set the two adapters:
  - [ ] **Adapter 1** → NAT (internet)
  - [ ] **Adapter 2** → Custom → **VMnet1 (Host-only)** (victim)
- [ ] Start Kali, check the interfaces:
  ```bash
  ip a
  ```
  You should see two interfaces (usually `eth0` = NAT, `eth1` = Host-Only). The Host-Only interface's IP must be in the same `192.168.x.` block as the victim.

---

## Part 6 — Physical Windows target

A normal Windows machine on your network. We do **not** add vulnerabilities — we just open a visible surface for practice and close it afterward.

- [ ] Confirm this machine is **yours**. (Never scan someone else's device, even with permission.)
- [ ] Two ways for Kali to reach it:
  - **Easy way:** add a third **Bridged** adapter to Kali → Kali joins the home LAN and can reach the Windows LAN IP.
  - **Alternative:** connect the Windows machine to the host directly via ethernet and build an isolated segment (cleaner but needs a cable/adapter).
- [ ] For a lab session on Windows (optional, instructive surface):
  - [ ] Enable file/printer sharing (SMB visibility)
  - [ ] Enable RDP (Remote Desktop)
  - [ ] Create a simple-password local user for testing
  - [ ] **Turn all of this back off when the session ends.**

> What you learn here: not exploitation, but **enumeration** — open ports, service versions, SMB shares, RDP config.

---

## Part 7 — Verification and first exercises

### 7.1 Verify isolation
- [ ] From **Metasploitable**, reach the internet — this must **FAIL** (proof of isolation):
  ```bash
  ping 8.8.8.8      # should get NO reply
  ```
- [ ] From **Kali**, reach the victim — this must **SUCCEED**:
  ```bash
  ping VICTIM_IP
  ```
- [ ] From **Kali**, reach the internet — this must **SUCCEED** (NAT):
  ```bash
  ping kali.org
  ```

### 7.2 First scan (Metasploitable)
```bash
nmap -sn 192.168.x.0/24        # find live hosts
nmap -sV -sC VICTIM_IP         # service versions + default scripts
nmap -p- VICTIM_IP             # all ports (slow)
```

### 7.3 First "box root" (Metasploitable — for motivation)
The classic vsftpd backdoor on Metasploitable:
```bash
msfconsole
search vsftpd_234_backdoor
use exploit/unix/ftp/vsftpd_234_backdoor
set RHOSTS VICTIM_IP
run
```
If a shell drops, `whoami` → `root`. Your first full takeover. This one is *intentionally* easy; the real skill is in 7.4.

### 7.4 Physical Windows enumeration
```bash
nmap -sV -sC -O WINDOWS_IP
enum4linux-ng WINDOWS_IP        # SMB / shares / users
smbclient -L //WINDOWS_IP/      # list shares
```
Goal: map the machine's surface completely. Take notes — which port, which version, which share.

---

## Part 8 — Snapshot discipline

- [ ] Take a snapshot **before** trying anything risky.
- [ ] If Kali breaks → revert to "kali-clean".
- [ ] If Metasploitable breaks → revert to "msf-clean" (or reset the VM).
- [ ] Once a month, update Kali and take a fresh "kali-clean" snapshot.

---

## Part 9 — Safety and legal boundaries

- **Only in your own lab.** Metasploitable, TryHackMe, HackTheBox, authorized bug bounty programs — these are fair game.
- **Physical target = only your machine.** Don't even run `nmap` against another device on the network (family, roommate).
- **Unauthorized scanning is a crime in Turkey.** What ends careers in this field is not lack of skill — it's crossing that line once.
- Even if Metasploitable is compromised, Host-Only means it has nowhere to go — so never switch it to Bridged.

---

## Appendix — Quick reference

**Default credentials**
| Machine | User | Password |
|---|---|---|
| Kali | `kali` | `kali` (change immediately) |
| Metasploitable | `msfadmin` | `msfadmin` |

**IP notebook** (fill in your own values)
- KALI_IP (Host-Only): `192.168.___.___`
- VICTIM_IP (Metasploitable): `192.168.___.___`
- WINDOWS_IP (physical): `192.168.___.___`

**Common commands**
```bash
ip a                          # Kali interfaces/IP
ifconfig                      # Metasploitable IP
nmap -sn <subnet>/24          # live-host discovery
nmap -sV -sC <ip>             # service + script scan
enum4linux-ng <ip>            # SMB enumeration
msfconsole                    # Metasploit framework
```

---

### Next stop
Once the lab is ready: **PortSwigger Web Security Academy** (free) → web security; **TryHackMe** paths in parallel. As you finish Metasploitable, add new victim VMs from VulnHub — all kept Host-Only.
