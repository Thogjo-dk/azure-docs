---
title: What is risk? Azure AD Identity Protection
description: Explaining risk in Azure AD Identity Protection

services: active-directory
ms.service: active-directory
ms.subservice: identity-protection
ms.topic: conceptual
ms.date: 08/30/2021

ms.author: joflore
author: MicrosoftGuyJFlo
manager: karenhoran
ms.reviewer: sahandle

ms.collection: M365-identity-device-management
---
# What is risk?

Risk detections in Azure AD Identity Protection include any identified suspicious actions related to user accounts in the directory. Risk detections (both user and sign-in linked) contribute to the overall user risk score that is found in the Risky Users report.

Identity Protection provides organizations access to powerful resources to see and respond quickly to these suspicious actions. 

![Security overview showing risky users and sign-ins](./media/concept-identity-protection-risks/identity-protection-security-overview.png)

> [!NOTE]
> Identity Protection generates risk detections only when the correct credentials are used. If incorrect credentials are used on a sign-in, it does not represent risk of credential compromise.

## Risk types and detection

Risk can be detected at the **User** and **Sign-in** level and two types of detection or calculation **Real-time** and **Offline**.

Real-time detections may not show up in reporting for five to ten minutes. Offline detections may not show up in reporting for two to twenty-four hours.

### User-linked detections

Risky activity can be detected for a user that is not linked to a specific malicious sign-in but to the user itself. These risk detections are calculated offline using Microsoft's internal and external threat intelligence sources including security researchers, law enforcement professionals, security teams at Microsoft, and other trusted sources.

These risks are calculated offline using Microsoft's internal and external threat intelligence sources including security researchers, law enforcement professionals, security teams at Microsoft, and other trusted sources.

| Risk detection | Description |
| --- | --- |
| Leaked credentials | This risk detection type indicates that the user's valid credentials have been leaked. When cybercriminals compromise valid passwords of legitimate users, they often share those credentials. This sharing is typically done by posting publicly on the dark web, paste sites, or by trading and selling the credentials on the black market. When the Microsoft leaked credentials service acquires user credentials from the dark web, paste sites, or other sources, they are checked against Azure AD users' current valid credentials to find valid matches. For more information about leaked credentials, see [Common questions](#common-questions). |
| Azure AD threat intelligence | This risk detection type indicates user activity that is unusual for the given user or is consistent with known attack patterns based on Microsoft's internal and external threat intelligence sources. |

### Sign-in risk

A sign-in risk represents the probability that a given authentication request isn't authorized by the identity owner. 

These risks can be calculated in real-time or calculated offline using Microsoft's internal and external threat intelligence sources including security researchers, law enforcement professionals, security teams at Microsoft, and other trusted sources.

| Risk detection | Detection type | Description |
| --- | --- | --- |
| Anonymous IP address | Real-time | This risk detection type indicates sign-ins from an anonymous IP address (for example, Tor browser or anonymous VPN). These IP addresses are typically used by actors who want to hide their login telemetry (IP address, location, device, etc.) for potentially malicious intent. |
| Atypical travel | Offline | This risk detection type identifies two sign-ins originating from geographically distant locations, where at least one of the locations may also be atypical for the user, given past behavior. Among several other factors, this machine learning algorithm takes into account the time between the two sign-ins and the time it would have taken for the user to travel from the first location to the second, indicating that a different user is using the same credentials. <br><br> The algorithm ignores obvious "false positives" contributing to the impossible travel conditions, such as VPNs and locations regularly used by other users in the organization. The system has an initial learning period of the earliest of 14 days or 10 logins, during which it learns a new user's sign-in behavior. |
| Anomalous Token | Offline | This detection indicates that there are abnormal characteristics in the token such as an unusual token lifetime or a token played from an unfamiliar location. This detection covers Session Tokens and Refresh Tokens. |
| Token Issuer Anomaly | Offline |This risk detection indicates the SAML token issuer for the associated SAML token is potentially compromised. The claims included in the token are unusual or match known attacker patterns. |
| Malware linked IP address | Offline | This risk detection type indicates sign-ins from IP addresses infected with malware that is known to actively communicate with a bot server. This detection is determined by correlating IP addresses of the user's device against IP addresses that were in contact with a bot server while the bot server was active. |
| Suspicious browser | Offline | Suspicious browser detection indicates anomalous behavior based on suspicious sign-in activity across multiple tenants from different countries in the same browser. |
| Unfamiliar sign-in properties | Real-time | This risk detection type considers past sign-in history (IP, Latitude / Longitude and ASN) to look for anomalous sign-ins. The system stores information about previous locations used by a user, and considers these "familiar" locations. The risk detection is triggered when the sign-in occurs from a location that's not already in the list of familiar locations. Newly created users will be in "learning mode" for a period of time in which unfamiliar sign-in properties risk detections will be turned off while our algorithms learn the user's behavior. The learning mode duration is dynamic and depends on how much time it takes the algorithm to gather enough information about the user's sign-in patterns. The minimum duration is five days. A user can go back into learning mode after a long period of inactivity. The system also ignores sign-ins from familiar devices, and locations that are geographically close to a familiar location. <br><br> We also run this detection for basic authentication (or legacy protocols). Because these protocols do not have modern properties such as client ID, there is limited telemetry to reduce false positives. We recommend our customers to move to modern authentication. <br><br> Unfamiliar sign-in properties can be detected on both interactive and non-interactive sign-ins. When this detection is detected on non-interactive sign-ins it deserves increased scrutiny due to the risk of token replay attacks.  |
| Admin confirmed user compromised | Offline | This detection indicates an admin has selected 'Confirm user compromised' in the Risky users UI or using riskyUsers API. To see which admin has confirmed this user compromised, check the user's risk history (via UI or API). |
| Malicious IP address | Offline | This detection indicates sign-in from a malicious IP address. An IP address is considered malicious based on high failure rates because of invalid credentials received from the IP address or other IP reputation sources. |
| Suspicious inbox manipulation rules | Offline | This detection is discovered by [Microsoft Cloud App Security (MCAS)](/cloud-app-security/anomaly-detection-policy#suspicious-inbox-manipulation-rules). This detection profiles your environment and triggers alerts when suspicious rules that delete or move messages or folders are set on a user's inbox. This detection may indicate that the user's account is compromised, that messages are being intentionally hidden, and that the mailbox is being used to distribute spam or malware in your organization. |
| Password spray | Offline | A password spray attack is where multiple usernames are attacked using common passwords in a unified brute force manner to gain unauthorized access. This risk detection is triggered when a password spray attack has been performed. |
| Impossible travel | Offline | This detection is discovered by [Microsoft Cloud App Security (MCAS)](/cloud-app-security/anomaly-detection-policy#impossible-travel). This detection identifies two user activities (is a single or multiple sessions) originating from geographically distant locations within a time period shorter than the time it would have taken the user to travel from the first location to the second, indicating that a different user is using the same credentials. |
| New country | Offline | This detection is discovered by [Microsoft Cloud App Security (MCAS)](/cloud-app-security/anomaly-detection-policy#activity-from-infrequent-country). This detection considers past activity locations to determine new and infrequent locations. The anomaly detection engine stores information about previous locations used by users in the organization. |
| Activity from anonymous IP address | Offline | This detection is discovered by [Microsoft Cloud App Security (MCAS)](/cloud-app-security/anomaly-detection-policy#activity-from-anonymous-ip-addresses). This detection identifies that users were active from an IP address that has been identified as an anonymous proxy IP address. |
| Suspicious inbox forwarding | Offline | This detection is discovered by [Microsoft Cloud App Security (MCAS)](/cloud-app-security/anomaly-detection-policy#suspicious-inbox-forwarding). This detection looks for suspicious email forwarding rules, for example, if a user created an inbox rule that forwards a copy of all emails to an external address. |
| Azure AD threat intelligence | Offline | This risk detection type indicates sign-in activity that is unusual for the given user or is consistent with known attack patterns based on Microsoft's internal and external threat intelligence sources. |

### Other risk detections

| Risk detection | Detection type | Description |
| --- | --- | --- |
| Additional risk detected | Real-time or Offline | This detection indicates that one of the above premium detections was detected. Since the premium detections are visible only to Azure AD Premium P2 customers, they are titled "additional risk detected" for customers without Azure AD Premium P2 licenses. |

## Common questions

### Risk levels

Identity Protection categorizes risk into three tiers: low, medium, and high. When configuring [custom Identity protection policies](./concept-identity-protection-policies.md#custom-conditional-access-policy), you can also configure it to trigger upon **No risk** level. No Risk means there is no active indication that the user's identity has been compromised.

While Microsoft does not provide specific details about how risk is calculated, we will say that each level brings higher confidence that the user or sign-in is compromised. For example, something like one instance of unfamiliar sign-in properties for a user might not be as threatening as leaked credentials for another user.

### Password hash synchronization

Risk detections like leaked credentials require the presence of password hashes for detection to occur. For more information about password hash synchronization, see the article, [Implement password hash synchronization with Azure AD Connect sync](../hybrid/how-to-connect-password-hash-synchronization.md).

### Leaked credentials

#### Where does Microsoft find leaked credentials?

Microsoft finds leaked credentials in various places, including:

- Public paste sites such as pastebin.com and paste.ca where bad actors typically post such material. This location is most bad actors' first stop on their hunt to find stolen credentials.
- Law enforcement agencies.
- Other groups at Microsoft doing dark web research.

#### Why aren't I seeing any leaked credentials?

Leaked credentials are processed anytime Microsoft finds a new, publicly available batch. Because of the sensitive nature, the leaked credentials are deleted shortly after processing. Only new leaked credentials found after you enable password hash synchronization (PHS) will be processed against your tenant. Verifying against previously found credential pairs isn't done. 

#### I haven't seen any leaked credential risk events for quite some time?

If you haven't seen any leaked credential risk events, it's because of the following reasons:

- You don't have PHS enabled for your tenant.
- Microsoft hasn't found any leaked credential pairs that match your users.

#### How often does Microsoft process new credentials?

Credentials are processed immediately after they have been found, normally in multiple batches per day.

### Locations

Location in risk detections is determined by IP address lookup.

## Next steps

- [Policies available to mitigate risks](concept-identity-protection-policies.md)
- [Security overview](concept-identity-protection-security-overview.md)
