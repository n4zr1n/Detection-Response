Executive Summary

Modern Security Operations Centers face an overwhelming volume of alerts that often lead to analyst fatigue and delayed incident response. Traditional manual containment strategies often fail to neutralize active threats before significant enterprise compromise occurs. This project demonstrates the design, deployment, and practical validation of an automated Security Orchestration, Automation, and Response ecosystem integrated directly with a central Security Information and Event Management platform, a Web Application Firewall, and remote endpoint management components.
By unifying Splunk Enterprise, ModSecurity WAF, Shuffle SOAR, and Windows Remote Management into a single cohesive pipeline, this implementation successfully converts passive security monitoring into proactive threat containment. The core outcomes of this integration include real-time ingestion of web application firewall audit logs for immediate detection of OWASP Top 10 attack vectors, interactive human-in-the-loop analyst approvals delivered via Telegram and Discord, and remote PowerShell execution on target endpoints to isolate compromised user accounts, block unauthorized communication ports, and terminate malicious host processes. Ultimately, this architecture drastically reduces the Mean Time to Respond from hours to under thirty seconds for critical security incidents.

Introduction & Technical Stack

The primary objective of this project is to simulate an end-to-end enterprise SOC workflow within a lab environment. The infrastructure relies on lightweight Docker containers hosting the application and firewall layers, a central Splunk instance for correlation and alert generation, a Shuffle SOAR engine for workflow logic, and a target Windows Server host for automated remediation.
The application layer consists of Damn Vulnerable Web Application running behind an Nginx web server configured with ModSecurity and the OWASP Core Rule Set. As incoming HTTP traffic flows through ModSecurity, malicious request logs are immediately forwarded to Splunk Enterprise for indexing and real-time detection via custom Search Processing Language queries. Upon detecting a threat, Splunk sends a JSON payload via Webhook to Shuffle SOAR. Shuffle parses the alert data, generates interactive notification prompts to analysts through Telegram and Discord, and waits for a single-click confirmation. Once approved by an analyst, Shuffle executes remote PowerShell containment scripts over WinRM directly on the Windows Server endpoint to neutralize the threat.

Use Case 1: Automated XSS Attack Incident Response & Containment

1. Overview & Objective
This use case demonstrates an end-to-end automated detection and containment workflow for Cross-Site Scripting (XSS) attacks. By integrating Web Application Firewall (WAF) logs, a SIEM platform (Splunk), and a SOAR platform (Shuffle), the system automatically captures malicious traffic, parses the attacker's IP address, requests human analyst authorization, and dynamically blocks the threat on the host firewall via WinRM.

2. Technical Architecture & Environment Setup
•	Target Application: Damn Vulnerable Web Application (DVWA) running in a Docker environment.
•	Web Application Firewall (WAF): Nginx integrated with ModSecurity (OWASP Core Rule Set).
•	SIEM Platform: Splunk Enterprise (ingesting ModSecurity audit logs in real-time).
•	SOAR Platform: Shuffle SOAR (handling webhook triggers, enrichment, conditions, human approval, and automation).
•	Target Endpoint / Response Agent: Windows Server / Host Managed via WinRM and PowerShell commands.

3. Step-by-Step Incident Execution & Workflow

Step 1: Attack Simulation (DVWA)

An attacker injects a malicious Reflected XSS payload into the search input field of the DVWA web application running at http://localhost:7575.

<img width="605" height="367" alt="image" src="https://github.com/user-attachments/assets/8330f0e9-835a-45ff-b687-a91640065682" />


Step 2: WAF Interception & Detection

ModSecurity WAF analyzes the HTTP request, identifies the pattern match against the OWASP Core Rule Set (XSS Injection), blocks the malicious execution, and returns an HTTP 403 Forbidden response to the attacker.

<img width="605" height="114" alt="image" src="https://github.com/user-attachments/assets/d3fce80c-d0db-40a8-90ef-b5a6d87639a1" />


Step 3: Log Ingestion & Real-Time Alerting (Splunk SIEM)

ModSecurity generates structured JSON audit logs detailing the transaction and client request parameters. Splunk ingests these logs under sourcetype=_json and identifies the XSS Attack Detected event.

<img width="605" height="342" alt="image" src="https://github.com/user-attachments/assets/f112793a-cb4a-4a02-a8da-c36e375670c2" />

Logs are saved as alerts. A real-time Splunk Alert (ModSecurity XSS Attack Detected) is triggered, configured to execute a Webhook action sending the event payload directly to Shuffle SOAR. 

<img width="448" height="459" alt="image" src="https://github.com/user-attachments/assets/ff24eeb2-e7ba-4826-9430-03f4b3c3562c" />

Step 4: SOAR Workflow Initiation & Data Parsing (Shuffle)

Shuffle workflow:

<img width="605" height="322" alt="image" src="https://github.com/user-attachments/assets/a662bbee-e0ff-4db6-b0a6-6367f5178872" />

Webhook Trigger: Shuffle receives the HTTP alert payload from Splunk.
IP Address Extraction (Shuffle Tools 1): A parsing node extracts the attacker’s source IP address from the log JSON structure using the variable field path $exec.result.transaction_remote_address.

<img width="605" height="323" alt="image" src="https://github.com/user-attachments/assets/ded24091-32cf-4e1a-9d5d-673f372c0a94" />

Step 5: Whitelist Condition Handling

To prevent accidental blocking of critical internal infrastructure or trusted networks (like local gateway 172.18.0.1), a conditional branch evaluates whether the extracted IP matches the whitelist:

<img width="605" height="323" alt="image" src="https://github.com/user-attachments/assets/8987f981-0f06-4b08-a158-5d324c727a9f" />

Condition: $shuffle_tools_1 DOES NOT EQUAL 172.18.0.1
Result: Whitelisted IP execution paths are halted safely, while external threat addresses proceed to the containment stage.

<img width="605" height="323" alt="image" src="https://github.com/user-attachments/assets/93da3850-d305-4b0f-98f5-3ea97d462914" />


Step 6: Human-in-the-Loop Analyst Approval (XSS Block Approval)

To eliminate false positives and maintain SOC oversight:

<img width="605" height="321" alt="image" src="https://github.com/user-attachments/assets/b45a85db-2a85-49cd-9681-3d7be85db924" />

The playbook enters a WAITING state. An interactive approval prompt is generated with contextual details: "XSS attack detected! Attacker IP address: 185.220.101.5. Do you allow this IP address to be blocked?" 

<img width="605" height="319" alt="image" src="https://github.com/user-attachments/assets/c867c757-6587-4739-b25c-bf813621aff8" />

The SOC analyst verifies the incident and triggers the approval link (/api/v1/workflows/.../execute), changing the status to SUCCESS and resuming workflow execution.

<img width="605" height="81" alt="image" src="https://github.com/user-attachments/assets/dd66fdc3-57fb-4b8f-aa38-d57f31cc650d" />



Step 7: Automated Response & Containment (Shuffle Tools 2)
Upon approval, a Python script (pywinrm) executes inside the workflow node:

<img width="605" height="321" alt="image" src="https://github.com/user-attachments/assets/3febf346-4ec0-40ee-bed7-678a3d510e30" />

It establishes a secure WinRM session to the target Windows endpoint and executes a remote PowerShell script to generate a new inbound firewall rule:
New-NetFirewallRule -DisplayName "Block Attacker IP - 185.220.101.5" -Direction Inbound -Action Block -RemoteAddress 185.220.101.5
The WinRM service returns a success payload ("success": true), confirming that the malicious IP address has been isolated.

<img width="605" height="341" alt="image" src="https://github.com/user-attachments/assets/1a21169d-ad6c-4fc7-b883-1bad97be8c59" />

4. Verification & Validation

The remediation action was verified directly on the target host by executing the following PowerShell command in an elevated prompt:
Get-NetFirewallRule -DisplayName "Block Attacker IP*" | Select-Object DisplayName, Enabled, Direction, Action
Output Confirmation:

<img width="605" height="59" alt="image" src="https://github.com/user-attachments/assets/349a9da9-b4be-4e23-8d31-ec33cb94b8c6" />

DisplayName: Block Attacker IP: 185.220.101.5

Enabled: True

Direction: Inbound

Action: Block

5. Conclusion

This use case demonstrates a complete SOC automation lifecycle: 
Threat Detection -> SIEM Alerting ->  Context Enrichment ->  Analyst Oversight ->  Automated Network Isolation.


Use Case 2: Brute Force Attack Detected with Successful Logon & Automated Mitigation

Platform: Shuffle SOAR, Python (pywinrm), Active Directory / Windows Server

1. Executive Summary

This security incident report details the investigation and automated response workflow for the use case: "Brute Force Attack Detected & Successful Logon". The automated incident response playbook was implemented within the SOC (Security Operations Center) environment using the Shuffle SOAR platform. 
Security monitoring systems registered multiple failed authentication attempts followed by a successful logon event on the target account. This triggered an automated containment playbook to isolate the compromised credentials and prevent potential lateral movement within the network.

2. Incident Analysis & Technical Findings

The incident originated from a SIEM alert payload forwarded to Shuffle SOAR. The incoming payload flagged a dictionary/brute-force attack pattern resulting in account compromise.

Step 1: Brute Force Detection in Splunk (SPL Query)

To detect potential brute-force attacks, an SPL (Splunk Processing Language) query is executed in Splunk Enterprise to search for failed logon attempts:

<img width="605" height="183" alt="image" src="https://github.com/user-attachments/assets/393a15af-4b6c-4126-8b15-e4641a177c8e" />

•	EventCode=4625: Filters Windows Security log events corresponding to failed account logons.

•	mvexpand & search: Expands multi-value fields and filters out empty values (-).

•	stats count by ...: Calculates the total number of failed login attempts grouped by user account.

Result: The user naz_testuser registered 84 failed logon events, indicating a ongoing brute-force attempt against this account.

Step 2: Configuring Real-Time Webhook Alert in Splunk

An alert named "Brute-Force Successful Logon Detected" (or Brute-Force Detection Alert) is configured in Splunk:
<img width="384" height="398" alt="image" src="https://github.com/user-attachments/assets/bfcfabce-2728-4b14-9260-13fdc7f7e9fb" />

Step 3: Simulating the Brute-Force Attack (PowerShell)

A brute-force attack is simulated against the target account naz_testuser using Windows PowerShell:

<img width="565" height="460" alt="image" src="https://github.com/user-attachments/assets/40c0dd1b-f22c-49e6-83f4-9fb79c78b7fd" />

Multiple net use commands are executed using incorrect passwords, resulting in Windows Error 1326 (Kullanıcı adı veya parola hatalı / Incorrect username or password).
Finally, the correct password credentials are entered, yielding the message "Komut başarıyla tamamlandı." (The command completed successfully), indicating a successful authentication after multiple failed attempts.

Step 4: Splunk Event Logs View

This view displays the raw Security Event Logs captured in Splunk:
<img width="605" height="325" alt="image" src="https://github.com/user-attachments/assets/8b136ed0-ecbf-4ffa-abd4-f6dbc1238634" />

LogName=Security and EventCode=4625 indicate logged failed attempt events. Specific event metadata such as host name (BB16675), timestamps, and sourcetype (WinEventLog:Security) confirm that Windows event logs are actively ingested and analyzed by Splunk.

Step 5: Shuffle Automation Workflow Setup

A automated SOAR workflow titled Brute-Force-Account-Disable is created in Shuffle:

<img width="605" height="285" alt="image" src="https://github.com/user-attachments/assets/a1cabe42-0dfa-471f-ab34-54cc24fcb446" />

•	Webhook 1: Receives the alert payload sent by Splunk when the brute-force condition is met.

•	Shuffle Tools 1: Parses incoming JSON data to extract the targeted username (naz_testuser).

•	Account Disable Approval: Sends an interactive approval node/notification asking whether to proceed with disabling the user account.

•	Sub-action / Execution Node: Executes the remote script upon confirmation/triggering.

Step 6: API Execution / Approval Endpoint Confirmation

This browser tab shows the execution response from the API approval endpoint.

<img width="605" height="115" alt="image" src="https://github.com/user-attachments/assets/724cebfb-bdea-4f4a-830e-a57c2ae6bd08" />

The response returning "success": true confirms that the authorization link/callback was successfully invoked to trigger the next step of the workflow.

Step 7: Python Remediation Script (WinRM Account Disabling)

A Python script running in Shuffle executes the automated mitigation action over WinRM (Windows Remote Management):

<img width="482" height="487" alt="image" src="https://github.com/user-attachments/assets/9efe1b78-29e8-46c0-bc15-a1a859f3a34c" />

It automatically verifies and installs the pywinrm package if not present, extracts target_user passed from Shuffle_Tools_1 (naz_testuser), connects to the host using WinRM and executes the PowerShell command that disables the compromised user account on the target machine automatically.

Step 8: Verifying Account Status (PowerShell)

<img width="605" height="66" alt="image" src="https://github.com/user-attachments/assets/f684f3d7-5d03-489a-aa6b-f69c4d6acc48" />

Output returns Enabled: False, confirming that the account naz_testuser has been successfully disabled by the automation workflow.

Step 9: Testing Post-Mitigation Logon Attempt

Testing logon functionality after the account has been disabled:

<img width="605" height="54" alt="image" src="https://github.com/user-attachments/assets/135a68cd-7d2c-43ce-ab7c-192d5fc65003" />

It validates that the end-to-end detection, alerting, and automated response pipeline successfully neutralized the threat.

Use Case 3: Arbitrary Command Execution (Remote Code Execution)

Detection & Workflow Triggering:

ModSecurity Web Application Firewall (WAF) detects a command injection attempt and sends the alert log to Splunk SIEM. Splunk triggers a real-time Webhook alert to Shuffle SOAR, initiating the alertt playbook workflow to parse incoming attack metadata.

<img width="551" height="331" alt="image" src="https://github.com/user-attachments/assets/1b94ccaf-aa25-4b6e-9456-07d3c96b5968" />

Vulnerability Identification & Attack Execution:

The target application's command injection interface was identified. The input field passes user data directly to the host shell without validation. A malicious payload combining a valid IP with an injected shell command (127.0.0.1; cat /etc/passwd) was submitted through the field to execute arbitrary code.  

<img width="559" height="336" alt="image" src="https://github.com/user-attachments/assets/22c16fa0-d492-44f6-a61a-c04c12d3552c" />

SOAR Parsing & Data Extraction:

Shuffle SOAR receives the POST request webhook containing payload details. The workflow executes internal tools (Shuffle Tools 1 and Shuffle Tools 2) to parse the JSON data, successfully extracting the attacker's source IP, request parameters, and targeted host details.  

<img width="176" height="300" alt="image" src="https://github.com/user-attachments/assets/660b4074-f5d6-45af-8940-8af3644e493d" />

SIEM Analysis & Payload Detection:

Splunk Enterprise ingests the raw ModSecurity WAF transaction logs. Using SPL regex extraction, Splunk parses the attacker_ip (172.18.0.1) and identifies command chaining operators (;&|) alongside target system commands (cat, whoami, ping):

<img width="553" height="331" alt="image" src="https://github.com/user-attachments/assets/aa7e6578-38a7-4804-acd4-8f3d49445245" />

Validation & Repeatability Analysis:

A second payload was transmitted to confirm exploitability. Splunk logs confirm that the injected command was carried unmodified to the server shell, verifying repeatable command execution on the target.

<img width="564" height="336" alt="image" src="https://github.com/user-attachments/assets/f09cc68d-0eab-4646-8ecf-424e22216704" />

Remote WinRM Connection & Automated Remediation Action:

To execute automated actions, the SOAR platform connects to the host via WinRM over TCP port 5985. Once connected, SOAR automatically executes PowerShell remediation scripts on the host:

•	New-NetFirewallRule: Creates a firewall rule ("Allow WinRM HTTP" / "Block Attacker IP") to enforce host isolation and secure remote management. 

•	Stop-Process: Terminates any malicious child processes spawned by the web server. 

•	Disable-LocalUser: Disables compromised local accounts if unauthorized access is flagged.

•	Restart-Service: Restores web services to guarantee operational baselines.

•	Collect forensic artifacts: Dumps network sockets, active processes, and logs for SOC investigation.

<img width="522" height="316" alt="image" src="https://github.com/user-attachments/assets/b4d30d84-dc3a-443a-a4af-83dc3447f141" />

Use Case 4: Restarting a System Service

Baseline Status Check:

<img width="560" height="197" alt="image" src="https://github.com/user-attachments/assets/08e384da-cfc5-4a9c-b270-1fdab8f4460b" />

Before triggering the scenario, the operational status of the target host service (Spooler - Print Spooler) was verified on the host machine using PowerShell (Stop-Service -Name Spooler, Get-Service -Name Spooler). The baseline check confirms the service was initially in a stopped/running state prior to automated orchestration testing.

Alert Payload Structuring:

Within Shuffle SOAR, the notification node is configured with JSON data to format critical alert messages. The expected output structure defines the exact alert parameters, including the target host IP (172.22.90.104) and affected service name (Spooler), preparing it for real-time SOC alerting:  

<img width="501" height="300" alt="image" src="https://github.com/user-attachments/assets/43abddaf-cf6f-4e07-b898-c319bac6c3c8" />

Communication Channel Authorization:

A dedicated Telegram bot (SOAR Alert Bot / @my_soar_ln0412_bot) was created via BotFather and initialized with an HTTP API token. This provides an authenticated webhook channel for Shuffle SOAR to transmit real-time incident notifications and recovery statuses directly to SOC analysts. 

<img width="378" height="300" alt="image" src="https://github.com/user-attachments/assets/c9ad66fb-8701-4595-b4fd-406c210ee260" />

Workflow Payload Validation:

The playbook execution engine validates the formatted alert payload in Shuffle SOAR. The system verifies that the JSON structure correctly maps the target host IP and service parameters before initiating automated remote action and alert transmission. 

<img width="539" height="300" alt="image" src="https://github.com/user-attachments/assets/eec1e76b-cc71-47da-b825-f9afb2c9f5f9" />

SOAR Orchestration & Automated Service Restoration:

Upon receiving the incident trigger, the Shuffle SOAR workflow (Malicious-Process-AutoResponse) executes automatically. The node Shuffle Tools 1 executes a Python script that connects to the target host via WinRM/SSH and runs PowerShell commands (Restart-Service / Start-Service -Name Spooler) to automatically restore the stopped service. Simultaneously, the http 1 node posts an automated alert to the Telegram API (/sendMessage), returning HTTP 200 success. 

<img width="536" height="323" alt="image" src="https://github.com/user-attachments/assets/8823bb0d-1a0c-45ee-bd2b-4548d8595034" />

Use Case 5: Stopping a Running Process

Baseline Feature & System State Audit:

The target system configuration is audited to confirm installed host features and baseline services (such as OpenSSH Sunucusu / OpenSSH Server). Establishing this operational baseline ensures that critical services and processes are actively monitored prior to testing process termination scenarios. 

<img width="215" height="300" alt="image" src="https://github.com/user-attachments/assets/3b6b03ca-61c3-470a-905e-e9ff3b2655eb" />

Malicious / Monitored Process Baseline:

A target process is set up and monitored on the host machine. A continuous PowerShell execution loop (while($true){ Start-Sleep -Seconds 1; Write-Host "Proses aktivdir..." }) runs in the foreground, establishing an active process baseline to simulate a critical operational task or monitoring agent. 

<img width="507" height="300" alt="image" src="https://github.com/user-attachments/assets/527a1f0f-019e-42f1-88ce-bc1c7cd5a86b" />

Automated Detection & SOAR Execution:

An unauthorized process-kill payload (e.g., Stop-Process or taskkill) is injected through the vulnerable web interface. Shuffle SOAR receives the incident trigger via Webhook 1 and executes the Malicious-Process-AutoResponse playbook. The Shuffle Tools 1 node runs automated response scripts while the http 1 node issues an automated POST request to the Discord Webhook endpoint, returning HTTP 204 success.

<img width="501" height="300" alt="image" src="https://github.com/user-attachments/assets/8d4f3282-4054-488e-8ba0-a04da504fe79" />

SOAR Alert Payload Structuring:

Inside Shuffle SOAR, the expected output JSON structure is configured for Discord alerting. The playbook formats an automated high-severity incident notification containing the process status:  

<img width="529" height="300" alt="image" src="https://github.com/user-attachments/assets/f4f1c817-a6f5-4128-a7b6-0c6d6b2b304c" />

SIEM Alerting & Automation Trigger:

In Splunk Enterprise, a dedicated scheduled alert rule named Suspicious PowerShell Execution is configured. Under Trigger Actions, a Webhook action is defined with the SOAR endpoint URL. When Splunk detects unauthorized process termination logs, it automatically sends a POST payload to Shuffle to initiate immediate containment and SOC notification. 

<img width="293" height="300" alt="image" src="https://github.com/user-attachments/assets/aa3c6e9a-713a-4db4-ab6d-cd4d59cf74e4" />

Production Environment Considerations

While this laboratory deployment utilizes lightweight protocols for demonstration and testing purposes, implementing these playbooks in a enterprise production environment requires the following security enhancements:

WinRM Transport Security: WinRM operates over HTTP (Port 5985) with Basic Authentication in this test setup. For production, WinRM over HTTPS (Port 5986) secured with Kerberos or NTLM authentication and valid TLS certificates should be enforced.

Webhook Authentication: Shuffle SOAR Webhook endpoints must be hardened using source IP whitelisting, strict request header validations, and Bearer Tokens to prevent unauthorized workflow triggers.

Least Privilege Access: Automated scripts executed via SOAR should run under service accounts bound by the Principle of Least Privilege (PoLP) rather than full Domain Admin rights.

Conclusion & Future Work

This project successfully established an end-to-end automated detection and response pipeline leveraging Splunk SIEM, Shuffle SOAR, and remote PowerShell management via WinRM. 

Future Enhancements:

•	Integration of Elastic Agents and Sysmon for deeper endpoint process visibility.

•	Expanding SOAR workflow integrations with open-source ticketing and case management platforms such as TheHive or Jira.

•	Implementing automated threat intelligence enrichment via VirusTotal and AbuseIPDB APIs prior to analyst notification.
