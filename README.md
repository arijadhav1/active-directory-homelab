# active-directory-homelab
Automated AD detection &amp; response homelab using Splunk, Shuffle (SOAR), and Slack. Detects unauthorized authentication, alerts SOC via Slack, triggers a Shuffle playbook for analyst approval via email, then disables the compromised Domain User on the DC and confirms in Slack.

<img width="1036" height="405" alt="Screenshot 2026-06-17 at 10 25 02 PM" src="https://github.com/user-attachments/assets/8efb07d3-bc3c-4c32-9d4c-6243353dea2f" />


## How It's Supposed to Work
 
1. An attacker machine authenticates successfully against the test machine, a domain-joined Windows Server.
2. That authentication generates telemetry that flows to Splunk, which alerts on it.
3. Splunk posts to Slack and triggers a Shuffle playbook.
4. Shuffle emails the SOC analyst, asking whether to disable the account.
5. If the analyst says no, the playbook stops there. Nothing happens.
6. If the analyst says yes, Shuffle moves to the next step.
7. Shuffle instructs the Domain Controller to disable the domain user account.
8. Shuffle confirms in Slack that the account has been disabled, naming the account.


Active Directory Home Lab – Setup & Configuration
Provisioned three cloud-based Windows/Linux VMs on Vultr (Silicon Valley region):

Ari-ADDC01 – Domain Controller (2 vCPU, 4GB RAM, Windows Server 2025)
Test Machine – Domain Client (1 vCPU, 2GB RAM, Windows Server 2025)
Ari-Splunk – Log Ingestion Server (4 vCPU, 8GB RAM, Ubuntu 22.04)

Configured a Vultr Cloud Firewall group allowing inbound SSH (22) and RDP (3389) from a trusted IP, attached to all instances. Enabled VPC networking across all three VMs for private internal communication, and manually configured static VPC adapter settings on each machine to resolve IP assignment issues.
On the ADDC, installed the Active Directory Domain Services role via Server Manager, promoted the server to a Domain Controller, and created a new forest: Ari.local. Created a domain user account (Rachel Wilson / RWilson) for testing.
On the test machine, updated the VPC adapter's preferred DNS server to point to the ADDC's private IP, then successfully joined the machine to the Ari.local domain. Configured RDP permissions to allow the domain user (RWilson) remote login access via the Remote Desktop Users group.
