---
layout: page
title: project 5
description: a project with a background image
img: assets/img/1.jpg
importance: 3
category: fun
---

| **Day 8** | The Brochure | PDF Generation Server-Side Request Forgery (SSRF) | `Burp Suite`, Web Browser | Medium |
| **Day 9** | CryptoCabana | Azure Blob SAS Token & Key Vault Historical Secret Versioning | `Azure CLI`, Browser DevTools | Medium |



---

## Day 8: # Towel on the Sunbed

* **Category:** Web Application Security
* **Vulnerability Type:** Race Condition (Limit Overrun / Concurrency Control Flaw)
* **Tools Used:** 
  * Web Browser
  * Burp Suite Professional / Community (Repeater Parallel Grouping)

---

## 🔍 Walkthrough & Exploitation

### Step 1: Account Registration & Initial Observation

1. Navigate to the web application target IP in your browser.
2. Click **Register** to create a new user account.
3. Log in with the newly created credentials to access the user dashboard.

---

### Step 2: Analyzing Application Logic

1. Upon logging in, locate the **Claim Reward** function.
2. Click **Claim Reward** once. You will successfully receive your initial allocation of "ponzi" points.
3. Attempting to click **Claim Reward** a second time triggers a 24-hour timer restriction.
4. Check the **Vault** requirements — unlocking it requires **150 ponzi**.

<img width="731" height="347" alt="vmware_jULDv07l5V" src="https://github.com/user-attachments/assets/54b559ff-74dd-4eb4-aaf1-34c2f7d8e4e8" />

---

### Step 3: Vulnerability Analysis

1. Intercept the `POST /claim` (or relevant reward claim endpoint) request in **Burp Suite** and send it to **Repeater**.
2. Resending the request individually yields an error indicating that the timer is actively blocking the request.
3. **Flaw:** The server validates the cooldown state, updates the points, and updates the timestamp without enforcing mutex/locking locks or atomic database operations. This opens a window for a **Single-Packet / Parallel Race Condition Attack**.

<img width="365" height="98" alt="vmware_Xuvkactdtf" src="https://github.com/user-attachments/assets/c6c1a96c-ee0c-4eb1-b4b5-77c59839cab0" />

---

### Step 4: Exploiting the Race Condition

To ensure clean state execution:

1. Register a fresh account (or clear state) and obtain a brand-new session cookie (`connect.sid`).
2. Intercept a fresh reward request in Burp Suite and capture the `connect.sid` value.
3. In **Burp Repeater**:
   * Create a **New Request Group**.
   * Add the captured reward claim request into the group.
   * Duplicate the request tab **~99 times** inside the same group.
   * Update all duplicated tabs with the fresh `connect.sid` session token.
4. Set the execution mode on the group tab dropdown to:
   * **`Send group in parallel (last-byte sync)`**
5. Click **Send Group**.

<img width="779" height="537" alt="vmware_4yI2G3fCBj" src="https://github.com/user-attachments/assets/1b3a1231-f4ee-402c-a8aa-718f97ee4d5e" />

---

### Step 5: Post-Exploitation & Flag Retrieval

1. Observe the response codes across all tabs in the group — the parallel execution forces all requests through the validation window prior to the cooldown timestamp update.
2. Return to the browser and refresh the application dashboard.
3. The total accumulated balance will exceed **150 ponzi**.
4. Navigate to the **Vault**, click **Open Vault**, and retrieve the flag!

<img width="409" height="136" alt="vmware_Z8isODACLG" src="https://github.com/user-attachments/assets/4f1d944d-bbe1-424b-ae85-e2ddb9808a5d" />

<img width="726" height="246" alt="vmware_Vis5CE7E4g" src="https://github.com/user-attachments/assets/02842d47-0ee2-472b-8b8d-e77943e988a6" />

### Remediation & Mitigations
To patch this race condition:
Atomic Operations & Mutexes: Enforce strict session/user locking or handle balance/cooldown operations inside an atomic database transaction.
Database Level Constraints: Implement unique transaction constraints or row-level locking (SELECT ... FOR UPDATE) before checking or modifying the timestamp/balance.
Rate Limiting & Queueing: Queue incoming reward claims per user ID sequentially rather than handling them asynchronously across multiple concurrent worker threads.
---

## Day 9: CryptoCabana
* **Category:** Cloud Security / Azure Storage & Key Vault
* **Target System:** Crypto Kiosk Azure Cloud Infrastructure

### Step-by-Step Execution
### Step 1: Initial Reconnaissance & Client-Side Source Inspection

1. Navigate to the target web application hosted on Azure Blob Static Storage:
   [https://cryptocabanaf5scjagc.z13.web.core.windows.net/](https://cryptocabanaf5scjagc.z13.web.core.windows.net/)

After inspecting the source code I see, app.js and Inspected that further.
Key Finding: The client-side code contains hardcoded credentials or a Shared Access Signature (SAS) token / connection string allowing direct API calls to Azure Storage blobs.

<img width="1211" height="64" alt="vmware_TClvlz5aUs" src="https://github.com/user-attachments/assets/94fed7b3-ae77-4b6a-bf6d-63169486825c" />

   
2. **Azure Storage Enumeration & Artifact Download:**
   With SAS key that has sp=rl rights im all to list all containers.
   
   
   * Download configuration backups containing Azure Service Principal credentials (`App ID`, `Password`, `Tenant ID`).

4. **Service Principal Authentication:**
   * Log into Azure CLI using Service Principal details:
   * 
  curl -s "https://cryptocabanaf5scjagc.blob.core.windows.net/?comp=list&sv=2022-11-02&ss=b&srt=sco&sp=rl&se=2099-12-31T23:59:59Z&st=2024-01-01T00:00:00Z&spr=https&sig=ZAo05W8KXdSLM9afYCNGogNRV2N5a6aB4dQI3LXz%2Fh0%3D"

and see $web, backup, and vault. Which is where i started to investigate next

<img width="924" height="698" alt="vmware_8IEUySkT6P" src="https://github.com/user-attachments/assets/99a80f9e-d289-4545-886a-0d28c0b745d8" />

and see $web, backup, and vault. Which is where i started to investigate next

curl -s "https://cryptocabanaf5scjagc.blob.core.windows.net/vault?restype=container&comp=list&sv=2022-11-02&ss=b&srt=sco&sp=rl&se=2099-12-31T23:59:59Z&st=2024-01-01T00:00:00Z&spr=https&sig=ZAo05W8KXdSLM9afYCNGogNRV2N5a6aB4dQI3LXz%2Fh0%3D"

<img width="617" height="512" alt="vmware_omBGSmKyWu" src="https://github.com/user-attachments/assets/860ad6e8-a171-4f25-bcb6-13c33ad3b48e" />

And I revelead two files backup-service-account.json and seed_phrase.txt (decoy).
Continuing with investigation.

curl -s "https://cryptocabanaf5scjagc.blob.core.windows.net/vault/backup-service-account.json?sv=2022-11-02&ss=b&srt=sco&sp=rl&se=2099-12-31T23:59:59Z&st=2024-01-01T00:00:00Z&spr=https&sig=ZAo05W8KXdSLM9afYCNGogNRV2N5a6aB4dQI3LXz%2Fh0%3D"

<img width="722" height="146" alt="vmware_cSXafPfrQF" src="https://github.com/user-attachments/assets/a5972a58-9a1c-4212-be8b-86f7238854a1" />

I find service principle keys and the exact place I need to look.

### Service Principle Authenication

I then authenicate as service principle and try grab the master keys. 

<img width="508" height="347" alt="vmware_oOW5G4Fm2y" src="https://github.com/user-attachments/assets/b92ace0c-95b1-465b-b6e3-ee00cb87f978" />

It gave me forbidden so i decided to see what I could read instead.

az keyvault secret list --vault-name ccabana-kv-f5scjagc --output table

and I reveal three key shards and an expired master key so I try to see if I can read those.

<img width="1074" height="349" alt="vmware_REORBczBeY" src="https://github.com/user-attachments/assets/d16ed8bd-7583-46b7-9214-f07f60ef82ba" />

I was able to read key shar 1 and 3 but 2 was rotated after IT flagged it. 

After some research I learned Azure Key Vault never overwrites secrets. Every update automatically generates a new version while preserving prior iterations. Anyone with get permissions for that secret can still access older, unpurged versions, meaning rotating a secret does not wipe its history. So decided to try and list it.

az keyvault secret list-versions --vault-name ccabana-kv-f5scjagc --name key-shard-2 -o json

and I found two but one is created 2 min later the the older one. (the one I want).

az keyvault secret show \
  --id "https://ccabana-kv-f5scjagc.vault.azure.net/secrets/key-shard-2/3d6492d2c6f74123bc754a9ded22b2a0" \
  --query value -o tsv

<img width="903" height="542" alt="vmware_TGo8X3v2Wy" src="https://github.com/user-attachments/assets/5f0e30bf-befa-4558-9051-9edd4bb3a8dc" />

I was able to read it and reveal the last part of the flag.


---

## Day 10 The Hollow Shell










---

## Day 11 Inifinity Pool









---

Day 12 After Hours


### Step 1: File Inspection & Initial Strings Analysis

1. Download and extract the challenge task files (`attachments-*.zip`).
2. Inspect the extracted files: `INDEX.BTR`, `MAPPING1.MAP`, `MAPPING2.MAP`, `MAPPING3.MAP`, `OBJECTS.DATA`. These are structural components of a Windows **WMI Repository**.
4. Search across all raw data files for standard PowerShell execution signatures:

    ```bash strings * | grep -i "powershell" ```
   




---

## Day 13 The Guestbook














---

## Day 14: Management Wants A Word

## Defensive Remediation & Best Practices Summary

| Vulnerability / Flaw | Impacted Day | Remediation & Defensive Control |
|---|---|---|
| **Prompt Injection** | Day 1 | Enforce strict System Prompts, enforce strict user privilege boundaries independent of LLM context, and deploy guardrail validation filters. |
| **Exposed `.git` Folder** | Day 2 | Restrict web server access to dotfiles/directories (`.git`, `.env`) via web server rules (e.g., Nginx `location ~ /\.git`). |
| **OSINT / MD5 Information Leak** | Day 3 | Avoid using plain MD5 email hashes for user profile identification; use random UUIDs or salted hashes. |
| **Unencrypted C2 / Keylogging** | Day 4 | Deploy Endpoint Detection & Response (EDR) agents to detect unauthorized keystroke hooks (`pynput`), and inspect egress traffic for anomaly patterns. |
| **Unsafe PyYAML Deserialization** | Day 5 | Always use `yaml.safe_load()` instead of `yaml.load()` in Python applications. |
| **Over-permissioned AWS Cognito** | Day 6 | Enforce Least Privilege IAM policies on Cognito Unauthenticated roles; restrict DynamoDB `Scan` permissions. |
| **NoSQL Injection & SSTI** | Day 7 | Sanitize inputs using strict typing/schema validation (e.g., Zod/Joi), and render templates securely using auto-escaping without direct system execution functions. |
| **PDF Converter SSRF** | Day 8 | Restrict PDF engines from fetching local resources (`127.0.0.1`, `file://`), run converters in isolated sandbox containers, and enforce strict URL whitelisting. |
| **Exposed SAS Tokens & Key Vault Secrets** | Day 9 | Implement Azure Key Vault automated purge/expiration rules upon rotation, and never embed full-access SAS tokens in frontend code (use backend delegation instead). |
