# Palo Alto PA-VM Centralized Log Forwarding to Bindplane Cloud

![Platform](https://img.shields.io/badge/Firewall-Palo%20Alto%20PA--VM-FA582D?style=flat-square)
![Collector](https://img.shields.io/badge/Collector-Bindplane%20OTel%20Agent-1E88E5?style=flat-square)
![OS](https://img.shields.io/badge/Host-Ubuntu%2022.04%20LTS-E95420?style=flat-square)
![Protocol](https://img.shields.io/badge/Transport-Syslog%20UDP%205514-4CAF50?style=flat-square)

## Overview
This repository documents an end-to-end telemetry ingestion lab integrating a **Palo Alto Networks PA-VM** firewall with **Bindplane Cloud** using the **Bindplane Distribution for OpenTelemetry (BDOT)** collector on Ubuntu Linux. The pipeline ingests firewall `TRAFFIC` and `THREAT` logs over Syslog (`RFC 3164` / `BSD` format) and validates live log parsing inside Bindplane Telemetry.

---

## Architecture & Lab Topology

```text
+-----------------------+       Syslog (UDP/5514)       +---------------------------+       HTTPS (OTLP)       +-------------------+
|   Palo Alto PA-VM     | ----------------------------> |  Ubuntu Collector Host    | -----------------------> |  Bindplane Cloud  |
|  Trust: 10.0.1.1/24   |   Source: ethernet1/2         |  Hostname: pa-client      |   Agent: observiq-otel-c | |  Live Telemetry   |
|  Mgmt: 10.170.1.115   |   Format: BSD (RFC 3164)      |  IP: 10.0.1.20/24         |   Version: v1.113.0      | |  & Pipelines      |
+-----------------------+                               +---------------------------+                          +-------------------+
```

### Network & Component Details
| Component | Role | Interface / IP | Details |
| :--- | :--- | :--- | :--- |
| **Proxmox VE** | Hypervisor | Host Bridge | Hosts PA-VM and Ubuntu Collector VM |
| **PA-VM** | Next-Gen Firewall | `ethernet1/2` (`10.0.1.1/24`) | Trust zone gateway & Syslog source |
| **pa-client** | OTel Collector Host | `ens18` (`10.0.1.20/24`) | Ubuntu 22.04 running `observiq-otel-collector` |
| **Bindplane Cloud** | Telemetry Control Plane | SaaS | Manages pipeline rollout & live log inspection |

---

## Step-by-Step Implementation

### Phase 1: Bindplane Cloud Configuration
1. **Create Configuration (`Palo_Alto_Firewall_Logs`):**
   - **Collector Type:** `BDOT 1.x (Stable)`
   - **Platform:** `Linux` | **Collector Role:** `Edge`
2. **Configure Syslog Source:**
   - **Listening IP Address:** `0.0.0.0`
   - **Listening Port:** `5514`
   - **Protocol:** `rfc3164` *(matches Palo Alto BSD format)*
   - **Transport Protocol:** `udp`
3. **Configure Destination & Rollout:**
   - Added `Dev_Null` destination for ingestion testing and live telemetry inspection.
   - Assigned configuration to collector **`pa-client`** and verified listening socket on Ubuntu:
     ```bash
     sudo ss -ulnp | grep 5514
     # Expected: UNCONN 0 0 *:5514 *:* users:(("observiq-otel-c",pid=...,fd=8))
     ```

### Phase 2: Palo Alto PA-VM Configuration
1. **Syslog Server Profile (`DEVICE > Server Profiles > Syslog`):**
   - **Profile Name:** `Bindplane-Syslog-Profile`
   - **Server Name:** `Bindplane-Collector`
   - **Syslog Server:** `10.0.1.20` | **Transport:** `UDP` | **Port:** `5514`
   - **Format:** `BSD` | **Facility:** `LOG_USER`
2. **Log Forwarding Profile (`OBJECTS > Log Forwarding`):**
   - **Profile Name:** `Forward-to-Bindplane`
   - **Match List 1 (`All-Traffic-Logs`):** Log Type `traffic` $\rightarrow$ Syslog `Bindplane-Syslog-Profile`
   - **Match List 2 (`All-Threat-Logs`):** Log Type `threat` $\rightarrow$ Syslog `Bindplane-Syslog-Profile`
3. **Security Policy Attachment (`POLICIES > Security`):**
   - Opened rule `Allow-Trust-to-Untrust` $\rightarrow$ **Actions** tab.
   - Enabled **Log at Session End** and set **Log Forwarding** to `Forward-to-Bindplane`.
4. **Service Route Configuration (`DEVICE > Setup > Services`):**
   - Navigated to **Service Route Configuration > Customize > IPv4 > Syslog**.
   - Set **Source Interface** to `ethernet1/2` and **Source Address** to `10.0.1.1/24`.
5. **Commit:** Applied all configuration changes via **Commit**.

---

## Troubleshooting Lessons Learned
* **Issue:** Logs did not initially appear in Bindplane Cloud even after committing the Syslog and Log Forwarding profiles and verifying that `ufw` on Ubuntu was inactive.
* **Root Cause:** By default, PAN-OS originates Syslog traffic from the **Management Interface (`10.170.1.115`)**, which had no route to the Trust subnet (`10.0.1.20/24`).
* **Resolution:** Configured a custom **Service Route** for `Syslog` to originate from the Trust interface (`ethernet1/2` / `10.0.1.1/24`), immediately enabling packet delivery to the collector.

---

## Verification
1. **Traffic Generation:**
   ```bash
   ping -c 4 8.8.8.8
   ```
2. **Packet-Level Verification on Collector (`pa-client`):**
   ```bash
   sudo tcpdump -i any udp port 5514 -n
   # Output: IP 10.0.1.1.60169 > 10.0.1.20.5514: UDP, length 754
   ```
3. **Bindplane Telemetry Verification:**
   - Confirmed active throughput (`13.2 KB/m`) on the `Syslog -> Dev_Null` pipeline.
   - Inspected parsed `TRAFFIC` log entries under **Collectors > pa-client > View Recent Telemetry**.