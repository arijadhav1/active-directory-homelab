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

## Phase 2: Installing Splunk and Configuring Telemetry
 
With the domain up and the machines talking to each other, the next step was getting visibility. That means Splunk on the Ubuntu server, universal forwarders on both Windows machines, and telemetry flowing into a centralized index.
 
### Setting Up Splunk Enterprise
 
Since I'm on a Mac, everything goes through Terminal over SSH. I SSH'd into the Splunk machine, updated the repositories, then headed to Splunk's site to grab the Enterprise free trial. I selected the Linux .deb package and copied the wget link directly into the SSH session to download it onto the server.
 
<img width="570" height="269" alt="root@Ari-Splunk-# wget -0 splunk-10 4 0-f798d4d49089-11nux-and64 deb" src="https://github.com/user-attachments/assets/1912d561-7b4d-465e-a5dd-3e6116ad4b25" />

 
Once downloaded, I installed it with `dpkg -i`, then navigated to `/opt/splunk/bin`.
 
<img width="513" height="131" alt="root@Ari-Splunkoptsplunk# 1s" src="https://github.com/user-attachments/assets/72b8957c-aaff-49cd-9a82-3d303990481b" />

 
Starting Splunk for the first time with `./splunk start --accept-license --answer-yes --run-as-root` initializes everything and is where you set the admin username and password for the web interface. The `--run-as-root` flag is required when running as root or Splunk won't start.
 
Accessing the web UI at `splunk-ip:8000` didn't work right away. Two things needed to happen: add a TCP rule for port 8000 in the Vultr firewall group restricted to my IP, and run `ufw allow 8000` inside the SSH session. After both, the login page loaded.
 
<img width="1383" height="359" alt="Aten 14" src="https://github.com/user-attachments/assets/87abb33e-b3c8-448d-b1fe-95e09711f5c3" />

 
<img width="823" height="431" alt="splunk› enterprise" src="https://github.com/user-attachments/assets/b326cecb-3c8e-42bd-b99d-8dd816929220" />

 
From there, a few configuration steps inside Splunk:
 
- Set timezone to GMT under preferences
- Installed the Splunk Add-on for Microsoft Windows from the app marketplace
- Created a new index called `ari-ad` under Settings > Indexes
<img width="1507" height="337" alt="Screenshot 2026-06-28 at 12 25 43 AM" src="https://github.com/user-attachments/assets/25d95af2-6437-4c4d-a1e1-266e91353af7" />

 
- Set up a receiving port on 9997 under Settings > Forwarding and Receiving

<img width="1506" height="298" alt="Receive data" src="https://github.com/user-attachments/assets/dee72358-6c11-418f-a418-c374003f5d52" />


 
### Installing the Universal Forwarder on the Target Machine
 
I downloaded the Splunk Universal Forwarder from Splunk's site and ran the installer on the Windows target machine. During setup I pointed the receiving indexer at the Splunk server's VPC IP on port 9997.
 
<img width="648" height="499" alt="image" src="https://github.com/user-attachments/assets/60adfbcb-6129-42e0-8467-97f87ba3fdfc" />
 
After install, I navigated to `C:\Program Files\SplunkUniversalForwarder\etc\system\local`. There was no `inputs.conf` file there by default, so I copied one over from the default directory and edited it with Notepad running as admin, adding the following at the bottom:
 
```
[WinEventLog://Security]
Index = ari-ad
Disabled = false
```
 
<img width="731" height="407" alt="inputsconf - Notecad" src="https://github.com/user-attachments/assets/9e901df3-001d-4cb3-8223-f2daeaf5897e" />

 
Then in Services, I opened the SplunkForwarder service, switched the log on account to Local System, and restarted it.
 
<img width="417" height="483" alt="SplunkForwarder Properties (Local Computer)" src="https://github.com/user-attachments/assets/0a587bf1-f226-484d-a299-3b8b7eeb8b01" />

 
After restarting, I went back to Splunk and ran `index=ari-ad` in Search and Reporting. Nothing came back at first. The fix was running `ufw allow 9997` on the Splunk server to open the forwarder port. After that, events started flowing in.
 
I repeated the same process on the Domain Controller. Once both forwarders were running, checking the host field in Splunk showed two values, confirming telemetry was coming in from both machines.
 
### Building the Alert
 
With telemetry flowing, I built a Splunk alert to catch successful RDP logins coming from outside the expected network.
 
Windows Security Event ID 4624 covers successful logon events. RDP sessions show up as Logon Type 7 or 10. I built the search out step by step:
 
```
index=ari-ad EventCode=4624 (Logon_Type=7 OR Logon_Type=10) Source_Network_Address=* Source_Network_Address!="-" Source_Network_Address!="40.*"
| stats count by _time, ComputerName, Source_Network_Address, user, Logon_Type
```
 

The `Source_Network_Address=*` filter removes events with no network address, `!="-"` drops local system noise, and `!="99.*"` filters out my own authorized IP so only unexpected sources trigger the alert.

<img width="1411" height="444" alt="New Search" src="https://github.com/user-attachments/assets/01b8be71-20ef-4454-b2ae-301271858a85" />


To generate a test event, I updated the Vultr firewall rule to allow RDP from anywhere instead of just my IP, then RDP'd in from the attacker machine. The plan was originally to use a local Kali VM in UTM, but OpenVPN was throwing errors during install so I spun up a lightweight Debian machine on Vultr instead and named it attacker. Same result, less friction.

<img width="1400" height="444" alt="Pasted Graphic 2" src="https://github.com/user-attachments/assets/888e87b1-5fe0-4b1b-8d42-bf81fd89768d" />


I saved the search as an alert named `Ari-Unauthorized-Successful-Login-RDP`, set it to run on a cron schedule every minute against the last 60 minutes of data (`* * * * *`), with severity set to Medium. Within a minute of saving it, the alert appeared under Activity > Triggered Alerts.


Next up is connecting Splunk to Shuffle and building the automated response playbook.

## Troubleshooting

A few things went sideways during this phase that are worth documenting.

SSH from Terminal was hanging on connection without throwing an error. I used Nmap to verify the port was actually reachable, found that my public IP had changed, updated the firewall rule, and it connected fine after that.

When I first ran `./splunk start`, it skipped the license agreement and credential setup entirely. Turns out running Splunk as root without explicitly passing the flag is deprecated. The fix was using `./splunk start --accept-license --answer-yes --run-as-root`.

After getting the forwarder running on the test machine and restarting the service, no events were showing up in Splunk. The issue was that I had disconnected and reconnected my SSH session without restarting Splunk on the server. Once I started it again on the Ubuntu machine, telemetry started flowing immediately.

Next part is creating telemetry and using Shuffle to create a playbook.
