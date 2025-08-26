

**Design and Implementation of a pfSense Firewall with Suricata IDS/IPS in a Virtualized Homelab Environment**

## Objective

The objective of this project was to build and configure a homelab that simulates an enterprise perimeter defense environment. The project aimed to:

* Deploy **pfSense** as a firewall to enforce segmentation between attacker and victim networks.
* Install and configure **Suricata** as an Intrusion Detection and Prevention System (IDS/IPS).
* Generate realistic attack traffic from a Kali Linux attacker VM to a Windows victim VM.
* Detect and prevent attacks using Suricata, including Nmap scans, ICMP sweeps, and brute-force attempts with Hydra.
* Explore Suricata rule tuning with **SID Management** to convert specific categories from ALERT to DROP.
* Document challenges encountered, their resolutions, and final outcomes.

---

## Lab Setup

### Tools and Technologies

* **VirtualBox** as the virtualization platform.
* **pfSense** as the firewall/router.
* **Suricata** (pfSense package) as IDS/IPS.
* **Kali Linux** as the attacker machine.
* **Windows 10 VM** as the victim machine.

### Network Topology

```
[Kali Attacker: 192.168.10.100]
          |
   [OPT1 - pfSense - LAN]
          |
[Windows Victim: 192.168.1.100]
```

* pfSense WAN: NAT/Bridged (Internet access).
* pfSense LAN: Internal network (192.168.1.0/24) with victim.
* pfSense OPT1: Internal network (192.168.10.0/24) with attacker.

### VM Resources

* pfSense: 2 vCPUs, 2 GB RAM, Intel PRO/1000 NICs.
* Windows 10: 2–4 GB RAM, 2 vCPUs.
* Kali Linux: 2 vCPUs, 2 GB RAM.

---

## Configuration Process

### pfSense Setup

1. Installed pfSense from ISO in VirtualBox.
2. Configured WAN, LAN, and OPT1 interfaces.
3. Assigned:

   * LAN: 192.168.1.1/24
   * OPT1: 192.168.10.1/24
4. Enabled DHCP on both LAN and OPT1.
5. Firewall rules:

   * LAN: allow any outbound.
   * OPT1: allow any outbound.

### Suricata Setup

1. Installed Suricata via pfSense Package Manager.
2. Enabled Suricata on the LAN interface (to monitor traffic destined for the victim).
3. Configured:

   * HOME\_NET = victim subnet/host.
   * EXTERNAL\_NET = !\$HOME\_NET.
4. Enabled rule categories:

   * emerging-scan.rules
   * emerging-icmp.rules
   * emerging-ssh.rules
   * emerging-ftp.rules
5. Enabled Inline IPS Mode (netmap driver supported).
6. Created **dropsid.conf** with:

   ```
   emerging-scan.rules
   emerging-icmp.rules
   emerging-ssh.rules
   emerging-ftp.rules
   ```
7. Assigned this drop list to LAN via SID Mgmt.
8. Restarted Suricata and confirmed rules were set to DROP.

---

## Attack Scenarios

### ICMP Ping Sweep

* From Kali: `ping 192.168.1.100`, `nmap -sn 192.168.1.100`.
* Suricata detected ICMP echo requests.
* With DROP enabled, pings were blocked silently.

### Nmap Port Scans

* From Kali:

  ```bash
  sudo nmap -sS 192.168.1.100
  sudo nmap -sV 192.168.1.100
  sudo nmap -A 192.168.1.100
  ```
* Suricata triggered rules including:

  * ET SCAN Nmap TCP Scan
  * ET SCAN Nmap Scripting Engine
* With DROP, ports appeared as filtered.

### Hydra SSH Brute-Force

* Windows victim configured with OpenSSH server.
* User created: `hydra:password`.
* Hydra attack:

  ```bash
  hydra -l hydra -P /usr/share/wordlists/rockyou.txt ssh://192.168.1.100 -t 4
  ```
* Suricata alerted on brute-force attempts and dropped repeated login traffic.

### Hydra FTP Brute-Force

* Windows victim configured with IIS FTP server.
* Hydra attack:

  ```bash
  hydra -l hydra -P /usr/share/wordlists/rockyou.txt ftp://192.168.1.100 -t 4
  ```
* Suricata triggered brute-force rules and dropped malicious connections.

---

## Results

* In IDS Mode (Legacy), only alerts were logged.
* In Inline IPS Mode with dropsid.conf, Suricata actively blocked malicious traffic.
* Alerts included Nmap scans, ICMP pings, SSH brute-force, and FTP brute-force.
* Nmap output changed to show filtered ports.
* Hydra sessions were interrupted after Suricata began blocking.
* Suricata logs (`alerts.log` and `block.log`) confirmed detection and prevention.

---

## Troubleshooting and Resolutions

### 1. pfSense Performance Issues

* Issue: Windows VM was extremely slow with 100% CPU usage.
* Resolution: Switched Paravirtualization Interface to **Hyper-V**, enabled Nested Paging, allocated 2 vCPUs and 2–4 GB RAM, enabled 3D acceleration, and installed VirtualBox Guest Additions.

### 2. Windows Firewall Blocking Ping

* Issue: Victim could not be pinged from Kali even after enabling ICMP rule.
* Cause: Windows network profile was Public; firewall blocked ICMP on Public profile.
* Resolution: For lab purposes, disabled Public profile firewall to allow ping. Noted that reclassifying to Private or creating custom ICMP rules would also resolve this.

### 3. Suricata Not Logging Attacks

* Issue: Initial Nmap scans did not trigger alerts.
* Cause: Attacker and victim were on same LAN subnet, bypassing pfSense.
* Resolution: Segmented attacker into OPT1 subnet. Ensured traffic traversed pfSense.

### 4. Suricata Only Alerting, Not Blocking

* Issue: Alerts were generated but no traffic was blocked.
* Cause: Suricata was in Legacy Mode and default rules were ALERT-only.
* Resolution: Switched to Inline IPS mode and created a **dropsid.conf** to convert categories (e.g., emerging-scan.rules) from ALERT to DROP.

### 5. dropsid.conf Syntax

* Issue: Used incorrect identifiers in dropsid.conf (e.g., ET-emergingthreats-scan).
* Resolution: Corrected syntax to use exact rule filenames (e.g., emerging-scan.rules).

### 6. Hydra Failing with FTP

* Issue: Hydra produced connection errors when targeting FTP.
* Cause: No FTP service running on victim.
* Resolution: Installed IIS FTP service, allowed port 21 in Windows firewall, then re-ran Hydra successfully.

### 7. Inline IPS Verification

* Issue: Difficulty confirming Suricata drops.
* Resolution: Verified with Nmap. Ports reported as filtered indicated drops. Confirmed in Suricata rules tab that affected rules were set to DROP.

---

## Conclusion

This homelab successfully demonstrated deployment and configuration of a firewall and IDS/IPS system using pfSense and Suricata. The lab:

* Implemented network segmentation with pfSense (separate LAN and OPT1 subnets).
* Installed and tuned Suricata in Inline IPS mode.
* Used SID Management to enforce DROP actions for scan, ICMP, SSH, and FTP categories.
* Detected and blocked real-world attack scenarios (Nmap, Hydra, ping sweeps).

### Key Takeaways

* IDS placement must ensure attacker traffic traverses the sensor.
* Default Suricata rules are ALERT-only and require SID Mgmt for DROP conversion.
* Inline IPS depends on using netmap-supported NIC drivers (e.g., em in VirtualBox).
* Windows firewall profiles significantly affect testing.

This project provides a complete demonstration of perimeter security principles and blue team defense techniques. It is suitable for inclusion in a professional portfolio or GitHub repository as evidence of practical network security skills.
