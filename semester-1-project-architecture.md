# 🎓 Windows Server Semester 1 Final Project: AWS Cloud Architecture

This document provides the complete, recommended cloud architecture to deploy all required Windows Server roles and services on Amazon Web Services (AWS) within student lab/Free Tier constraints.

---

## 📑 Table of Contents
1. [Project Requirements Scope](#project-requirements-scope)
2. [Recommended AWS Architecture (Optimized 4-Server Model)](#recommended-aws-architecture-optimized-4-server-model)
3. [Server Roles & Services Breakdown](#server-roles--services-breakdown)
4. [Cloud Networking & VPC Layout](#cloud-networking--vpc-layout)
5. [How Each Required Service is Implemented on AWS](#how-each-required-service-is-implemented-on-aws)
6. [Cost & Resource Optimization for Students](#cost--resource-optimization-for-students)

---

## 📋 1. Project Requirements Scope

| Category | Required Services |
| :--- | :--- |
| **Directory & Core Network** | Active Directory (AD DS), DNS Server, DHCP Server, Radius Server (NPS) |
| **Web & Application Services** | Web Server (IIS), FTP Server, Database Server (PostgreSQL / Oracle) |
| **Messaging & Remote Access** | Mail Server (hMailServer), VPN Server (RRAS), Terminal Server (RDS) |
| **Security & Routing** | Proxy Server (Reverse Proxy / ARR), Security Groups / Firewalls |
| **High Availability & Storage** | Load Balancing (ALB / NLB), Failover Cluster (WSFC), File Server, Backup Server |

---

## 🏗️ 2. Recommended AWS Architecture (Optimized 4-Server Model)

Running 14 separate servers simultaneously would exceed AWS student credit limits and memory caps. **The industry standard for lab projects is consolidating compatible roles onto 4 logically grouped servers:**

```
                                  ┌──────────────────────────────┐
                                  │      INTERNET / CLIENTS      │
                                  └──────────────┬───────────────┘
                                                 │
                                                 ▼
               ┌──────────────────────────────────────────────────────────────────┐
               │              AWS APPLICATION LOAD BALANCER (ALB)                 │
               │                   (Port 80 / 443 Traffic Distribution)           │
               └─────────────────┬──────────────────────────────┬─────────────────┘
                                 │                              │
                                 ▼                              ▼
    ┌────────────────────────────────────────┐     ┌────────────────────────────────────────┐
    │  🌐 PUBLIC SUBNET (10.0.1.0/24)        │     │  🌐 PUBLIC SUBNET (10.0.1.0/24)        │
    │                                        │     │                                        │
    │  🖥️ SERVER 2: Web, FTP & Proxy         │     │  🖥️ SERVER 3: Remote Access & Mail    │
    │  • IIS Web Server                      │     │  • VPN Server (RRAS)                   │
    │  • FTP Server (IIS FTP)                │     │  • Mail Server (hMailServer)           │
    │  • Proxy / Reverse Proxy (IIS ARR)     │     │  • Terminal Server (RDS Session Host)  │
    └──────────────────┬─────────────────────┘     └──────────────────┬─────────────────────┘
                       │                                              │
                       └──────────────────────┬───────────────────────┘
                                              │
                                              ▼
    ┌───────────────────────────────────────────────────────────────────────────────────────┐
    │  🔒 PRIVATE SUBNET (10.0.2.0/24) - Internal Domain Zone                               │
    │                                                                                       │
    │  ┌─────────────────────────────────────┐     ┌─────────────────────────────────────┐  │
    │  │ 👑 SERVER 1: Domain & Core Infra    │     │ 🗄️ SERVER 4: Database & Storage     │  │
    │  │ • Active Directory (AD DS)          │     │ • Database (PostgreSQL / Oracle)    │  │
    │  │ • DNS Server & DHCP Server          │     │ • File Server (SMB / DFS Share)     │  │
    │  │ • Radius Server (NPS)               │     │ • Failover Cluster Node (WSFC)      │  │
    │  │ • Backup Server (Windows Backup)    │     │                                     │  │
    │  └─────────────────────────────────────┘     └─────────────────────────────────────┘  │
    └───────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 💻 3. Server Roles & Services Breakdown

### 👑 Server 1: `DC-CORE-01` (Domain Controller & Infrastructure)
* **OS:** Windows Server 2022/2025 Base
* **IP:** `10.0.2.10` (Private Subnet)
* **Services Hosted:**
  1. **Active Directory Domain Services (AD DS):** Forest root domain (e.g., `rupp.local`).
  2. **DNS Server:** Active Directory Integrated DNS for internal name resolution.
  3. **DHCP Server:** Configured DHCP scope (demonstrated for class requirements).
  4. **Radius Server (NPS):** Network Policy Server for authenticating VPN and network users.
  5. **Backup Server:** Windows Server Backup / Volume Shadow Copies configured for system state.

---

### 🌐 Server 2: `WEB-PROXY-01` (Web, FTP & Proxy Gateway)
* **OS:** Windows Server 2022/2025 Base
* **IP:** `10.0.1.20` (Public Subnet)
* **Services Hosted:**
  1. **Web Server:** Microsoft IIS (Internet Information Services) hosting HTTP/HTTPS websites.
  2. **FTP Server:** IIS FTP Service configured with user isolation.
  3. **Proxy / Reverse Proxy Server:** IIS Application Request Routing (ARR) / URL Rewrite module to route incoming traffic safely.

---

### 📬 Server 3: `REMOTE-COMM-01` (Communication & Remote Access)
* **OS:** Windows Server 2022/2025 Base
* **IP:** `10.0.1.30` (Public Subnet)
* **Services Hosted:**
  1. **VPN Server:** Routing and Remote Access Service (RRAS) configured with SSTP / L2TP.
  2. **Mail Server:** hMailServer (Open-source Windows SMTP/POP3/IMAP) with WebMail (Roundcube).
  3. **Terminal Server:** Remote Desktop Services (RDS Session Host) for remote app publishing.

---

### 🗄️ Server 4: `DATA-STORAGE-01` (Database & Central Storage)
* **OS:** Windows Server 2022/2025 Base
* **IP:** `10.0.2.40` (Private Subnet)
* **Services Hosted:**
  1. **Database Server:** PostgreSQL 16 for Windows (or Oracle Database 19c / 21c XE).
  2. **File Server:** File and Storage Services, SMB Shares, File Server Resource Manager (FSRM) with quotas.
  3. **Failover Cluster / HA Demonstration:** Windows Server Failover Clustering (WSFC).

---

## ⚙️ 4. How Each Required Service is Implemented on AWS

| Service | Recommended Windows Role / Software | Implementation Notes on AWS |
| :--- | :--- | :--- |
| **Active Directory** | `AD DS` role | Root domain controller `rupp.local`. |
| **DNS Server** | `DNS Server` role | AD-integrated DNS forward & reverse lookup zones. |
| **DHCP Server** | `DHCP Server` role | *Note:* AWS VPC handles IP assignment natively; configure Windows DHCP scope for lab grading. |
| **Web Server** | `Web Server (IIS)` role | Default website on Port 80/443. |
| **FTP Server** | `IIS FTP Service` role | Passive mode FTP with specified AWS public IP. |
| **Mail Server** | `hMailServer` (SMTP/POP3/IMAP) | Lightweight, free, and industry-proven for Windows Server labs. |
| **VPN Server** | `Routing and Remote Access (RRAS)` | SSTP (Port 443) or L2TP/IPSec. |
| **Radius Server** | `Network Policy Server (NPS)` | Authenticates VPN users against Active Directory credentials. |
| **Terminal Server** | `Remote Desktop Services (RDS)` | Remote Desktop Session Host. |
| **Load Balancing** | **AWS Application Load Balancer (ALB)** | Distributes web traffic across web servers. |
| **Failover Cluster** | `Failover Clustering (WSFC)` | High-availability cluster service. |
| **Proxy Server** | `IIS + Application Request Routing (ARR)` | Inbound reverse proxy & caching. |
| **Backup Server** | `Windows Server Backup` feature | Scheduled automated backups to dedicated EBS volume. |
| **Database Server** | **PostgreSQL for Windows** / **Oracle XE** | Relational DB accessible by Web Server. |

---

## 💰 5. Cost & Resource Optimization for AWS Free Tier

* **Instance Type:** Use `t3.micro` or `t3.small` (Free tier gives 750 hours/month).
* **Storage:** 30 GB EBS gp3 volume per server (Free tier allows up to 30 GB total free storage, keep stopped instances clean).
* **Best Practice:** When not presenting or testing, **Stop** the EC2 instances from the AWS Console to save compute hours and budget!
