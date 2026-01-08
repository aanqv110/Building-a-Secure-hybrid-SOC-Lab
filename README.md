# 🛡️ Hybrid SOC Lab – Active Directory + Microsoft Sentinel (Beginner Friendly)

## 📌 Overview

This project is a **hands-on security and infrastructure lab** designed to help beginners understand how to:

- Build an **on-prem Active Directory environment**
- Identify security and availability weaknesses
- Improve resilience using **Domain Controller redundancy**
- Centralize logs in **Azure**
- Deploy **Microsoft Sentinel** as a SIEM
- Detect and investigate **real authentication attacks**

The lab mirrors a **real consulting / enterprise scenario** using cost-effective tools.

---

## 🏦 Scenario

A small financial organization relies on an on-premises Active Directory environment but has:

- No centralized logging
- No real-time attack detection
- A single Domain Controller (single point of failure)
- No SOC or monitoring capability

You are asked to **assess the environment**, **improve security**, and **validate the solution** without redesigning everything or increasing costs significantly.

---

## 🧱 Architecture Overview

### On-Prem (VMware Workstation)
- **DC01** – Primary Domain Controller  
- **DC02** – Secondary Domain Controller (Disaster Recovery)  
- **Windows 10/11 Client** – User activity simulation  
- **Active Directory Domain:** `northbridge.local`

### Azure
- **Log Analytics Workspace**
- **Microsoft Sentinel**
- **Azure Arc**
- **Azure Monitor Agent (AMA)**
- **Data Collection Rules (DCRs)**

---

## 🧰 Prerequisites

### Local Machine
- Windows PC
- VMware Workstation
- Minimum 16 GB RAM recommended

### ISOs
- Windows Server 2022
- Windows 10 or Windows 11

### Azure
- Azure subscription (free tier is fine)
- Contributor permissions

---

## PART 1 — Build the Existing (Insecure) Infrastructure

### 1.1 Configure VMware Networking

1. Open **VMware Virtual Network Editor**
2. Ensure **VMnet8 (NAT)** exists
3. Enable:
   - NAT
   - DHCP (optional)
4. Note subnet (example):
   192.168.40.0/24


---

### 1.2 Create DC01 (Primary Domain Controller)

**VM Settings**
- Name: `DC01`
- OS: Windows Server 2022
- CPU: 2
- RAM: 4–8 GB
- Network: VMnet8 (NAT)

**Static IP Configuration**


IP Address: 192.168.40.10
Subnet Mask: 255.255.255.0
Gateway: 192.168.40.2
DNS: 192.168.40.10


---

### 1.3 Install Active Directory

1. Open **Server Manager**
2. Add Roles:
   - Active Directory Domain Services
   - DNS
3. Promote to Domain Controller
4. Create domain:


northbridge.local

5. Reboot

---

### 1.4 Baseline Issues (Before Improvement)

- ❌ Single Domain Controller
- ❌ No centralized logging
- ❌ No attack detection
- ❌ No disaster recovery
- ❌ No monitoring or alerting

---

## PART 2 — Improve Availability (Disaster Recovery)

### 2.1 Create DC02 (Secondary Domain Controller)

**VM Settings**
- Name: `DC02`
- OS: Windows Server 2022
- Network: VMnet8

**Static IP**


IP Address: 192.168.40.11
DNS: 192.168.40.10


---

### 2.2 Join DC02 to the Domain

1. Rename computer to `DC02`
2. Join domain `northbridge.local`
3. Reboot

---

### 2.3 Promote DC02

1. Install AD DS role
2. Promote as **Additional Domain Controller**
3. Use existing domain
4. Reboot

---

### 2.4 Validate Replication

On DC01:
```powershell

## repadmin /replsummary

Test failover:

Power off DC01

Authenticate via DC02

✅ Disaster recovery validated

---

## PART 3 — Client Machine Setup (User Activity Simulation)

## 3.1 Create the Windows Client VM

This VM represents a standard corporate user workstation and is used to simulate authentication activity and attacks.

**VM Configuration**
- Name: `WIN11-CLIENT`
- OS: Windows 10 Pro or Windows 11 Pro
- CPU: 2
- RAM: 4 GB
- Network: **VMnet8 (NAT)**

---

### 3.2 Configure Networking

You can use **DHCP** or a static IP.

**DNS Servers (IMPORTANT)**
192.168.40.10
192.168.40.11

yaml
Copy code

These point to DC01 and DC02.

---

### 3.3 Join the Domain

1. Open **System Properties**
2. Rename computer to `WIN11-CLIENT`
3. Join domain:
northbridge.local

yaml
Copy code
4. Reboot
5. Log in using a domain user account

✅ Client can now authenticate against Active Directory.

---

## PART 4 — Centralized Logging & Microsoft Sentinel

### 4.1 Create Log Analytics Workspace

In Azure Portal:

1. Go to **Log Analytics Workspaces**
2. Click **Create**
3. Use:
Name: law-nb-soc
Region: Same as Sentinel

yaml
Copy code

This workspace will store all security logs.

---

### 4.2 Enable Microsoft Sentinel

1. Go to **Microsoft Sentinel**
2. Click **Create**
3. Select workspace: `law-nb-soc`

This converts the workspace into a SIEM.

---

### 4.3 Onboard Domain Controllers to Azure Arc

Azure Arc allows Azure services to manage on-prem servers.

#### Steps:
1. Azure Portal → **Azure Arc**
2. Machines → **Add**
3. Choose **Add a single server**
4. Generate the onboarding PowerShell script
5. Run the script **as Administrator** on:
- DC01
- DC02

#### Verify Connection
```powershell
"C:\Program Files\AzureConnectedMachineAgent\azcmagent.exe" show
Status should show:

nginx
Copy code
Connected
4.4 Install Azure Monitor Agent (AMA)
AMA collects logs from on-prem servers.

For each Domain Controller:

Azure Portal → Azure Arc → Machines

Select DC01 / DC02

Extensions → Add

Install:

nginx
Copy code
AzureMonitorWindowsAgent
4.5 Create Data Collection Rule (DCR)
The DCR defines what logs are collected.

Azure Portal → Monitor

Data Collection Rules → Create

Platform: Windows

Data Sources
Windows Event Logs

Log name:

nginx
Copy code
Security
Destination
Azure Monitor Logs

Workspace: law-nb-soc

Resources
Assign:

DC01

DC02

⏱ Wait 5–10 minutes for ingestion.

4.6 Validate Log Ingestion
Run in Microsoft Sentinel → Logs:

Heartbeat (Agent Health)
kql
Copy code
Heartbeat
| summarize count() by Computer
Security Logs
kql
Copy code
SecurityEvent
| summarize count() by Computer
You should see DC01 and DC02.

PART 5 — Attack Simulation & Detection
5.1 Simulate Brute Force Authentication
From the client VM or a DC:

powershell
Copy code
for ($i=1; $i -le 6; $i++) {
  net use \\DC01\IPC$ /user:northbridge\BadUser WrongPass123
}
This generates Event ID 4625 (Failed Logon).

5.2 Detect Failed Logons (KQL)
kql
Copy code
SecurityEvent
| where EventID == 4625
| summarize FailedAttempts = count()
  by Account, Computer, bin(TimeGenerated, 5m)
| where FailedAttempts >= 5
This confirms the attack is visible.

5.3 Create Sentinel Analytics Rule
Microsoft Sentinel → Analytics

Create → Scheduled rule

Settings

Run every: 5 minutes

Lookup period: 5 minutes

Create incidents: Enabled

Entity Mapping

Account → Account

Host → Computer

Enable the rule.

5.4 Validate Incident Creation
Microsoft Sentinel → Incidents

Confirm:

New incident created

Severity: Medium

Account & Host populated

Open the incident and review:

Timeline

Entities

Raw events

PART 6 — Outcomes & Validation
Security Improvements Achieved
Area	Before	After
Availability	Single DC	Dual DC (HA)
Logging	Local only	Centralized
Detection	None	Real-time
SOC	None	Sentinel
DR	Untested	Validated

What This Lab Demonstrates
Active Directory architecture

Disaster recovery design

Azure Arc onboarding

Azure Monitor Agent & DCRs

Microsoft Sentinel operations

KQL detection & hunting

SOC-style incident response

PART 7 — Next Enhancements (Optional)
Automation playbooks (Logic Apps)

Defender for Endpoint integration

MITRE ATT&CK mapping

Sentinel Workbooks (dashboards)

Email / Teams alerting
