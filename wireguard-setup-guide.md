# 🛡️ Manual WireGuard Site-to-Site Configuration Guide (AWS & Google Cloud)

A complete, production-ready tutorial to launch Ubuntu instances, configure swap memory to prevent freezing, generate cryptographic keys, configure, and establish an encrypted peer-to-peer tunnel between **AWS EC2** and **Google Cloud (GCP)** using raw **WireGuard**, with detailed technical explanations of **WHY** each step, setting, and parameter is required.

---

## 📑 Table of Contents
1. [Network IP Plan & Architecture](#1-network-ip-plan--architecture)
2. [Why Ubuntu Server for Cloud & WireGuard?](#2-why-ubuntu-server-for-cloud--wireguard)
3. [Step 1: Launching Ubuntu Instances on AWS & Google Cloud](#3-step-1-launching-ubuntu-instances-on-aws--google-cloud)
   - [A. AWS EC2 Launch Flow & Field Explanations](#a-aws-ec2-launch-flow--field-explanations)
   - [B. Google Cloud Compute Engine Launch Flow & Field Explanations](#b-google-cloud-compute-engine-launch-flow--field-explanations)
4. [Step 2: Server Memory Optimization — Enabling 2GB SWAP (Fixes VS Code Freezing)](#4-step-2-server-memory-optimization--enabling-2gb-swap-fixes-vs-code-freezing)
5. [Step 3: Cloud Firewall Rules (UDP 51820)](#5-step-3-cloud-firewall-rules-udp-51820)
6. [Step 4: Install WireGuard on Both Nodes](#6-step-4-install-wireguard-on-both-nodes)
7. [Step 5: Generate Cryptographic Key Pairs](#7-step-5-generate-cryptographic-key-pairs)
8. [Step 6: Create Configuration Files (`wg0.conf`) & Parameter Breakdown](#8-step-6-create-configuration-files-wg0conf--parameter-breakdown)
9. [Step 7: Enable Kernel IP Forwarding & Start Service](#9-step-7-enable-kernel-ip-forwarding--start-service)
10. [Step 8: Test & Verify Tunnel Handshake](#10-step-8-test--verify-tunnel-handshake)
11. [Bonus: Configuring WireGuard on Windows Server](#11-bonus-configuring-wireguard-on-windows-server)
12. [Troubleshooting Common Issues](#12-troubleshooting-common-issues)

---

## 1. Network IP Plan & Architecture

```
┌──────────────────────────────────────────────┐              ┌──────────────────────────────────────────────┐
│                🟧 AMAZON AWS                 │              │            🟨 GOOGLE CLOUD (GCP)             │
│                                              │              │                                              │
│ • Hostname: `aws-node-01`                    │              │ • Hostname: `gcp-node-01`                    │
│ • Public IPv4: `16.170.200.50` (Example)     │◄──WireGuard─►│ • Public IPv4: `34.120.100.80` (Example)     │
│ • WG Listen Port: `51820/UDP`                │   Encrypted  │ • WG Listen Port: `51820/UDP`                │
│ • WG Tunnel IP: `10.0.0.1/24`                │    Tunnel    │ • WG Tunnel IP: `10.0.0.2/24`                │
└──────────────────────────────────────────────┘              └──────────────────────────────────────────────┘
```

---

### 📍 Deep Dive: What is the Tunnel IP (`10.0.0.1/24`), Where Does It Come From, and What is It Used For?

#### 1. Where Does This IP Come From?
**You invent and choose this IP yourself!**
* It is **NOT** assigned by AWS, Google Cloud, or an internet service provider (ISP).
* It is a **Virtual Private IP** defined in `wg0.conf` using standard private IP ranges ([RFC 1918](https://en.wikipedia.org/wiki/Private_network)):
  * `10.0.0.0` – `10.255.255.255`
  * `172.16.0.0` – `172.31.255.255`
  * `192.168.0.0` – `192.168.255.255`
* You can choose any private subnet (e.g., `10.0.0.1` and `10.0.0.2`, or `192.168.50.1` and `192.168.50.2`), as long as it does not clash with your existing AWS or GCP VPC subnets.

#### 2. What is It Used For?
The Tunnel IP is assigned to the **Virtual Network Card (`wg0`)** created by WireGuard inside your server:
* **🔒 Isolated Communication:** Your databases, APIs, or Kubernetes nodes bind directly to `10.0.0.1` or `10.0.0.2`. Because these IPs do not exist on the public internet, hackers cannot scan or attack your services directly.
* **📦 The Secret Walkie-Talkie (Routing & Encapsulation):**
  1. An application sends data to `10.0.0.2`.
  2. The Linux kernel routes it into the `wg0` virtual interface.
  3. WireGuard **encrypts the data** using GCP's public key.
  4. WireGuard wraps the encrypted payload in an outer UDP packet and sends it across the public internet to GCP's real public IP (`34.120.100.80:51820`).
  5. GCP receives the packet, **decrypts it**, and delivers it to the local app as coming from `10.0.0.1`.
* **🛡️ Cryptokey Routing (Anti-Spoofing):** WireGuard cryptographically pairs each public key with its `AllowedIPs`. It will immediately drop any packet claiming to come from `10.0.0.2` if it was not signed with GCP's corresponding private key.

---

> ### 🧠 Why Do We Need a Separate Subnet (`10.0.0.0/24`)?
> * **Prevents IP Collisions:** AWS default VPC subnets usually use `172.31.0.0/16`, while Google Cloud VPC uses `10.128.0.0/9`.
> * **Isolated Overlay Interface:** Creating a dedicated virtual network (`10.0.0.0/24`) creates a clean, independent communication layer (`wg0`) that does not interfere with the underlying cloud network routing.

---

## 2. Why Ubuntu Server for Cloud & WireGuard?

When deploying Linux servers on the cloud, **Ubuntu Server 24.04 LTS (Noble Numbat)** or **22.04 LTS (Jammy Jellyfish)** is the top recommendation worldwide:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ 🐧 WHY UBUNTU SERVER IS THE BEST CHOICE:                                    │
│                                                                             │
│ 1. Native Kernel WireGuard: Linux Kernel 5.6+ has WireGuard built directly  │
│    into the kernel (`kmod-wireguard`), giving near-zero CPU context loss.   │
│                                                                             │
│ 2. LTS Stability: Long-Term Support gives 5 full years of free security     │
│    patches and rock-solid system stability.                                 │
│                                                                             │
│ 3. Cloud-Optimized Kernels: Both AWS and Google Cloud maintain customized    │
│    kernels (`linux-aws` and `linux-gcp`) for maximum I/O performance.       │
│                                                                             │
│ 4. Massive Documentation: 90%+ of cloud networking & Kubernetes tutorials   │
│    use Ubuntu syntax (`apt`, `systemd`, `netplan`).                         │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Step 1: Launching Ubuntu Instances on AWS & Google Cloud

### A. 🟧 AWS EC2 Launch Flow & Field Explanations

1. Open **AWS Management Console** &rarr; Navigate to **EC2** &rarr; Click orange **Launch instances** button.
2. Fill out the launch setup form as follows:

| Field Name | What to Select / Type | 🧠 Why We Choose This |
| :--- | :--- | :--- |
| **Name and tags** | `aws-node-01` | Identifies this instance in your AWS EC2 dashboard. |
| **Application and OS Images (AMI)** | Click **Ubuntu** &rarr; Select **Ubuntu Server 24.04 LTS (HVM), SSD Volume Type** | Modern 64-bit OS with native WireGuard kernel module. Eligible for AWS Free Tier. |
| **Instance type** | `t3.micro` (or `t2.micro`) | Provides 2 vCPUs and 1 GiB RAM. Covered under the 750 free hours/month AWS Free Tier. |
| **Key pair (login)** | Select existing or click **Create new key pair** &rarr; Name: `aws-key` &rarr; Type: `RSA` &rarr; `.pem` | Used to SSH securely into your Ubuntu server using public-key authentication without passwords. |
| **Network settings & Firewall** | ☑️ **ENABLE:** Check **Allow SSH traffic from Anywhere** (`0.0.0.0/0`)<br>☑️ **ADD RULE:** Click **Edit** &rarr; Add Rule: **Custom UDP**, Port **51820**, Source **Anywhere** | Port 22 lets you administer the server via SSH. Port 51820/UDP allows WireGuard tunnel traffic. |
| **Configure storage** | `8 GiB` or `20 GiB` gp3 (General Purpose SSD) | Sufficient space for OS, WireGuard, Docker, and apps (up to 30 GB total is free on AWS). |

3. Click **Launch instance** (bottom right).

---

### B. 🟨 Google Cloud Compute Engine Launch Flow & Field Explanations

1. Open **Google Cloud Console** &rarr; Navigate to **Compute Engine** &rarr; **VM instances** &rarr; Click **Create instance**.
2. Fill out the configuration form as follows:

| Field Name | What to Select / Type | 🧠 Why We Choose This |
| :--- | :--- | :--- |
| **Name** | `gcp-node-01` | Name of your virtual machine on Google Cloud. |
| **Region & Zone** | `asia-southeast1` (Singapore) &rarr; `asia-southeast1-a` | Choose a region geographically close to your AWS region (e.g. AWS Singapore) for lowest network ping (<5ms). |
| **Machine configuration** | **General-purpose** &rarr; Series: **E2** &rarr; Machine type: **`e2-micro`** | 2 vCPUs, 1 GB memory. Eligible for Google Cloud Free Tier. |
| **Boot disk** | Click **Change** &rarr; OS: **Ubuntu** &rarr; Version: **Ubuntu 24.04 LTS x86/64** &rarr; Size: `20 GB` &rarr; Click **Select** | Modern Ubuntu LTS base with full cloud driver support. |
| **Firewall** | Leave HTTP/HTTPS unchecked (unless hosting a public web server). We will open **UDP 51820** in Step 3. | Keeps the GCP server private and secure, matching AWS. |

3. Click the blue **Create** button at the bottom.

---

## 4. Step 2: Server Memory Optimization — Enabling 2GB SWAP (Fixes VS Code Freezing)

> ⚠️ **CRITICAL STEP FOR `t3.micro` / `e2-micro` (1 GB RAM Instances)**:
> By default, cloud Linux instances come with **0 MB of Swap memory**. When VS Code Remote - SSH connects, its internal Node.js server consumes 500–700 MB of RAM, pushing total memory over 100% and causing the server to completely **freeze or disconnect**.

```
┌─────────────────────────────────────────────────────────────┐
│ 🧠 Physical RAM (Hardware Memory):    1 GB (Fast)           │
│ 💾 SSD Hard Drive (Storage Space):    20 GB (Large)         │
└─────────────────────────────────────────────────────────────┘
                                │
               Borrow 2 GB from your SSD Hard Drive!
                                │
                                ▼
┌─────────────────────────────────────────────────────────────┐
│ 🚀 TOTAL USABLE MEMORY: 1 GB RAM + 2 GB SWAP = 3 GB TOTAL!  │
└─────────────────────────────────────────────────────────────┘
```

### How to Enable SWAP (Run via Terminal SSH right after launching):
Connect to your instance using Windows PowerShell / Terminal (`ssh -i key.pem ubuntu@<IP>`) and run:

```bash
# 1. Allocate a 2 GB file on your SSD storage
sudo fallocate -l 2G /swapfile

# 2. Restrict permissions so only root can access it
sudo chmod 600 /swapfile

# 3. Format the file as Linux Swap space
sudo mkswap /swapfile

# 4. Activate the swap file immediately
sudo swapon /swapfile

# 5. Make the swap file permanent across server reboots
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

# 6. Clean up any corrupted VS Code cache from previous freezes
rm -rf ~/.vscode-server

# 7. Verify memory status
free -h
```

**Expected Output:**
```text
               total        used        free      shared  buff/cache   available
Mem:           952Mi       210Mi       500Mi       1.0Mi       242Mi       620Mi
Swap:          2.0Gi          0B       2.0Gi  <-- ✅ 2 GB virtual memory active!
```

---

## 5. Step 3: Cloud Firewall Rules (UDP 51820)

### 💡 Important Concept: AWS Security Groups vs. Google Cloud Firewall Scope

Before configuring the firewall, understand how AWS and Google Cloud handle firewall rules differently:

#### 1. 🟧 On AWS: Security Groups apply to **Specific Instances** (Not the whole account)
* An AWS Security Group acts like a **personal firewall attached to one specific virtual machine (NIC)**.
* When you edit a security group (e.g. `launch-wizard-3`), it **only affects the EC2 instance(s) that have `launch-wizard-3` attached to them**.
* If you launch a new instance tomorrow with a different security group, it will **NOT** have port 51820 open unless you attach `launch-wizard-3` to it.

```
AWS VPC:
├── 🖥️ EC2 Instance 1 (Attached to 'launch-wizard-3') ──► Port 51820 OPEN ✅
└── 🖥️ EC2 Instance 2 (Attached to 'default')         ──► Port 51820 CLOSED ❌
```

#### 2. 🟨 On Google Cloud (GCP): Firewall Rules apply to **The Whole VPC Network**
* In Google Cloud, firewall rules live at the **VPC Network level**.
* When you set **Targets = "All instances in the network"**, it automatically applies to **EVERY VM inside that VPC network**!

```
Google Cloud VPC (default):
├── 🖥️ GCP VM 1 ──► Port 51820 OPEN ✅ (Automatically)
└── 🖥️ GCP VM 2 ──► Port 51820 OPEN ✅ (Automatically)
```

#### 📊 Summary Scope Comparison:
| Cloud | Firewall Concept | Default Scope |
| :--- | :--- | :--- |
| **🟧 AWS** | **Security Group** | **Per Instance:** Only protects the specific VMs you attach it to. |
| **🟨 Google Cloud** | **VPC Firewall Rule** | **Whole Network:** Protects all VMs in that VPC (when Targets = *All instances*). |

---

### 🚀 How to Configure the Firewall Rules:

WireGuard requires **UDP port 51820** to be open on both cloud firewalls so encrypted packets can reach the machines.

### 🟧 On AWS Security Group:
1. In EC2 Console &rarr; Select `aws-node-01` &rarr; **Security** tab &rarr; Click the Security group link (e.g., `launch-wizard-3`).
2. On the Security Group page, under the **Inbound rules** tab &rarr; Click the button **`Edit inbound rules`** *(this opens the rule manager)*.
3. Click the **`Add rule`** button at the bottom left to create a **NEW row** *(⚠️ Do NOT edit or delete the existing SSH Port 22 row!)*.
4. In the new row, fill in:
   * **Type:** Select `Custom UDP`
   * **Port range:** Type `51820`
   * **Source:** Select `Anywhere-IPv4` (shows `0.0.0.0/0`).
5. Click the orange **`Save rules`** button.

> **✅ Verification:** Your Inbound rules table must now show **2 rules**:
> * **Rule 1:** `SSH` | TCP | Port `22` | `0.0.0.0/0` *(For laptop SSH access)*
> * **Rule 2:** `Custom UDP` | UDP | Port `51820` | `0.0.0.0/0` *(For WireGuard VPN tunnel)*

### 🟨 On Google Cloud (GCP) Firewall:
1. In the top search bar, type **firewall** &rarr; click **Firewall (VPC network)** (opens the *Firewall policies* / *VPC firewall rules* page).
2. At the top of the screen (next to the page title), click the second button: **`➕ Create firewall rule`** *(do NOT click Create firewall policy)*.
3. Fill in the firewall rule configuration:
   * **Name:** `allow-wireguard-51820`
   * **Network:** `default`
   * **Targets:** Select **All instances in the network**
   * **Source IPv4 ranges:** Type `0.0.0.0/0` (or AWS's Public IPv4 address)
   * **Protocols and ports:** Select **Specified protocols and ports** &rarr; Check **☑ udp** &rarr; type `51820`
4. Scroll to the bottom and click the blue **Create** button.

---

> ### 🧠 Why Do We Have to Do This?
> * **Why UDP instead of TCP?** WireGuard exclusively uses UDP. Running TCP inside a TCP-based VPN tunnel causes a severe performance penalty called **"TCP Meltdown"** (where retransmission timers collide and choke the network). UDP allows raw, low-latency transmission.
> * **Why Port 51820?** `51820` is the standard IANA-assigned default port for WireGuard.
> * **Default Block Behavior:** Both AWS and GCP block all inbound traffic by default. If this port is not explicitly opened, WireGuard packets will be silently dropped before reaching your VM.

---

## 6. Step 4: Install WireGuard on Both Nodes

SSH into **both** your AWS and GCP servers and run:

```bash
# Ubuntu / Debian
sudo apt update && sudo apt install -y wireguard iptables
```

---

> ### 🧠 Why Do We Have to Do This?
> * **Kernel-Space Performance:** Unlike older VPNs (like OpenVPN) that run in slow "User Space" and constantly switch CPU context via `/dev/net/tun`, WireGuard is compiled directly into the **Linux Kernel (`kmod-wireguard`)**. This allows it to process cryptographic packets at near-wireline speeds with minimum CPU overhead.
> * **`wireguard-tools`:** Installs the essential command-line utilities (`wg`, `wg-quick`) used to configure and manage tunnel interfaces.

---

## 7. Step 5: Generate Cryptographic Key Pairs

You must generate a private and public key pair on **each** server.

### 🟧 On AWS Node (`aws-node-01`):
Run this command block (uses `sudo bash -c` to grant proper permissions):
```bash
sudo bash -c '
mkdir -p /etc/wireguard
cd /etc/wireguard
umask 077
wg genkey | tee aws_private.key | wg pubkey > aws_public.key

echo "================================"
echo "AWS Private Key: $(cat aws_private.key)"
echo "AWS Public Key:  $(cat aws_public.key)"
echo "================================"
'
```

### 🟨 On GCP Node (`gcp-node-01`):
Run this command block on Google Cloud:
```bash
sudo bash -c '
mkdir -p /etc/wireguard
cd /etc/wireguard
umask 077
wg genkey | tee gcp_private.key | wg pubkey > gcp_public.key

echo "================================"
echo "GCP Private Key: $(cat gcp_private.key)"
echo "GCP Public Key:  $(cat gcp_public.key)"
echo "================================"
'
```

---

> ### 🧠 Why Do We Have to Do This?
> * **Curve25519 Public-Key Cryptography:** WireGuard uses asymmetric cryptography (similar to SSH keys). 
>   * The **Private Key** is kept strictly secret on the local server to decrypt incoming data.
>   * The **Public Key** is shared with the other cloud node so it can verify your identity and encrypt outgoing packets.
> * **Why `umask 077`?** Sets strict Linux file permissions (`-rw-------`) ensuring that **only the root user** can read your private key file, protecting it from other users or compromised processes on the system.

---

## 8. Step 6: Create Configuration Files (`wg0.conf`) & Parameter Breakdown

### 📋 Values Checklist Before Creating Files:
Make sure you have gathered these 4 values:
1. **AWS Private Key:** Run on AWS &rarr; `sudo cat /etc/wireguard/aws_private.key`
2. **AWS Public Key & Public IP:** Run on AWS &rarr; `sudo cat /etc/wireguard/aws_public.key` & check EC2 Public IPv4
3. **GCP Private Key:** Run on GCP &rarr; `sudo cat /etc/wireguard/gcp_private.key`
4. **GCP Public Key & Public IP:** Run on GCP &rarr; `sudo cat /etc/wireguard/gcp_public.key` & check GCP External IP

---

### A. 🟧 On AWS EC2 Node 1 (`/etc/wireguard/wg0.conf`):
Run: `sudo nano /etc/wireguard/wg0.conf` and paste:

```ini
[Interface]
# AWS Virtual Tunnel IP
Address = 10.0.0.1/24
ListenPort = 51820
# Paste your AWS Private Key below:
PrivateKey = <PASTE_YOUR_AWS_PRIVATE_KEY_HERE>

[Peer]
# Paste Google Cloud's Public Key below:
PublicKey = <PASTE_YOUR_GCP_PUBLIC_KEY_HERE>
# Replace with your GCP VM's real External Public IP:
Endpoint = <PASTE_YOUR_GCP_EXTERNAL_PUBLIC_IP_HERE>:51820
# Route GCP's tunnel IP through this peer:
AllowedIPs = 10.0.0.2/32
# Keeps the connection open across cloud NAT firewalls:
PersistentKeepalive = 25
```

---

### B. 🟨 On Google Cloud Node 2 (`/etc/wireguard/wg0.conf`):
Run: `sudo nano /etc/wireguard/wg0.conf` and paste:

```ini
[Interface]
# GCP Virtual Tunnel IP
Address = 10.0.0.2/24
ListenPort = 51820
# Paste your GCP Private Key below:
PrivateKey = <PASTE_YOUR_GCP_PRIVATE_KEY_HERE>

[Peer]
# Paste AWS's Public Key below:
PublicKey = <PASTE_YOUR_AWS_PUBLIC_KEY_HERE>
# Replace with your AWS EC2 instance's real Public IPv4 address:
Endpoint = <PASTE_YOUR_AWS_PUBLIC_IP_HERE>:51820
# Route AWS's tunnel IP through this peer:
AllowedIPs = 10.0.0.1/32
# Keeps the connection open across cloud NAT firewalls:
PersistentKeepalive = 25
```

---

### 🧠 Deep Dive: Why Every Single Parameter Exists

| Parameter | Section | Technical Purpose ("Why We Need It") |
| :--- | :---: | :--- |
| **`Address`** | `[Interface]` | Assigns the static IP address to the virtual network adapter `wg0` on this local machine. |
| **`ListenPort`** | `[Interface]` | Specifies which UDP port the Linux kernel should listen on for incoming encrypted WireGuard packets. |
| **`PrivateKey`** | `[Interface]` | The local node's private cryptographic key used to authenticate itself and decrypt packets. |
| **`PublicKey`** | `[Peer]` | The remote node's public key. WireGuard uses this to encrypt traffic destined for that peer and verify that received packets genuinely originated from them (**Cryptokey Routing**). |
| **`Endpoint`** | `[Peer]` | The public IP and port (`IP:51820`) of the remote peer where initial handshake packets are sent. |
| **`AllowedIPs`** | `[Peer]` | **Crucial Double Role:**<br>1. **Routing Table:** Any traffic sent to `10.0.0.2` is routed into the `wg0` tunnel.<br>2. **Internal Firewall:** WireGuard will **drop and reject** any packet claiming to come from any IP address not listed in `AllowedIPs` (prevents IP spoofing). |
| **`PersistentKeepalive = 25`** | `[Peer]` | **Essential for Cloud & NAT Firewalls:** Cloud NAT routers (like AWS NAT Gateway or GCP Cloud NAT) automatically close idle UDP connection states after 30–60 seconds. Sending a tiny heartbeat packet every 25 seconds forces the firewall state table to **keep the connection open 24/7**. |

---

### 📍 Deep Dive: What does `/32` mean and is this IP defined by YOU?

#### 1. Yes! You define this IP 100% yourself:
* **YOU** chose to give AWS the IP: `10.0.0.1`
* **YOU** chose to give Google Cloud the IP: `10.0.0.2`
* Because you chose those two IPs, in `AllowedIPs` you tell each node to send traffic to the other node's defined IP address!

#### 2. Why `/24` in `[Interface]` vs. `/32` in `[Peer]`?
* **`/24` (Subnet Range):** Represents 256 IPs (`10.0.0.0` &rarr; `10.0.0.255`). When you set `Address = 10.0.0.1/24`, it tells the local operating system: *"My `wg0` interface belongs to a 256-node virtual network."*
* **`/32` (Single Specific IP):** Represents **EXACTLY 1 IP address** (no range). When you set `AllowedIPs = 10.0.0.2/32`, it tells WireGuard: *"The remote peer is ONLY allowed to represent this ONE exact IP (`10.0.0.2`)."*

#### 3. Why this is crucial (Split Tunneling & Anti-Spoofing):
* **Split Tunneling:** Only traffic specifically sent to `10.0.0.2` is encrypted and routed through the tunnel. Your normal web browsing and server updates continue running at full speed through your normal internet connection.
* **Anti-Spoofing:** If a compromised node tries to send packets claiming to be from any other IP (like `192.168.1.1`), WireGuard **instantly drops them**.

---

## 9. Step 7: Enable Kernel IP Forwarding & Start Service

### 1. Enable Kernel IP Forwarding:
```bash
sudo sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
```

### 2. Start & Enable WireGuard Service:
```bash
# Start WireGuard interface wg0 and enable on system reboot
sudo systemctl enable --now wg-quick@wg0
```

---

### 🧠 Deep Dive: Why Both Commands are Required (RAM vs. Disk)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ 🛑 ip_forward = 0 (Default - Host Mode):                                    │
│    Packet from Cloud ──► eth0 ──► [ Kernel drops packet ❌ ]                │
│                                                                             │
│ 🟢 ip_forward = 1 (Router Mode - WireGuard):                                │
│    Packet from Cloud ──► eth0 ──► [ Kernel forwards to wg0 tunnel ✅ ]      │
└─────────────────────────────────────────────────────────────────────────────┘
```

1. **`sudo sysctl -w net.ipv4.ip_forward=1` (Modifies Live RAM):**
   * Modifies the active Linux Kernel parameter in live memory right now.
   * *Limitation:* Cleared when the server reboots.
2. **`echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf` (Saves to SSD Disk):**
   * Appends the setting to the permanent system configuration file.
   * *Purpose:* Ensures that every time the server reboots or restarts, Linux automatically loads and enables IP forwarding.
3. **`sudo systemctl enable --now wg-quick@wg0`:**
   * Starts the `wg0` virtual interface immediately and registers it with `systemd` so the VPN tunnel starts automatically on server boot.

---

## 10. Step 8: Test & Verify Tunnel Handshake

### 1. Check WireGuard Status:
```bash
sudo wg show
```

**Expected Output:**
```text
interface: wg0
  public key: <PUBLIC_KEY>
  private key: (hidden)
  listening port: 51820

peer: <PEER_PUBLIC_KEY>
  endpoint: 34.120.100.80:51820
  allowed ips: 10.0.0.2/32
  latest handshake: 14 seconds ago     <-- ✅ Confirms connection!
  transfer: 1.42 KiB received, 1.88 KiB sent
```

### 2. Ping Across Clouds:
* **From AWS:**
  ```bash
  ping -c 4 10.0.0.2
  ```
* **From Google Cloud:**
  ```bash
  ping -c 4 10.0.0.1
  ```

---

> ### 🧠 Why Do We Have to Do This?
> * **WireGuard is "Stealth & Silent by Design":** Unlike other VPNs, WireGuard **never responds to port scans or pings** if the request does not have a valid cryptographic signature. It does not send error messages or acknowledgments to unauthorized packets.
> * **Why `latest handshake` is the true test:** A timestamp showing `latest handshake: X seconds ago` is the definitive cryptographic proof that both servers successfully exchanged keys and established an encrypted tunnel.

---

## 11. Bonus: Configuring WireGuard on Windows Server

If one of your nodes is a **Windows Server**:

1. Download and install **WireGuard for Windows** from [wireguard.com/install/](https://www.wireguard.com/install/).
2. Open WireGuard &rarr; Click **Add Tunnel** &rarr; **Add empty tunnel...**.
3. Replace the content with:
   ```ini
   [Interface]
   PrivateKey = <AUTOMATICALLY_GENERATED_PRIVATE_KEY>
   Address = 10.0.0.3/24
   ListenPort = 51820

   [Peer]
   PublicKey = <AWS_OR_GCP_PUBLIC_KEY>
   Endpoint = 16.170.200.50:51820
   AllowedIPs = 10.0.0.0/24
   PersistentKeepalive = 25
   ```
4. Click **Save** &rarr; Click **Activate**.

---

## 12. Troubleshooting Common Issues

| Issue | Root Cause | Technical Explanation & Fix |
| :--- | :--- | :--- |
| **VS Code Remote - SSH Freezes on connection** | Out of memory (0 MB swap default on 1 GB RAM instance). | Follow **Step 2** to allocate a 2 GB swap file and clean `~/.vscode-server`. |
| **`latest handshake` is missing** | UDP 51820 blocked by cloud firewall. | Packets cannot reach the VM. Check AWS Security Group & GCP Firewall rules for `51820/UDP`. |
| **`0 B received` in transfer** | Key mismatch or wrong Endpoint IP. | The receiver could not verify the cryptographic signature. Verify that AWS has GCP's Public Key, and GCP has AWS's Public Key. |
| **Connection drops after 1–2 minutes** | NAT mapping expired on cloud gateway. | Ensure `PersistentKeepalive = 25` is present in the `[Peer]` section so heartbeats keep the NAT session alive. |
| **`RTNETLINK answers: File exists`** | Interface `wg0` already running. | An old instance is locked. Run `sudo wg-quick down wg0` and then `sudo wg-quick up wg0`. |
