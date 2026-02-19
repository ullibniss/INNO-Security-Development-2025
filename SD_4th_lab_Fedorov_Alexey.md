# SD Lab 4: SIEM

## Completed by Fedorov Alexey (tg: @ullibniss)

---

```
This lab is designed to introduce students to security solutions, specifically a SIEM. In
this lab, students can use any SIEM of choice; regardless, a solid recommendation is to
use the open source security platform Wazuh as this provides a fleet of capabilities at
no cost.
In this lab, students will interact with additional tools such as virustotal, YARA, osquery,
SOAR and also gain experience with SIEM log analysis, vulnerability detection and
more.
```

# Task 1 - Introduction

## 1.a Give a brief explanation of the architecture of your SIEM solution.

I've chosen `Wazuh` SIEM, because it was recommended.

![image](https://github.com/user-attachments/assets/e0b5eba9-00b7-4b94-bc0c-a840927de5b7)

## 1.b Provide 3 advantages of open source solutions and how do these vendors actually make money?

## Task 2 - Setup infrastructure

## 2.a. Configure a SIEM solution with 3(or more) unique devices. e.g Windows, Linux and a Network device. Can you view log data from each connected device? If yes show this.

### Wazuh deployment

Firstly, I need to deploy `Wazuh`. I will use installation script from documentation

```
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh
sudo bash wazuh-install.sh -a
```

So, i installed Wazuh. Now i can open it in browser.

<img width="3104" height="1054" alt="image" src="https://github.com/user-attachments/assets/b7682156-77ab-453c-b4e1-69330e02dd6f" />

I also connected devices to WAZUH:

- Ubuntu Linux
- Windows
- Mikrotik RouterOS

I connected Ubuntu and Windows using Wazuh Agents:

<img width="3098" height="1152" alt="image" src="https://github.com/user-attachments/assets/35c165dd-b6fd-476d-a081-83b12fba5aab" />

<img width="3074" height="1710" alt="image" src="https://github.com/user-attachments/assets/94baba58-c674-4f7e-8e85-3747d405548f" />

But Mikrotik was connected via remote syslog:

<img width="2460" height="1198" alt="image" src="https://github.com/user-attachments/assets/09a39a0b-d8cf-4c46-985c-29e5a2b732eb" />

After configuring special decoder and rules for remote logs, i have a result:

<img width="3068" height="1366" alt="image" src="https://github.com/user-attachments/assets/2068a865-5482-4d20-ae9b-a27c28592a8c" />

## 2.b

You can view MikroTik router logs in Wazuh SIEM because we established a complete logging pipeline. For example, a warning log like "script,warning TEST MESSAGE" indicates a non-critical event from MikroTik's script subsystem, while an error log like "script,error CONNECTION FAILED" represents a more severe issue requiring attention. These logs are visible because MikroTik sends them via syslog protocol (UDP port 514) to the Wazuh Manager, where custom decoders parse the MikroTik-specific format (extracting fields like severity, program, and message), and custom rules (IDs 100201-100204) evaluate them based on severity levels, generating structured alerts stored in 

```
/var/ossec/logs/alerts/alerts.json.
```

The alerts become visible in the SIEM Dashboard through Wazuh's indexing pipeline: Filebeat continuously monitors the alerts.json file and ships new events to the Wazuh Indexer (OpenSearch/Elasticsearch), which stores them in time-based indices like `wazuh-alerts-4.x-2026.02.19`. The Wazuh Dashboard queries these indices to display events in real-time under Threat Hunting → Events, where you can filter, search, and analyze them. This architecture enables centralized visibility because it standardizes diverse log formats (MikroTik syslog → structured JSON), applies security rules to detect issues, and provides a searchable interface for security monitoring across your infrastructure—transforming raw network device logs into actionable security intelligence.
