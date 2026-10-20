# Personal Cybersecurity Lab: Kali Linux + OWASP Juice Shop

A safe, isolated penetration-testing lab built on VirtualBox. A Kali Linux attacker VM hosts an intentionally vulnerable web application (OWASP Juice Shop running in Docker) and uses a segmented **NAT + Host-Only** network so that lab traffic stays away from real systems.

Built as **Task 2** of the Cybersecurity & Ethical Hacking Internship at Maincrafts Technology.

## What this lab contains

| Component | Details |
|---|---|
| Virtualization | Oracle VirtualBox |
| Attacker machine | Kali Linux (kali-rolling), 2 CPUs, 4 GB RAM |
| Vulnerable target | OWASP Juice Shop in Docker, published on port 3000 |
| Network | NAT adapter (updates and downloads) + Host-Only adapter (lab traffic) |
| Tools validated | Nmap 7.99, Burp Suite Community v2026.3.2, Wireshark |

## Architecture

![Lab architecture](screenshots/01_architecture.png)

- **NAT adapter (eth0, 10.0.2.15/24):** outbound internet access only, used for `apt` updates and for pulling the Docker image.
- **Host-Only adapter (eth1, 192.168.56.10/24):** isolated lab network on `192.168.56.0/24`. The host adapter is `192.168.56.1`.
- **Target:** the Juice Shop container runs on the Kali VM and is reachable at `http://192.168.56.10:3000` or `http://localhost:3000`.

## Lab configuration

| Item | Value |
|---|---|
| Kali Adapter 1 | NAT (eth0), 10.0.2.15/24 |
| Kali Adapter 2 | Host-Only (eth1), 192.168.56.10/24 (static) |
| Host-Only network | VirtualBox Host-Only Ethernet Adapter #2, 192.168.56.1/24, DHCP disabled |
| Target service | OWASP Juice Shop (Docker), TCP port 3000 |

## Setup steps

### 1. Create the Host-Only network
Install VirtualBox, then create a Host-Only network in the VirtualBox network manager. This lab uses Adapter #2 (`192.168.56.1/24`).

![Host-only networks](screenshots/02_host_only_networks.png)

### 2. Create the Kali VM
Allocate 2 CPUs and 4096 MB RAM, then set Adapter 1 to NAT and Adapter 2 to Host-only.

| CPU and memory | Network adapters |
|---|---|
| ![CPU](screenshots/03a_kali_cpu.png) ![RAM](screenshots/03b_kali_ram.png) | ![NAT](screenshots/04a_adapter1_nat.png) ![Host-only](screenshots/04b_adapter2_host_only.png) |

Update the system and create the evidence folders:

```bash
sudo apt update && sudo apt upgrade -y
mkdir -p ~/lab/{scans,notes,pcaps,screenshots}
ls ~/lab
```

![apt update](screenshots/05_apt_update.png)
![lab folders](screenshots/06_lab_folders.png)

### 3. Deploy the target (OWASP Juice Shop)

```bash
sudo apt install -y docker.io
sudo systemctl enable --now docker
sudo docker pull bkimminich/juice-shop
sudo docker run -d -p 3000:3000 --name juice bkimminich/juice-shop
sudo docker ps
```

![docker running](screenshots/07_docker_running.png)
![Juice Shop in browser](screenshots/08_juice_shop_browser.png)

<!-- Add after taking the screenshot (VirtualBox main window showing the Kali VM running):
![VM list](screenshots/09_vm_list.png)
-->

### 4. Verify connectivity
The Host-Only network has DHCP disabled, so `eth1` was given a static address (`192.168.56.10/24`).

```bash
ip a
ping -c 4 192.168.56.10
nmap -sn 192.168.56.0/24
```

![ip a](screenshots/10_ip_config.png)

Results: 4 of 4 ping replies with 0% packet loss. The Nmap sweep found 2 hosts up: the VirtualBox host adapter (`192.168.56.1`) and the Kali VM (`192.168.56.10`).

![ping and nmap](screenshots/11_ping_nmap_sn.png)

### 5. Validate with Nmap, Burp Suite and Wireshark

```bash
nmap -sV 192.168.56.10 -oN ~/lab/scans/target_scan.txt
```

<!-- Add after taking the screenshot (nmap -sV output showing port 3000 open):
![nmap -sV](screenshots/12_nmap_sv.png)
-->

**Burp Suite:** the browser was routed through Burp's proxy and HTTP requests to the lab host were intercepted with Intercept on.

![Burp intercept](screenshots/13_burp_intercept.png)

**Wireshark:** because the target runs on the same VM as Kali, the traffic stays on the loopback interface (`lo`). The capture below uses the filter `tcp.port == 3000` and Follow TCP Stream to show the plain-text HTTP request and response.

![Wireshark](screenshots/14_wireshark_loopback.png)

## Lab isolation and safety

- The Kali VM has only two adapters (NAT and Host-Only) and no bridged adapter, so Juice Shop is not exposed to the home or college network.
- NAT is used only for updates and downloads; VirtualBox NAT does not accept unsolicited inbound connections by default.
- Juice Shop runs inside a Docker container, adding separation from the Kali operating system.
- All scanning was limited to `192.168.56.0/24` and the local host.
- VM snapshots can be taken before testing to restore a clean state.

## Troubleshooting log

| Issue | Cause | Fix |
|---|---|---|
| `sudo: upgrade: command not found` | The second `apt` was left out of the chained command | Re-ran `sudo apt upgrade -y` |
| `unknown shorthand flag: '-' in -p` | Missing space between `-d` and `-p` | Re-ran with `-d -p 3000:3000` |
| `eth1` had no IP address | DHCP disabled on Host-Only Adapter #2 | Assigned static `192.168.56.10/24` |
| `ping: ...: Name or service not known` | Included the `/24` suffix in the ping target | Pinged `192.168.56.10` (plain IP) |

## What I learned

- Building an isolated lab with VirtualBox NAT and Host-Only networking.
- Deploying an intentionally vulnerable web application with Docker.
- Verifying connectivity and discovering hosts with `ip`, `ping` and Nmap.
- Intercepting HTTP traffic with Burp Suite and analysing packets and TCP streams with Wireshark.
- Why same-host traffic appears on the loopback interface rather than `eth1`.

## Responsible use

This lab is for learning. Only test systems you own or have explicit permission to test. Juice Shop is intentionally vulnerable: keep it on the Host-Only network and never expose it to the internet.
