# active-directory-homelab
Automated AD detection &amp; response homelab using Splunk, Shuffle (SOAR), and Slack. Detects unauthorized authentication, alerts SOC via Slack, triggers a Shuffle playbook for analyst approval via email, then disables the compromised Domain User on the DC and confirms in Slack.

<img width="1568" height="635" alt="Pasted Graphic" src="https://github.com/user-attachments/assets/707963fc-548c-4bbf-a6cc-fcc224a9bcee" />


## How It's Supposed to Work
 
1. An attacker machine authenticates successfully against the test machine, a domain-joined Windows Server.
2. That authentication generates telemetry that flows to Splunk, which alerts on it.
3. Splunk posts to Slack and triggers a Shuffle playbook.
4. Shuffle emails the SOC analyst, asking whether to disable the account.
5. If the analyst says no, the playbook stops there. Nothing happens.
6. If the analyst says yes, Shuffle moves to the next step.
7. Shuffle instructs the Domain Controller to disable the domain user account.
8. Shuffle confirms in Slack that the account has been disabled, naming the account.
