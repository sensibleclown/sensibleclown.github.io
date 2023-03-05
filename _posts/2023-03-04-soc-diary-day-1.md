---
title: SOC Diary - Day 1 - Assessing Organization Current Security Profile
date: 2023-03-04 08:00:00 +/-TTTT
categories: [security, blue-team]
tags: [soc] # TAG names should always be lowercase
---

In order to evaluate an organization's cyber security profile and uncover any gaps or weaknesses, it is necessary to review the measures implemented to protect against cyber attacks and data breaches.

To accomplish this, We first identify the organization's assets, including servers, workstations, databases, networks, and applications, as well as the protective technologies and controls that have been implemented to secure those assets.

As an example, we have identified the following organization's assets :

- 20 servers,
- 500 workstations (PC/Laptops),
- 2 box of firewall devices
- 2 box of IPS devices
- 5 applications,
- 25 network segments and
- 10 rules/policies enforced by the Network Access Control devices

## Protection Capabilities - Layered Protection

Once this information has been gathered, it is possible to create a comprehensive overview of any gaps or weaknesses within the organization's defenses. This can be accomplished by mapping the protective technologies and controls to each protection layer in the Defense in Depth model, as demonstrated in the example table below.

| SECURITY DEVICES / CONTROLS                           | TOTAL | APPLIED | PERCENTS (AVG) |
| :---------------------------------------------------- | :---- | ------: | -------------: |
| **A. PERIMETER SECURITY**                             |       |         |    **70,75 %** |
| **A.1. Firewall**                                     |       |         |       **75 %** |
| A.1.1. Perform monthly audit on firewall rules        | 2     |       1 |           50 % |
| A.1.2. Update firewall with the latest patch          | 2     |       2 |          100 % |
| **A.2. IPS**                                          |       |         |     **66,5 %** |
| A.2.1. Perform monthly audit on IPS rules             | 2     |       2 |          100 % |
| A.2.2. Implement blocking mode                        | 2     |       1 |           50 % |
| A.2.3. Update IPS with the latest patch               | 2     |       1 |           50 % |
| **B. NETWORK SECURITY**                               |       |         |     **83,5 %** |
| **B.1. Network Access Control**                       |       |         |       **67 %** |
| B.1.1. Perform monthly audit on NAC rules/policy      | 10    |       7 |           70 % |
| B.1.2. Apply posturing policy on each network segment | 25    |      16 |           64 % |
| **B.2. Separate Network Using Segmentation**          | 1     |       1 |      **100 %** |
| **C. ENDPOINT SECURITY**                              |       |         |    **64,26 %** |
| **C.1. Antivirus**                                    |       |         |     **55,4 %** |
| C.1.1. Virus definition update                        | 500   |     354 |         70,8 % |
| C.1.2. Host scanned                                   | 500   |     200 |           40 % |
| **C.2. EDR**                                          |       |         |     **82,4 %** |
| C.2.1. EDR Successfully Deployed to endpoints         | 500   |     412 |         82,4 % |
| **C.3. OS Patching**                                  |       |         |       **55 %** |
| C.3.1. PC/Laptop with latest update                   | 500   |     275 |           55 % |
| C.3.2. Server with latest update                      | 20    |      11 |           55 % |
| **D. APPLICATION SECURITY**                           |       |         |       **40 %** |
| **D.1. Web application firewall**                     |       |         |       **60 %** |
| D.1.1. WAF with blocking mode enabled for each apps   | 5     |       3 |           60 % |
| **D.2. MFA**                                          |       |         |       **20 %** |
| D.2.1. Application with MFA enabled                   | 5     |       1 |           20 % |
| **E. DATA SECURITY**                                  |       |         |     **97,7 %** |
| **E.1. Data Loss Prevention**                         |       |         |     **97,7 %** |
| E.1.1. Deployed DLP to endpoints                      | 500   |     489 |         97,8 % |
| E.1.2. Endpoints with applied USB-block policy        | 500   |     500 |          100 % |
| E.1.3. Endpoints with applied watermark policy        | 500   |     477 |         95,4 % |

Using the data in the table, it is possible to generate a graphical representation of the organization's protection capabilities.

![protection capabilities](assets/img/2023-03-03/layered-protection.png)
_Protection Capabilities - Layered Protection_

The graphics presented above clearly highlight the organization's strengths and weaknesses in terms of protection capabilities. Specifically, the organization appears to have a strong protections in the area of data security, while there is room for improvement in the area of application security.

## Protection Capabilities - Mitre Att&ck Phases

One alternative method for evaluating protection capability is to utilize the Mitre Att&ck Framework, which involves mapping protective measures and controls against the various tactics and techniques that are visible in the framework.

![protection capabilities - Mitre Att&ck Phases](assets/img/2023-03-03/attack-phases.png)
_Protection Capabilities - Mitre Att&ck Phases_

Mitre Att&ck provide very comprehensive of detailed controls for every phases of an attack. To achieve this I have compiled all of these controls into the following [Excel](assets/img/2023-03-03/attack-phases.png) file.

## Related Posts

1. [SOC Diary - Day 0 - Initiation](https://sensibleclown.com/posts/soc-diary-day-0/)
