# SOC Homelab: Infrastructure First

A small, hand-built network lab for learning SOC analyst fundamentals: **what** is being protected, **how** it is protected, and **why** solid network and OS knowledge matters when reading security logs.

Everything runs as virtual machines in Oracle VirtualBox on an 8 GB Windows laptop. No commercial firewall appliance is used. The router and firewall are a plain Ubuntu Server configured by hand, so every function (routing, NAT, DHCP, DNS, filtering, logging) is visible and understood.

## Architecture

```
                      Internet
                         |
              VirtualBox NAT (10.0.2.0/24)
                         |
                   enp0s3 (WAN, DHCP)
                 +---------------------+
                 |        rtr01        |
                 |  Ubuntu 26.04 LTS   |
                 | routing, NAT, DHCP, |
                 | DNS, nftables FW    |
                 +---------------------+
          enp0s8 10.10.10.1     enp0s9 10.10.20.1
                 |                       |
       lab-servers network      lab-clients network
         10.10.10.0/24             10.10.20.0/24
                 |                       |
             srv01                    kali01
          10.10.10.10              10.10.20.10 (DHCP reservation)
        nginx + SSH               user now, attacker later
        (the asset)
```

| Machine | OS | RAM | Address | Role |
|---|---|---|---|---|
| rtr01 | Ubuntu Server 26.04.1 LTS | 1 GB | WAN DHCP, 10.10.10.1, 10.10.20.1 | Router, firewall, DHCP, DNS |
| srv01 | Ubuntu Server 26.04.1 LTS | 1 GB | 10.10.10.10 (static) | Web server and SSH: the asset to protect |
| kali01 | Kali Linux 2026.2 | 2 GB | 10.10.20.10 (DHCP reservation) | Client, later attacker |

Total RAM for the lab is about 4 GB, which fits on an 8 GB laptop.

## Why it is built this way

- **Separate networks for servers and clients.** Real companies segment their networks so a compromised laptop cannot reach everything. Every packet between the two networks must pass through rtr01, which makes it the single point where traffic is filtered and logged.
- **Router built by hand instead of pfSense.** A ready-made firewall hides the moving parts. Building it from Ubuntu means configuring each function separately and learning what each one does.
- **Default deny.** The firewall blocks everything, then allows only what the "business" needs. Anything not explicitly allowed is dropped and logged.
- **Logs as evidence.** Every blocked packet is logged with source, destination, protocol and port. These logs are what a SOC analyst reads during an investigation.

## What was built

### rtr01: the router and firewall

| Function | How | Config file |
|---|---|---|
| Interface addresses | netplan | [`configs/rtr01/50-cloud-init.yaml`](configs/rtr01/50-cloud-init.yaml) |
| Routing between networks | `net.ipv4.ip_forward=1` | [`configs/rtr01/99-router.conf`](configs/rtr01/99-router.conf) |
| DHCP and DNS for the lab | dnsmasq, `lab.local` domain, reservation for kali01 | [`configs/rtr01/lab.conf`](configs/rtr01/lab.conf) |
| NAT to the internet | nftables masquerade | [`configs/rtr01/nftables.conf`](configs/rtr01/nftables.conf) |
| Firewall with logging | nftables, default deny | [`configs/rtr01/nftables.conf`](configs/rtr01/nftables.conf) |

### srv01: the asset

- Static address 10.10.10.10, gateway and DNS 10.10.10.1 ([`configs/srv01/netplan-equivalent.yaml`](configs/srv01/netplan-equivalent.yaml)).
- nginx serving a "Lab Corp intranet" page ([`configs/srv01/index.html`](configs/srv01/index.html)).
- OpenSSH for administration.
- Attack surface checked with `ss -tlnp`: only ports 22 (SSH) and 80 (HTTP) listening.

### kali01: the client

- Gets 10.10.20.10 from rtr01's DHCP through a MAC-address reservation.
- Default `kali/kali` password changed.

## Firewall policy

| From | To | Allowed | Why |
|---|---|---|---|
| Clients | srv01 | TCP 80, TCP 22, ping | Users reach the intranet; admins use SSH |
| Clients and Servers | rtr01 | DNS (53), DHCP (67), ping | Basic network services |
| Clients | rtr01 | TCP 22 | Router administration |
| Clients and Servers | Internet | Blocked (temporary rule for updates) | The lab is isolated by default |
| Anything else | Anything | Dropped and logged | Default deny |

## Validation

Tests run from kali01 after the firewall was loaded:

| Test | Expected | Result |
|---|---|---|
| `curl http://10.10.10.10` | Lab Corp page | Page returned (allowed) |
| `ping -c 3 srv01.lab.local` | Replies | 3/3 replies; DNS and ping work (allowed) |
| `nc -zv -w 3 10.10.10.10 8080` | Timeout | Connection timed out (blocked) |
| `ping -c 3 8.8.8.8` | No replies | 100% packet loss (blocked) |
| `ssh anis@srv01.lab.local` | Login | Logged in through the router (allowed) |

![Allowed vs blocked tests from Kali](screenshots/06-allow-vs-block-tests.png)

## Log analysis

Blocked traffic was reviewed on rtr01 with:

```
sudo journalctl -k | grep NFT-
```

![Firewall drop log](screenshots/07-firewall-drop-log.png)

### Finding 1: most log lines were background noise, not the tests

Most entries were not caused by the tests. They were the lab machines trying to reach the internet on their own:

| Log fields | Meaning |
|---|---|
| `SRC=10.10.10.10 DST=140.248.130.49 PROTO=TCP DPT=443 SYN` | srv01 trying to open an HTTPS connection to the internet, most likely an update check |
| `SRC=10.10.10.10 / 10.10.20.10 PROTO=UDP DPT=123` | srv01 and kali01 trying to sync their clocks (NTP) |
| `IN=enp0s3 PROTO=ICMPv6 TYPE=134` | Router advertisement from the VirtualBox NAT network: harmless |

**Lesson:** an analyst first learns what normal traffic looks like (a baseline), so that unusual traffic stands out.

### Finding 2: one connection attempt, three log lines

Filtering for the test port (`grep DPT=8080`) showed three entries one second apart, all with the same source port (`SPT=44976`). The client retransmitted its SYN twice after getting no answer. One attempt can produce several log lines; a port scan would instead show the same source hitting many different destination ports in quick succession.

![Filtered log for port 8080](screenshots/08-log-filtered-8080.png)

### Finding 3: "timed out" vs "refused"

From the attacker's side, a port blocked by the firewall times out (the packet was silently dropped). A closed port on a reachable host is refused (the host answered with a reset). The difference tells an attacker, and an analyst, whether a firewall is in the path.

## Troubleshooting log

Problems hit during the build and how they were solved:

| Problem | Cause | Fix |
|---|---|---|
| netplan error `expected mapping` | YAML file incomplete; YAML requires exact space indentation | Retyped the file with 2-space indentation |
| srv01 could ping 8.8.8.8 but not resolve names | dnsmasq had no upstream DNS server | Added `no-resolv` and `server=10.0.2.3` (VirtualBox NAT DNS) |
| dnsmasq would not start | Typo in the `dhcp-host` line (wrong MAC, extra space) | Removed the line with `sed`, re-added it exactly |
| Kali VM would not boot after renaming | The `.vdi` disk file was missing from the renamed VM folder | Re-extracted the Kali image and re-added the VM |
| Firewall syntax risk | A typo in nftables can lock out traffic | Checked with `nft -c -f` before loading |

## Skills demonstrated

- Linux server administration (Ubuntu Server, systemd, netplan, sysctl)
- IP addressing, subnetting and routing between network segments
- NAT, DHCP and DNS configuration (dnsmasq)
- Host-based and network firewalling with nftables, default-deny design
- Firewall log analysis: reading SRC, DST, PROTO, DPT and TCP flags
- Network troubleshooting layer by layer (own IP, gateway, internet by IP, DNS)
- Virtualization with VirtualBox (internal networks, snapshots)

## Known gaps and next steps

- **Time sync is blocked.** NTP is dropped by the firewall, so clocks will drift and log timestamps will become hard to compare. Planned fix: serve time from rtr01 or allow NTP out.
- **Next stages:**
  1. Linux OS fundamentals: users, permissions, processes, services, log locations
  2. Building and understanding the web service
  3. Hardening: SSH and firewall improvements
  4. Monitoring: tcpdump and log review
  5. Attack and investigate: nmap, hydra and nikto from kali01, then investigating the evidence
  6. Detection: Suricata IDS and fail2ban
- **Later, with a RAM upgrade:** Wazuh SIEM, a Windows endpoint with Sysmon.

## More screenshots

| | |
|---|---|
| dnsmasq running on rtr01 | ![](screenshots/01-dnsmasq-running.png) |
| kali01 gets its reserved address | ![](screenshots/02-kali-dhcp-address.png) |
| kali01 reaches srv01 by name | ![](screenshots/03-kali-reaches-srv01.png) |
| SSH from kali01 to srv01 through the router | ![](screenshots/04-ssh-kali-to-srv01.png) |
| Firewall checked and loaded | ![](screenshots/05-firewall-loaded.png) |
