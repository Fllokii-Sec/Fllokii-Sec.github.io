---
layout: page
title: Web Application Firewall
description: Simulated attacks on DVWA, the configured WAF to detect and block the attacks.
img: assets/img/3.jpg
importance: 2
category: Home Lab
giscus_comments: true
---

# Web Application Firewall Lab

## Objective

This project demonstrates the deployment, securing, and testing of a vulnerable web application within an isolated environment. The objective was to host the Damn Vulnerable Web Application (DVWA), implement proactive defense mechanisms using the SafeLine Web Application Firewall (WAF), secure data-in-transit via SSL/TLS, and validate the security posture by executing and analyzing web-based attacks.

### Skills Learned

- Systems Administration: Linux (Kali Linux/Ubuntu OS), Docker, & Network Configuration
- Application Security: WAF configuration, reverse proxy setup, and traffic routing
- Ability to generate and recognize attack signatures and patterns.
- Cryptography: OpenSSL, self-signed certificate generation, and SSL/TLS termination
- Penetration Testing: SQL Injection (SQLi) and Reflected Cross-Site Scripting (XSS)

### Tools Used

- VMware for isolated Lab enviroment using virtual machines.
- Security Tools: SafeLine WAF, OpenSSL
- Vulnerable Software: DVWA (Damn Vulnerable Web Application)

## Steps


Step 1: Target Environment Setup (DVWA) 
Deployed an Ubuntu Virtual Machine to act as the hosting environment and Kali Linux as the attacking machine. Installed, configured, and initialized the Damn Vulnerable Web Application (DVWA) within a containerized environment to serve as the target application for security testing.
 - I Changed apache2 port conf file to listen on port 8080 instead 80 and also changed 000-default.conf file to port 8080 and added server name webserver.fllokiisec.
 - On both Kali Linux and Ubuntu, I updated the /etc/hosts file and added Ubuntu ip to the domain webserver.fllokiisec, so I could reach the DVWA through the domain www.webserver.fllokii instead of ip.
 - I then made sure the MariaDB was up and there was users and passwords in the database.
 - DVWA (script) sudo bash -c "$(curl --fail --show-error --silent --location https://raw.githubusercontent.com/IamCarron/DVWA-Script/main/Install-DVWA.sh)"

<img width="1804" height="686" alt="vmware_eyW22TqnAx" src="https://github.com/user-attachments/assets/ed55c0f9-a992-456f-84e0-5f0ad04054de" />


Step 2: WAF Integration (SafeLine) & SSL/TLS Hardening & Traffic Redirection
Downloaded and deployed the SafeLine Web Application Firewall (WAF) to intercept and analyze incoming HTTP/HTTPS traffic.
Configured SafeLine as a reverse proxy, establishing a protected routing path to the underlying DVWA container.
 - Safeline (script) bash -c "$(curl -fsSLk https://waf.chaitin.com/release/latest/manager.sh)" -- --en
 - I created a new directory /etc/ssl/dvwa where I would creat the private key and self-signed SSL/TLD certificate.
 - I used OpenSSl to generate a custom private key and self-signed certificate.
 - Once I had the private key and private certificate, I Added DVWA as an application on safeline and bound the self-signed certificate in Safeline.
 - Configured upstream routing and traffic redirection rules on the WAF to ensure all external access to DVWA is strictly forced over a secure channel on Port 443.
   
<img width="486" height="216" alt="vmware_rhsmjGZ5E1" src="https://github.com/user-attachments/assets/a197fd5d-fdd5-45ef-b7c6-c6ab3fb617bc" />

<img width="1796" height="761" alt="vmware_ovKdFwpR14" src="https://github.com/user-attachments/assets/24a4352f-cf94-4d5f-b42a-c65a71d22995" />


Step 4: Security Validation & Exploitation Testing
Conducted simulated attacks against the deployment to verify application vulnerabilities and observe traffic behavior.
 - On the Kali Linux virtual machine, I simulated multiple requests and was blocked by the rate limitiing rule that I set up.

<img width="881" height="252" alt="vmware_GOHNL1Tu5z" src="https://github.com/user-attachments/assets/834c3715-a8b6-42fb-b5d2-d4949d4ede18" />

<img width="1549" height="144" alt="vmware_YmyWWHiz8S" src="https://github.com/user-attachments/assets/0cb4faae-15aa-4148-bfc4-243dd9428276" />

<img width="1920" height="1032" alt="vmware_MWL5MVZfSS" src="https://github.com/user-attachments/assets/1a256eec-1372-42f2-9587-bc4b75344623" />

 - Simulated SQL Injection attack and extrated users names and passwords of users in the the database, After I security parameters up in fafeline, the attack was blocked.

<img width="870" height="478" alt="vmware_ZImrl5F3yn" src="https://github.com/user-attachments/assets/bc74d781-144b-4386-a2b9-a74a4d31da63" />

<img width="1085" height="729" alt="vmware_EiBGDjogsE" src="https://github.com/user-attachments/assets/39530782-14b9-4d90-95ce-efb6787c0fda" />

<img width="1920" height="1032" alt="vmware_NxwqOmyaQF" src="https://github.com/user-attachments/assets/bda45bb5-ceb0-465c-b7c1-d24584446ee0" />

 - Simulated reflected XXS and I got an alert on screen, and after setting security parameters up in Safeline, Attack was bloacked.

<img width="868" height="427" alt="vmware_iSc41onHnj" src="https://github.com/user-attachments/assets/ed174cb9-db22-481d-afef-26c0bf25ab35" />

<img width="1920" height="1032" alt="vmware_7ygRE2JEw8" src="https://github.com/user-attachments/assets/83343103-0cdd-42bc-86d6-d14d8f1efb17" />


 - Once attacks were simulated, I then set up an authorization page, to make you authenicate yourself before accessing web page.

<img width="988" height="528" alt="vmware_16zQMD7pRF" src="https://github.com/user-attachments/assets/a9e3c62a-c3bd-41e0-96ab-d55f0ccc479f" />

<img width="1302" height="701" alt="vmware_bcfrqKyRdS" src="https://github.com/user-attachments/assets/3aa52c58-4557-4e62-9ea7-b8cb903f74c6" />

<img width="1824" height="768" alt="vmware_K19lDcbj6T" src="https://github.com/user-attachments/assets/e1341639-edc0-4436-a799-a31c05f2732b" />


Through this project, I gained practical experience in building a defense-in-depth security architecture by combining infrastructure hardening, transport security, and perimeter defense. Specifically, I mastered:
 - Secure Reverse Proxy Architecture: How to securely intercept, inspect, and route public traffic to internal, isolated application containers.
 - Public Key Infrastructure (PKI) Basics: Generating cryptographic keys and implementing self-signed certificates via OpenSSL to secure data-in-transit.
 - Attack Surface Analysis: The mechanics of common web exploits like SQL Injection (SQLi) and Reflected XSS, and how they behave from both an attacker's perspective and an administrator's defensive posture.
 
Deploying the SafeLine WAF in front of a vulnerable application like DVWA underscored exactly why Web Application Firewalls are a critical standard in modern enterprise security:
 - Shielding Vulnerable Legacies: Codebases often contain legacy vulnerabilities or zero-day flaws that cannot be instantly patched. A WAF acts as an immediate virtual patch, blocking malicious payloads before they ever      reach the underlying application code.
 - Layer 7 Deep Packet Inspection: Traditional firewalls filter traffic based on IP addresses and ports. A WAF inspects the actual content of HTTP/HTTPS requests, distinguishing legitimate user input from malicious           injection attacks (OWASP Top 10).
 - Centralized Traffic & SSL Management: By terminating SSL/TLS at the WAF level, organizations can offload resource-heavy cryptographic processes from the web servers, enforce global HTTPS redirection rules, and monitor     all ingress security alerts from a single dashboard.
