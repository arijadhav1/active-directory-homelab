# Home SOC Lab: Active Directory Detection and Automated Response with Splunk + Shuffle

This project is about building a real detection and response pipeline from scratch. The idea is simple: simulate a credential-based attack against a domain-joined machine, catch it in Splunk, and instead of just generating an alert that sits in a queue, have the entire response automated through a SOAR playbook that puts the decision in a human's hands before acting.

<img width="1036" height="405" alt="Screenshot 2026-06-17 at 10 25 02 PM" src="https://github.com/user-attachments/assets/8efb07d3-bc3c-4c32-9d4c-6243353dea2f" />

The full flow looks like this: an attacker authenticates to a target machine with valid credentials, Splunk fires an alert, Slack gets notified, Shuffle triggers a playbook, a SOC analyst receives an email asking whether to disable the compromised account, and if they say yes, Shuffle instructs the Domain Controller to disable the user and sends a final Slack confirmation.

This first part covers standing up the infrastructure and getting Active Directory running.

## The Setup

Everything runs on three cloud VMs hosted on Vultr, all in the same Silicon Valley region so they can communicate over a private VPC network.

| Machine | Role | Specs |
|---|---|---|
| Ari-ADDC01 | Domain Controller | 2 vCPU, 4GB RAM, 80GB storage |
| Cloud Instance | Target Machine | 1 vCPU, 2GB RAM, 55GB storage |
| Ari-Splunk | Splunk Server | 4 vCPU, 8GB RAM, 160GB storage |

The Splunk machine is intentionally beefier than the others. Log ingestion is resource-hungry and I didn't want performance issues getting in the way of the detection work.

<img width="1568" height="282" alt="Pasted Graphic" src="https://github.com/user-attachments/assets/be5d5546-d7a6-4b31-85b3-5115515513ae" />


## Network and Firewall

For firewall rules, I created a group called Ari-AD-Project and locked inbound access down to two rules: SSH on port 22 and RDP on port 3389, both restricted to my public IP only. Everything else drops by default.

<img width="1409" height="668" alt="Femall Groups  Manage All AD Projec" src="https://github.com/user-attachments/assets/8386f0c5-744f-49c5-bd11-82eff2724b01" />


All three machines are attached to the same VPC, which gives them private IP addresses they can use to communicate internally without bouncing traffic out to the internet. The VPC is region-specific, which is why picking Silicon Valley for all three mattered from the start.

One issue I ran into early: after enabling VPC on each machine, the network adapters were showing a 169.x.x.x address instead of the actual VPC IP. Had to go into the adapter settings on each Windows machine and manually configure the IP to match the assigned VPC address. After that, pings between machines worked fine.

<img width="1408" height="696" alt="Pasted Graphic 2" src="https://github.com/user-attachments/assets/2199540a-6a21-45ea-9f63-73cfab9407d9" />



## Installing Active Directory

With the network sorted, I remoted into the Domain Controller and installed Active Directory Domain Services through Server Manager under Add Roles and Features. Once the install finished, a flag notification appears in the top right prompting you to promote the server to a domain controller. I clicked that, created a new forest named `Ari.local`, set the directory services restore mode password, and let it run through the rest of the wizard.

The machine reboots automatically after promotion. When it comes back up, it's a fully functioning domain controller.

## Creating a Domain User

Inside Active Directory Users and Computers, I navigated to `Ari.local > Users`, right-clicked, and created a new user to act as the target account for the attack simulation.

**Name:** Rachel Wilson  
**Username:** RWilson

<img width="752" height="520" alt="Active Directory Users and Computen" src="https://github.com/user-attachments/assets/96dd4fcb-e309-4df7-8ef2-3ad899313586" />


## Joining the Target Machine to the Domain

On the target machine, I went to System Properties, clicked Rename this PC (Advanced), and changed the membership from Workgroup to Domain, entering `Ari.local`. Before this worked though, the machine couldn't find the domain at all.

The fix was a DNS change. The target machine's preferred DNS server needed to point to the Domain Controller's VPC address rather than a public DNS resolver. Active Directory relies on DNS to locate domain services, so without that pointing at the DC, the domain join has nowhere to look. Once I updated that in the adapter's IPv4 settings and retried, the join went through immediately.

<img width="396" height="453" alt="Internet Protocol Version 4 (TCPIPv4) Properties" src="https://github.com/user-attachments/assets/d8592ac4-6bff-4c71-8fdf-ded1611824bb" />


After a restart, I signed into the test machine as `Ari\RWilson`. One more thing to sort out: RWilson wasn't authorized for remote login by default. Had to go into the remote desktop user settings on the target machine, add RWilson explicitly, and confirm with the domain admin credentials from the DC.

<img width="1506" height="943" alt="Screenshot 2026-06-18 at 1 59 06 AM" src="https://github.com/user-attachments/assets/abad8183-040c-41e6-a685-0b464546b41d" />

<img width="376" height="332" alt="Remote Desktop Users" src="https://github.com/user-attachments/assets/34f4df85-998d-4167-9dbd-eac7a40e9ae7" />


With that done, the domain is up, the user is in place, and the target machine is joined and accessible. Next up is installing Splunk and configuring it to start ingesting telemetry from the Windows machines.ed the VPC adapter's preferred DNS server to point to the ADDC's private IP, then successfully joined the machine to the Ari.local domain. Configured RDP permissions to allow the domain user (RWilson) remote login access via the Remote Desktop Users group.


