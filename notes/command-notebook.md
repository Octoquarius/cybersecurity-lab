# 📓 Command Notebook

Commands I use often. Add new ones here as you learn them.

## Network discovery
```bash
nmap -sn 192.168.x.0/24        # find live hosts (ping sweep)
arp-scan --localnet            # local network discovery via ARP
```

## Port / service scanning
```bash
nmap -sV -sC <ip>              # version + default scripts
nmap -p- <ip>                  # all 65535 ports (slow)
nmap -A <ip>                   # aggressive: OS + version + script + traceroute
```

## SMB / Windows enumeration
```bash
enum4linux-ng <ip>
smbclient -L //<ip>/           # list shares
crackmapexec smb <ip>
```

## Web
```bash
gobuster dir -u http://<ip> -w /usr/share/wordlists/dirb/common.txt
whatweb http://<ip>
nikto -h http://<ip>
```

## Metasploit
```bash
msfconsole
search <service>
use <exploit/path>
set RHOSTS <ip>
run
```

## My lab IPs
<!-- Isolated lab only — never write real-system IPs -->
- KALI (Host-Only): 192.168.___.___
- METASPLOITABLE: 192.168.___.___
