# Network-Traffic-Analysis-Wireshark-Basics-
# SOC Internship · Week 3: Task 9 — Network Traffic Analysis (Wireshark Basics)

## 📌 Repository Overview
This repository contains documentation, filter references, and workflow note Security Operations Center (SOC) analyst internship. The primary objective is mastering packet capture, interface management, protocol dissection, and stream reassembly using Wireshark within a virtualized Linux environment.

---

## 🛠️ Lab Environment & Architecture
* **Virtualization Platform:** Oracle VirtualBox
* **Operating System:** Kali Linux VM
* **Assigned Host IPv4 Address:** `10.0.0.2`
* **Primary Network Interfaces:**
  * `lo` (Loopback Interface — `127.0.0.1`): Used for internal system communication and local service testing.
  * `eth0` (Ethernet Interface — `10.0.0.2`): Handles external network traffic, internet access, and gateway routing.

---

## 🔍 Part 0: Local Baseline & Interface Testing (`lo` & `curl`)
Before capturing external internet traffic, baseline testing is conducted on the internal loopback interface to verify local service responses without generating external network noise.

* **Target Interface:** `lo`[cite: 1]
* **Terminal Command Executed:**
  ```bash
  curl [http://127.0.0.1](http://127.0.0.1)
  
  Operational Purpose: Generates internal HTTP traffic communicating entirely within the local host (testing local Apache web server responses) without hitting physical network hardware.

Verification: Confirmed loopback packet generation, TCP 3-way handshake on localhost, and plaintext HTTP response headers inside Wireshark.

## 📊 3. Part 1: Core Display Filters & Functions Reference

| Display Filter Syntax | Function & Description |
| :--- | :--- |
| `ip.addr == 10.0.0.2` | Filters all IPv4 packets where your specific virtual machine IP (`10.0.0.2`) is either the source or destination address. |
| `ip.src == 10.0.0.2` | Filters packets originating strictly from your virtual machine's source IP address. |
| `ip.dst == 10.0.0.2` | Filters packets destined strictly for your virtual machine's destination IP address. |
| `http` | Filters packets to display only unencrypted Hypertext Transfer Protocol web traffic (ideal for loopback testing on `127.0.0.1`). |
| `dns` | Isolates Domain Name System queries and responses for name resolution inspection. |
| `tcp.stream eq [ID]` | Groups and reassembles every packet belonging to a specific TCP conversation session from handshake to teardown. |---

## 🌐 4. Part 2: External Traffic Capture & Protocol Dissection (`eth0`)

1. **Interface Switch:** Switched capture target from loopback (`lo`) to `eth0` (VM IP: `10.0.0.2`) to handle external internet and DNS traffic.
2. **Standard Query Executed in Terminal:**
   ```bash
   nslookup google.com
   ```
   *(Note: Executing standard `nslookup` without `-debug` cleanly generates standard outbound queries, Type A / class IN, sent from `10.0.0.2` to public resolvers like `8.8.8.8`.)*
3. **DNS Filter Applied:** Entered `dns` in the Wireshark display filter bar to isolate and inspect outbound resolution packets.
4. **Host Isolation Filter Applied:** Entered `ip.addr == 10.0.0.2` to strip background noise and isolate all incoming/outgoing traffic for your virtual machine.

---

## 🔬 5. Part 3: Advanced Stream Inspection, DNS Analysis & TLS SNI Correlation

* **TCP Handshake Tracking:** Inspected outbound connection sequences (`SYN`, `SYN-ACK`, `ACK`) destined for external endpoints (e.g., `104.21.40.220`).
* **DNS-to-TCP Stream Correlation:** Connected the initial `dns` query resolution phase to the subsequent TCP connection establishment. When a domain is resolved via DNS, the host immediately initializes a TCP session to the resulting destination IP address.
* **Stream Reassembly:** Utilized the **Follow TCP Stream** feature on active handshake packets.
* **Handling Stream ID Shifts:** Noted that activating stream tracking automatically overwrites the display filter to `tcp.stream eq 0` (or the specific stream index), restricting the view exclusively to that active conversation window.
* **Encryption Handling:** Recognized that application-layer payloads for HTTPS/TLS 1.3 traffic appear as binary/garbled text due to session encryption.
* **TLS SNI Extraction:** Located the unencrypted **Server Name Indication (SNI)** field located at the very top of the TLS Client Hello handshake preamble, successfully verifying destination identifiers such as `cti.wazuh.com`.

---

## 🚨 6. Troubleshooting & Roadblocks Addressed

### **Roadblock 1: Unexpected Stream Filter Redirection**
* **Issue:** Attempting to inspect individual packets after clicking **Follow TCP Stream** caused Wireshark to automatically apply `tcp.stream eq 0`, shifting the active view away from the broader IP filter.
* **Resolution:** Understood that stream reassembly locks onto an isolated index. Cleared or closed the stream window and re-applied `ip.addr == 10.0.0.2` when returning to global host analysis.

### **Roadblock 2: Interpreting Encrypted Payloads**
* **Issue:** Viewing raw TCP stream data initially presented unreadable binary characters, leading to confusion over whether data was captured incorrectly.
* **Resolution:** Identified that HTTPS traffic on port 443 is encrypted via TLS 1.3. Successfully bypassed the encrypted body by scrolling to the very top of the stream transcript to extract the cleartext SNI domain (`cti.wazuh.com`) exposed during the initial handshake.

### **Roadblock 3: Interface Differentiation (`lo` vs `eth0`)**
* **Issue:** Local `curl` queries executed on `127.0.0.1` did not appear when capturing on external interfaces.
* **Resolution:** Verified that local testing requires monitoring the loopback (`lo`) interface, while external DNS queries (`nslookup`) require the ethernet (`eth0`) interface.

