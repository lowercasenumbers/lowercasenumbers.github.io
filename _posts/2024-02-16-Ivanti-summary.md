---
layout: post
title: Ivanti Vulnerability Updates
subtitle: It can get worse, and it will
cover-img: /assets/img/Ivanti.png
thumbnail-img: /assets/img/Ivanti_thumb.png
share-img: /assets/img/Ivanti.png
tags: [ivanti,vpn]
author: Mike Vedete
---
For cyber adversaries, Ivanti is the gift that keeps on giving. The latest vulnerabilities in Ivanti VPN devices have been actively exploited, leading to several critical security concerns. Here's a summary of the key vulnerabilities and related information.

# CVE Details
**CVE-2024-22024 (XXE):** This XML external entity injection (XXE) vulnerability affects Ivanti Connect Secure and Ivanti Policy Secure in specific versions, allowing cyber threat actors to potentially take control of an affected system. Ivanti released software updates on February 8, 2024, to address this and other vulnerabilities​. The vulnerable endpoint `/dana-na/auth/saml-sso.cgi` can be exploited my making a POST request to which includes a parameter called SAML Request with a malicious XML included. For easy testing, a [nuclei template](https://github.com/projectdiscovery/nuclei-templates/blob/main/http/cves/2024/CVE-2024-22024.yaml) exists which includes a sample payload. 

**CVE-2024-21888 (Privilege Escalation):** This flaw allows users to elevate their privileges to that of an administrator, posing a high security risk with a CVSS v3 base score of 8.8. The vulnerability is characterized by low attack complexity, requiring low privileges and no user interaction, and it has a high impact on the confidentiality, integrity, and availability of the affected systems​​​.


**CVE-2024-21893 (SSRF):** A server-side request forgery (SSRF) vulnerability could be exploited to take control of an affected system. This flaw enables attackers to access certain restricted resources without authentication. Updates and mitigation guidance was released by Ivanti to cover this vulnerabilities​​. CVE-2024-21893 also has a published [nuclei template](https://github.com/projectdiscovery/nuclei-templates/blob/main/http/cves/2024/CVE-2024-21893.yaml) which attacks the endpoint `/dana-ws/saml20.ws` and could allow some restricted resources to be accessed without authentication.S

**CVE-2023-46805: (Authentication Bypass):** This authentication bypass vulnerability found in the web component of Ivanti ICS versions 9.x, 22.x, and Ivanti Policy Secure. This vulnerability allows remote attackers to access restricted resources by bypassing control checks. Similar to the others in this list, CVE-2023-46805 has been exploited in the wild, with reports indicating that attacks have been attributed to nation-state actors. Attackers have deployed webshells and utilized compromised credentials for further network penetration and data exfiltration. 

**CVE-2024-21887 (Command Injection):** This command injection vulnerability allows an authenticated administrator to execute arbitrary commands on the appliance by sending specially crafted requests. This can be completed by sending a GET request such as: `/api/v1/totp/user-backup-code/../../license/keys-status/%3bcurl%20www.google.com`. For simplified tested a [nuclei template](https://github.com/projectdiscovery/nuclei-templates/blob/main/http/cves/2024/CVE-2024-21887.yaml) exists with additional details. 

# Mitigation Bypass Techniques
Threat actors have developed techniques to bypass initial mitigations provided by Ivanti, including deploying custom web shells, such as BUSHWALK, for further exploitation. These activities suggest targeted attacks aiming to maintain access to compromised systems​​.

# Exploitation Activities
Attackers have modified legitimate components of Ivanti Connect Secure to steal user credentials and deploy additional web shells, indicating a sophisticated level of compromise and highlighting the criticality of detecting and responding to such intrusions​​.

Emergency Directives: CISA has issued emergency directives and supplemental directions requiring federal agencies to disconnect affected Ivanti products and perform specific remediation steps due to the severity of these vulnerabilities and the ongoing exploitation​​.

Ivanti has responded to these vulnerabilities by releasing patches and updates. However, the discovery of new vulnerabilities even after patches have been applied indicates a challenging security landscape. Agencies and organizations using Ivanti VPN products are strongly encouraged to apply all available updates, monitor their systems for signs of compromise, and implement recommended security practices to mitigate these risks​.

Speaking to others closely tracking these developments, there seems to be a consensus that more CVEs are on the way. By design, VPN appliances provide remote access into otherwise (hopefully) inaccessible environments. This makes them a big target for adversaries. Many security professionals are taking decisive and painful actions to mitigate these issues. In one conversation someone said, "if there ever was a time impact end users, the time is now."

The situation underscores the importance of continuous vigilance and timely application of security patches in the face of evolving cyber threats. Organizations are also advised to review CISA's guidance and implement recommended actions to safeguard their networks​.
