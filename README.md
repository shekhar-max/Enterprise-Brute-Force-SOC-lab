# Enterprise-Brute-Force-SOC-lab
A multi-zone virtualized SOC home lab featuring a segmented pfSense firewall environment, automated log pipelines (Windows Event Logs &amp; Linux Syslogs) via Splunk Universal Forwarders, and structured Suricata NIDS telemetry ingestion into a centralized Splunk SIEM platform for brute force attack simulation
# Multi-Zone Enterprise SOC Home Lab & Log Ingestion Pipeline

## 📌 Project Overview
This project demonstrates the end-to-end design, deployment, and configuration of an enterprise-grade Security Operations Center (SOC) home lab environment. Built entirely within a virtualized network environment, this lab showcases real-world blue-teaming fundamentals including network segmentation, Host-based and Network-based Intrusion Detection Systems (HIDS/NIDS), centralized SIEM log collection, and firewall access control configurations.

---

## 🏗️ Architecture Topology
The infrastructure is orchestrated using a virtualized firewall router segmenting four distinct, isolated internal virtual networks (using host-only internal switches) to simulate a hardened corporate environment.

*Insert your custom topology diagram image here:*
![SOC Lab Topology](<img width="1258" height="832" alt="image_69be1945" src="https://github.com/user-attachments/assets/84407bcd-0d6e-4dc2-bf7e-7d39bdd0d176" />
)

### 🌐 Network Segmentation Matrix

| Zone Name | Underlying Interface | Subnet Range | Active Assets & Monitoring Daemons |
| :--- | :--- | :--- | :--- |
| **WAN_NET** | `vtnet0` / Bridged | DHCP (External) | Physical Host Uplink (Internet Access) |
| **SIEM_NET** | `vtnet1` / Internal | `192.168.10.0/24` | **Ubuntu Server:** Splunk Enterprise Server (Search Head & Indexer) |
| **DMZ_NET** | `vtnet2` / Internal | `192.168.20.0/24` | **Windows Server & Ubuntu Client:** Splunk Universal Forwarders, Suricata NIDS |
| **APPLICATION_NET** | `vtnet3` / Internal | `192.168.30.0/24` | **Linux Node:** Microservices stack (Docker, Kubernetes, Nginx Web Gateway) |
| **ATTACKER_NET** | `vtnet4` / Internal | `192.168.40.0/24` | **Kali Linux:** Pentesting Suite (Nmap, Metasploit, Hydra) |

---

## 🔧 Technical Implementations & Steps

### 1. Network Infrastructure & Routing (pfSense Setup)
* Configured virtual adapters within the hypervisor mapping to dedicated isolated internal networks (`siem_net`, `dmz_net`, `applications_net`, `attacker_net`).
* Provisioned static IP mapping gateways and allocated active DHCP server pools for target environments.
* **Firewall Access Control Lists (ACLs):** Implemented a strict perimeter logic matrix under **Firewall > Rules**:
  * **Rule 1 (Data Ingestion):** Allowed `dmz_net` and `applications_net` to communicate with the SIEM host exclusively over `TCP 9997` and `UDP 1514`.
  * **Rule 2 (Adversary Isolation):** Explicitly `BLOCKED` any lateral communication originating from `attacker_net` destined for `siem_net`.
  * **Rule 3 (Exploit Testing):** Allowed `attacker_net` to route traffic into target zones (`dmz_net` and `applications_net`) for explicit detection testing.

### 2. SIEM Engine Deployment (Splunk Enterprise)
* Deployed Splunk Enterprise onto a headless Ubuntu Server instance (`192.168.10.102`).
* Configured local system layout settings and enabled automatic server booting persistence:
  ```bash
  sudo /opt/splunk/bin/splunk enable boot-start -user splunk
  sudo systemctl start splunk
  ```
* Configured global data listeners within **Settings > Data Inputs**:
  * Opened data receiver port **`TCP 9997`** for standard Universal Forwarder ingestion streams.
  * Opened data receiver port **`UDP 1514`** bound to the **`syslog`** source type for centralized security data parsing.

### 3. Log Ingestion & Endpoint Engineering (Universal Forwarder)
* **Windows Target Integration:** Deployed the Splunk Universal Forwarder binary, updating the local configuration layout (`C:\Program Files\SplunkUniversalForwarder\etc\system\local\outputs.conf`) to push log streams to the indexer node:
  ```ini
  [tcpout:ubuntu_siem]
  server = 192.168.10.102:9997
  ```
* **Linux Target Integration:** Deployed the `.deb` forwarder package on Ubuntu targets, modified group variables (`usermod -aG adm splunk`) to allow secure system visibility, and appended continuous file parsing metrics:
  ```bash
  sudo /opt/splunkforwarder/bin/splunk add monitor /var/log/syslog -sourcetype syslog
  sudo /opt/splunkforwarder/bin/splunk add monitor /var/log/auth.log -sourcetype linux_secure
  ```

### 4. Network Detection Engineering (Suricata IDS Integration)
* Provisioned Suricata NIDS at the perimeter layer.
* Updated configuration metrics (`/etc/suricata/suricata.yaml`) to track regional interfaces and group assignments:
  ```yaml
  vars:
    address-groups:
      HOME_NET: "[192.168.10.0/24,192.168.20.0/24,192.168.30.0/24]"
      EXTERNAL_NET: "!\$HOME_NET"
  
  af-packet:
    - interface: enp0s3
      cluster-id: 99
  
  community-id: true
  ```
* Handed off real-time intrusion monitoring data to the centralized forwarding line by forcing output channels directly into pfSense's syslog engine, routing clean events down custom pipelines straight to Splunk port **`1514`**.

---

## ⚔️ Threat Simulation & Verification Logs

To test and confirm that the entire defensive pipeline was operational, aggressive network reconnaissance scans were executed from the **Kali Linux Attacker node** targeting the internal lab gateways:
```bash
nmap -A -T4 192.168.20.1
```

### 🔍 Verification inside Splunk
Running the following query in the Splunk Search and Reporting UI confirms that network anomalies are successfully captured, routed across firewalls, and indexed:

```splunk
source="udp:1514" "suricata"
```

The system output displays functional alerts confirming the specific signature, category classification, and prioritization tier of the inbound attack vector.

---

## 💡 Key Skills Demonstrated
* **Security Operations:** SIEM Engineering (Splunk Ingestion), Core Configuration Analytics (`outputs.conf`, `inputs.conf`).
* **Network Security & Engineering:** Segmentation Design, Statefully Managed Virtual Firewalls (pfSense Architecture), Routing Matrix Definition, Access Control Lists (ACLs).
* **Threat Intelligence & Detection:** Intrusion Detection System Tuning (Suricata Rulesets), Log Parsing Frameworks (`syslog-ng`, `udpdump`).
* **System Administration:** Hardened Linux Administration (UFW management, group assignment logic), Windows Security Event management.
