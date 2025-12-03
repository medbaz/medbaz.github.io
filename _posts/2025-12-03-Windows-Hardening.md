---
title: "Windows Hardening"
categories:
  - Edge Case
tags:
  - content
  - css
  - edge case
  - lists
  - markup
---

### **System Services & Monitoring**
**Role:** To understand and manage the core background processes, system configuration database, and event logs. This is fundamental for troubleshooting and spotting unauthorized changes.

| Technique                 | Explanation                                                                                                   | Command / UI Action                                                      |
| :------------------------ | :------------------------------------------------------------------------------------------------------------ | :----------------------------------------------------------------------- |
| **Manage Services**       | Control background processes that handle critical functions like networking and security.                     | `services.msc` (Run command)                                             |
| **Edit Windows Registry** | Access the hierarchical database that stores low-level settings for the OS and applications.                  | `regedit` (Run command)                                                  |
| **Review System Logs**    | Use Event Viewer to inspect logs for errors, warnings, and other system events crucial for diagnosing issues. | `Eventvwr.msc` (Run command)                                             |
| **Check Data Telemetry**  | Interact with the service that collects system data for improving user experience and identifying issues.     | `services.msc` > Find "Connected User Experiences and Telemetry" service |

### **Identity & Access Management**

| Technique                      | Explanation                                                                                                  | Command / UI Action                                                                                                         |
| :----------------------------- | :----------------------------------------------------------------------------------------------------------- | :-------------------------------------------------------------------------------------------------------------------------- |
| **User Account Control (UAC)** | A fundamental security feature that prevents unauthorized changes by prompting for admin consent.            | `Control Panel` > `User Accounts` > `Change User Account Control settings`                                                  |
| **Local Group Policy Editor**  | Configure system-wide policies for users and computers in a non-domain environment (Windows Pro/Enterprise). | `gpedit.msc` (Run command)                                                                                                  |
| **Password Policies**          | Enforce strong password creation rules (length, complexity, history) to resist brute-force attacks.          | `gpedit.msc` > `Computer Config` > `Windows Settings` > `Security Settings` > `Account Policies` > `Password Policy`        |
| **Account Lockout Policy**     | Mitigate password guessing by locking accounts after a set number of failed login attempts.                  | `gpedit.msc` > `Computer Config` > `Windows Settings` > `Security Settings` > `Account Policies` > `Account Lockout Policy` |


### **Network Management**
**Role:** To secure the computer's network interfaces and communications, preventing unauthorized access and common network-based attacks like spoofing.

| Technique | Explanation | Command / UI Action |
| :--- | :--- | :--- |
| **Windows Firewall Configuration** | Control inbound and outbound network traffic based on rules. The first line of defense against network threats. | `wf.msc` (Run command) |
| **Disable Unused Network Adapters** | Reduce the system's attack surface by disabling network hardware that is not in use (e.g., unused Wi-Fi, VPN adapters). | `devmgmt.msc` > `Network adapters` > Right-click adapter > `Disable device` |
| **Inspect Hosts File** | Check for malicious DNS redirections, as attackers can modify this file to redirect legitimate traffic to malicious sites. | Navigate to `C:\Windows\System32\drivers\etc\hosts` |
| **Manage ARP Cache** | Prevent ARP poisoning (Man-in-the-Middle) attacks by viewing and clearing the cache that maps IP addresses to physical MAC addresses. | View: `arp -a`<br>Clear: `arp -d` (Command Prompt) |
| **Disable Remote Desktop** | If remote access is not needed, disable this service to close a significant potential entry point for attackers. | `Settings` > `System` > `Remote Desktop` > Toggle to **Off** |

### **Application Management**
**Role:** To control which applications can run and how they behave, protecting the system from malware and potentially unwanted software.

| Technique                         | Explanation                                                                                                                                     | Command / UI Action                                                                                                        |
| :-------------------------------- | :---------------------------------------------------------------------------------------------------------------------------------------------- | :------------------------------------------------------------------------------------------------------------------------- |
| **Restrict App Sources**          | Prevent users from installing untrusted, potentially malicious software from outside the curated Microsoft Store.                               | `Settings` > `Apps` > `Apps & features` > Select `The Microsoft Store only`                                                |
| **Review Defender Exclusions**    | Ensure critical file types are not excluded from antivirus scans, which could allow malware to go undetected.                                   | `Windows Security` > `Virus & threat protection` > `Manage settings` > `Exclusions`                                        |
| **Application Hardening Scripts** | Run specialized scripts (e.g., `office.bat`) to apply security settings that disable vulnerable features in applications like Microsoft Office. | Right-click `office.bat` > `Run as administrator`                                                                          |
| **Configure AppLocker**           | Create whitelisting or blacklisting rules to control exactly which executables, scripts, and installers are allowed to run.                     | `gpedit.msc` > `Computer Config` > `Windows Settings` > `Security Settings` > `Application Control Policies` > `AppLocker` |
| **Browser Hardening (Edge)**      | Enable security features like SmartScreen to block phishing/malware sites and strict tracking prevention to enhance privacy.                    | `Edge Settings` > `Privacy, search, and services` > Set `Tracking prevention` to **Strict**                                |

### **Data Encryption & Protection**
**Role:** To protect data at rest and create isolated execution environments, ensuring confidentiality and integrity even if the device is lost or stolen.

| Technique                    | Explanation                                                                                                                   | Command / UI Action                                                                            |
| :--------------------------- | :---------------------------------------------------------------------------------------------------------------------------- | :--------------------------------------------------------------------------------------------- |
| **Enable BitLocker**         | Use full-disk encryption to render data unreadable without the recovery key, protecting it from physical theft.               | `Control Panel` > `System and Security` > `BitLocker Drive Encryption` > `Turn on BitLocker`   |
| **Secure Boot Verification** | Check that the system uses Secure Boot, which ensures only trusted software is loaded during the startup process.             | `msinfo32` (Run command) > Find "Secure Boot State"                                            |
| **Enable Windows Sandbox**   | Create a temporary, disposable desktop environment to safely run untrusted software without affecting the host OS.            | Search "Windows Features" > `Turn Windows features on or off` > Check `Windows Sandbox` > `OK` |
| **Configure File Backups**   | Set up a routine backup plan to ensure data can be recovered in case of ransomware, hardware failure, or accidental deletion. | `Settings` > `Update & Security` > `Backup`                                                    |

### **Windows Update**
**Role:** To keep the operating system patched against known vulnerabilities, which is one of the most critical steps in maintaining a secure system.

| Technique | Explanation | Command / UI Action |
| :--- | :--- | :--- |
| **Check for Updates** | Manually trigger a search for the latest security patches and feature updates from Microsoft. | `Settings` > `Update & Security` > `Windows Update` > `Check for updates` |
