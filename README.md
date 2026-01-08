# 🛡️ Beginner Guide: Building a Secure Hybrid SOC Lab with Active Directory & Microsoft Sentinel
## 🎯 What You Will Build (In Simple Terms)

You will create:
A small company network with Active Directory

Improve its security and availability

Send logs to Microsoft Sentinel

Detect and investigate real attacks

You do not need prior Sentinel or SOC experience.

##  🧠 Scenario (Why We Are Doing This)

A small financial company has:

An insecure on-prem Active Directory

No central monitoring

No disaster recovery

No way to detect attacks

You are asked to:

Assess the current setup

Improve security with minimal cost

Make it resilient

Prove it works

# PART 1 — DESIGN THE EXISTING (INSECURE) INFRASTRUCTURE
## 1.1 Tools You Need
Local Machine

Windows PC with:

VMware Workstation

Minimum 16 GB RAM recommended

## Software ISOs

Windows Server 2022 ISO

Windows 10 / Windows 11 ISO

## Azure

Free Azure account

Contributor access

## 1.2 Create the Virtual Network (VMware)

In VMware Workstation:

Open Virtual Network Editor

Ensure VMnet8 (NAT) exists

Enable:

NAT

DHCP (optional)

Note subnet (example):

192.168.40.0/24


👉 NAT allows internet access without breaking AD.

## 1.3 Create Domain Controller (DC01)
VM Settings

Name: DC01

OS: Windows Server 2022

CPU: 2

RAM: 4–8 GB

Network: VMnet8 (NAT)

Static IP (IMPORTANT)

On DC01:

IP: 192.168.40.10
Subnet: 255.255.255.0
Gateway: 192.168.40.2
DNS: 192.168.40.10

## 1.4 Install Active Directory

On DC01:

Server Manager → Add Roles

Install:

Active Directory Domain Services

DNS

Promote to Domain Controller

Create domain:

northbridge.local


Reboot.

## 1.5 Existing Problems (Before Improvements)

At this stage:

❌ Only one DC

❌ Logs stored locally

❌ No attack detection

❌ No monitoring

❌ Single point of failure

This is the baseline.

# PART 2 — IMPROVE AVAILABILITY (DISASTER RECOVERY)
## 2.1 Create Second Domain Controller (DC02)
## VM Settings

Name: DC02

OS: Windows Server 2022

Network: VMnet8

## Static IP
IP: 192.168.40.11
DNS: 192.168.40.10

## 2.2 Join DC02 to Domain

System → Rename PC → DC02

Join domain: northbridge.local

Reboot

## 2.3 Promote DC02

Install AD DS role

Promote as Additional Domain Controller

Use existing domain

Reboot

## 2.4 Validate DR

On DC01:

repadmin /replsummary


Test:

Power off DC01

Log into DC02

Authentication still works

✅ Disaster recovery achieved

# PART 3 — ADD A CLIENT MACHINE
## 3.1 Create Client VM

Name: WIN11-CLIENT

OS: Windows 10/11

Network: VMnet8

## IP (DHCP OK)

DNS:

192.168.40.10
192.168.40.11

# 3.2 Join Domain

Join domain northbridge.local

Reboot

Log in with domain user

# PART 4 — CONNECT ON-PREM TO AZURE (SECURITY VISIBILITY)
## 4.1 Create Log Analytics Workspace

Azure Portal:

Log Analytics → Create

Name:

law-nb-soc


Region: same as Sentinel later

## 4.2 Enable Microsoft Sentinel

Microsoft Sentinel → Create

Select workspace law-nb-soc

## 4.3 Install Azure Arc Agent (DC01 & DC02)

Azure Portal:

Azure Arc → Machines → Add

Generate onboarding script

Run script on:

DC01

DC02

Verify:

"C:\Program Files\AzureConnectedMachineAgent\azcmagent.exe" show


Status must be Connected.

## 4.4 Install Azure Monitor Agent (AMA)

For each DC:

Azure Arc → Machine → Extensions

Add AzureMonitorWindowsAgent

## 4.5 Create Data Collection Rule (DCR)

Azure Portal:

Monitor → Data Collection Rules → Create

Platform: Windows

Data source:

Windows Event Logs

Log: Security

Destination:

Log Analytics → law-nb-soc

Assign resources:

DC01

DC02

Wait 5–10 minutes.

## 4.6 Validate Logs

In Sentinel:

Heartbeat
| summarize count() by Computer


Then:

SecurityEvent
| summarize count() by Computer

# PART 5 — ATTACK DETECTION (SOC WORK)
## 5.1 Simulate an Attack

From client or DC:

for ($i=1; $i -le 6; $i++) {
 net use \\DC01\IPC$ /user:northbridge\BadUser WrongPass123
}

## 5.2 Detect Failed Logons
SecurityEvent
| where EventID == 4625
| summarize FailedAttempts=count()
  by Account, Computer, bin(TimeGenerated, 5m)
| where FailedAttempts >= 5

## 5.3 Create Analytics Rule

Type: Scheduled

Run every 5 minutes

Create incident

Map entities (Account, Host)

## 5.4 Investigate Incident

Sentinel → Incidents:

Review timeline

Review entities

Confirm brute-force attempt

# PART 6 — DOCUMENT OUTCOMES
Improvements Achieved
Area	Before	After
Availability	1 DC	2 DCs
Logging	Local	Central
Detection	None	Real-time
DR	None	Tested
SOC	None	Sentinel

## What You Learned

Active Directory architecture

Disaster recovery design

Azure Arc

Azure Monitor Agent

Microsoft Sentinel

KQL

SOC workflows

## Final Note (For Learners)

This lab mirrors:

Real SOC onboarding

Real Sentinel mistakes

Real enterprise constraints

