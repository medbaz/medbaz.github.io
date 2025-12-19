---
title: "Windows Processes Explained A SOC Analysts Guide to Understanding Windows Processes"
categories:
  - SOC Analysis
tags:
  - windows-processes
  - system-security
  - threat-hunting
---

## **Unmasking the Digital Underworld: A SOC Analyst's Guide to Windows Processes**

Ever wondered what's really happening inside your computer? While you're browsing the web or typing a document, a silent army of programs is working tirelessly in the background. These are **Windows Processes**, and understanding them is the secret weapon for any Security Operations Center (SOC) analyst, incident responder, or threat hunter. Think of your Windows system as a bustling city, and each process as an individual citizen or organization, each with a specific role.

### **What Exactly is a Windows Process?**

A Windows process is essentially a running program or a part of the operating system carrying out its functions. Every single action on your computer – from a simple mouse click to a complex application launch is tied to a process. They each have their own dedicated memory space and resources, making them independent "workers" within the system.

### **The "ID Card" of a Process: P.P.C.U.P.**

Just like in any well-organized city, every "citizen" (process) has an ID card with crucial information. As a SOC analyst, you need to quickly assess this ID to determine if a process is legitimate or a potential threat. Remember the acronym **P.P.C.U.P.**:

* **P**ath: Where does this process "live" or execute from? (e.g., `C:\Windows\System32`)
* **P**arent: Who "hired" or launched this process? This is crucial for understanding its lineage.
* **C**ommand Line: What specific instructions or arguments was this process given?
* **U**ser: Under which user account is this process running? This dictates its privileges.
* **P**ID: Its unique, temporary identification number (Process ID).



    ![](/assets/images/img1.png)

### **Navigating the City: Key Standard Windows Processes**
Now, let's meet the most important "citizens" – the **Standard Windows Processes**. These are the core services and applications that make Windows run. Understanding their normal behavior is like knowing the regular routes and roles of essential city workers. Any deviation from this "normal" is a red flag!

Imagine the startup of a Windows computer as the **"Grand Opening of a Digital City"**:

#### **Phase 1: The City's Infrastructure (Kernel & Session Management)**

1. **System (PID 4): The Mayor's Office.**
* **Role:** This is the highest authority, responsible for all kernel-mode operations. It's always PID 4.
* **Normal Behavior:** Always running, no parent, runs as `SYSTEM`.


2. **smss.exe (Session Manager Subsystem): The City Planner.**
* **Role:** The first "user-mode" process, responsible for creating new user sessions (like opening new districts in the city).
* **Normal Behavior:** Spawns `csrss.exe` and then `wininit.exe` (for system services) or `winlogon.exe` (for user logins).



#### **Phase 2: Setting Up Essential City Services (Session 0)**

Once `smss.exe` has laid out the groundwork, it activates critical services for the city.

3. **csrss.exe (Client Server Runtime Subsystem): The Public Works Department.**
* **Role:** Manages graphical instructions, processes, and threads.
* **Normal Behavior:** One instance per session, runs as `SYSTEM`.


4. **wininit.exe (Windows Initialization): The City Hall Manager.**
* **Role:** Initializes crucial background services.
* **Normal Behavior:** Spawns `services.exe` and `lsass.exe`. Runs as `SYSTEM`.
* **services.exe (Service Control Manager): The HR Department.**
* **Role:** Manages all other Windows services and drivers (the "employees").
* **Normal Behavior:** Spawns `svchost.exe`. Runs as `SYSTEM`.
* **svchost.exe (Service Host): The Freelancers/Contractors.**
* **Role:** A generic host process that runs multiple services from DLLs.
* **Normal Behavior:** Many instances, each with a unique `-k` parameter to group similar services. Can run as `SYSTEM`, `LOCAL SERVICE`, or `NETWORK SERVICE`.




* **lsass.exe (Local Security Authority Subsystem Service): The City Security Department.**
* **Role:** Authenticates users, enforces security policies, and stores login credentials. **This is a prime target for attackers!**
* **Normal Behavior:** Runs as `SYSTEM`, spawned by `wininit.exe`. Only one instance.





#### **Phase 3: Opening Doors for Citizens (User Sessions)**

After the core services are up, the city is ready for its residents.

5. **winlogon.exe (Windows Logon): The Welcome Center.**
* **Role:** Handles interactive user logins and logouts.
* **Normal Behavior:** Spawns `LogonUI.exe` to display the login screen. Runs as `SYSTEM`.
* **LogonUI.exe (Logon User Interface): The Welcome Screen.**
* **Role:** Displays the login interface and collects user credentials.
* **Normal Behavior:** Spawns from `winlogon.exe`, passes credentials to `lsass.exe`. Runs as `SYSTEM`.




6. **explorer.exe (Windows Explorer): The Tourist Guide/City Map.**
* **Role:** Provides the user interface (desktop, taskbar, file manager). This is what most users interact with.
* **Normal Behavior:** Runs as the **logged-in user**, not `SYSTEM`. One instance per interactive user.



---

### **Spotting the Imposters: How Malware Hides**

In our bustling city, not everyone is a legitimate citizen. Malware often tries to blend in by mimicking standard processes. As a SOC analyst, you're the detective looking for the "imposters." Here's how they give themselves away:

* **Wrong Neighborhood (Process Path):** A core process like `lsass.exe` should *always* be in `%Systemroot%\System32`. If you see it in `C:\Users\Downloads` or `C:\Temp`, it's a huge red flag!
* **Suspicious Employer (Parent Process):** `services.exe` should be spawned by `wininit.exe`. If `cmd.exe` or a web browser spawned it, that's highly abnormal.
* **Fake ID (Misspelled Name):** Attackers might use `svch0st.exe` or `scvhost.exe` to trick you. Always double-check spelling.
* **Unusual Privileges (Username):** If a process that should run as `SYSTEM` (like `lsass.exe`) is running under a regular user account, investigate immediately.
* **Unfamiliar Instructions (Command Line):** Analyze the command-line arguments. Legitimate processes usually have predictable arguments; unusual or encoded strings can indicate malicious activity.

Here's an example of what to look for:
`

---

### **The Security Cameras: Windows Event Logs**
 ![](/assets/images/img2.png)

Even if an imposter slips past your initial observation, they leave traces. Windows Event Logs are your digital security cameras, recording every significant activity.

* **Event ID 4688 (Process Creation):** This iis your most valuable log for process monitoring. It records when a new process is created, often including its full command line, parent process, and user.
 ![](/assets/images/img3.png)

To dive deeper into the world of Windows process forensics, we need to look at the **technical nuances** that separate a junior analyst from a senior threat hunter.

---

## **1. Deep Dive: Event ID 4688 Fields**

When you open an Event 4688, these specific fields tell a story beyond just "a program started."

### **The Token Elevation Types (UAC Insights)**

Windows uses these codes to tell you if the process has the power to change the system:

- **%%1936 (Type 1):** Full Token. Used when UAC is off. The process has "God Mode."
    
- **%%1937 (Type 2):** Elevated Token. The user clicked "Run as Administrator." This is a high-value target for attackers.
    
- **%%1938 (Type 3):** Limited Token. Standard user rights. Most malware starts here and tries to "Privilege Escalate" to Type 2.
    

### **Mandatory Labels (Integrity Levels)**

This tells you how much Windows "trusts" the process.

- **System (S-1-16-16384):** High-level OS processes.
    
- **High (S-1-16-12288):** Administrator-level processes.
    
- **Medium (S-1-16-8192):** Standard user processes (like Chrome or Word).
    
- **Low (S-1-16-4096):** Highly restricted (like a browser sandbox). If you see a "Low" integrity process spawning a "High" one, you’ve found an exploit in progress.
    

---

## **2. Advanced Attack Techniques (The "Why")**

### **A. Living Off The Land (LOTL) - The "Invisible" Attack**

Attackers don't bring guns to a heist; they use the tools already inside the bank.

- **Rundll32.exe:** Used to run malicious code hidden inside a `.dll` file.
    
- **Wmic.exe / PowerShell:** Used to steal passwords or move to other computers on the network.
    
- **Certutil.exe:** A legitimate tool used to manage certificates, but attackers use it to download malware from the internet.
    

> **Verification Tip:** Use the [LOLBAS Project](https://lolbas-project.github.io/) to see if a command line looks like a known attack pattern.

**First Exemple :**

 ![](/assets/images/img4.png)
 ![](/assets/images/img5.png)
 ![](/assets/images/img6.png)
 

**Second Exemple :**

 ![](/assets/images/img7.png)
 ![](/assets/images/img8.png)
 
 ### **B. Process Injection : The Parasite 

This is the most dangerous technique. An attacker starts a legitimate process (like `explorer.exe`) and "injects" their code into its memory.

- **How to spot it:** Look for a process that starts behaving out of character. For example, if `svchost.exe` suddenly starts a network connection to an unknown IP in another country, it might be injected.

 ![](/assets/images/img9.png)
 
---

## **3. Mastering the Process Tree (The "Detective" Workflow)**

To catch a sophisticated attacker, you must learn to "climb" the process tree using **Hexadecimal Correlation**.

1. **Find the Suspect:** You see a weird process `malware.exe` (**PID: 0x1A4**).
    
2. **Find the Parent:** Look at the **Creator Process ID** in that log (e.g., **0x2B8**).
    
3. **Search for the Creator:** Look for a 4688 log where the **New Process ID** is **0x2B8**.
    
4. **Repeat:** Keep going up until you find the "Patient Zero" (the first process that started the mess, like a phishing email attachment).
    


## **Conclusion: The Analyst's Mindset**

Process investigation is all about **Context**. A `cmd.exe` window isn't scary if an IT Admin opened it. But if `winword.exe` (Microsoft Word) opens `cmd.exe`, which then opens `powershell.exe`, you are looking at a high-priority security incident..
