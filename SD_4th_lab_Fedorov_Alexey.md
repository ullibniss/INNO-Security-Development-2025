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

# Task 2 - Setup infrastructure

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

## 2.b Why specifically are you able to view these logs i.e select two visible logs, explain these logs, and explain why and how you are able to view it on the SIEM

You can view MikroTik router logs in Wazuh SIEM because we established a complete logging pipeline. For example, a warning log like "script,warning TEST MESSAGE" indicates a non-critical event from MikroTik's script subsystem, while an error log like "script,error CONNECTION FAILED" represents a more severe issue requiring attention. These logs are visible because MikroTik sends them via syslog protocol (UDP port 514) to the Wazuh Manager, where custom decoders parse the MikroTik-specific format (extracting fields like severity, program, and message), and custom rules (IDs 100201-100204) evaluate them based on severity levels, generating structured alerts stored in 

```
/var/ossec/logs/alerts/alerts.json.
```

The alerts become visible in the SIEM Dashboard through Wazuh's indexing pipeline: Filebeat continuously monitors the alerts.json file and ships new events to the Wazuh Indexer (OpenSearch/Elasticsearch), which stores them in time-based indices like `wazuh-alerts-4.x-2026.02.19`. The Wazuh Dashboard queries these indices to display events in real-time under Threat Hunting → Events, where you can filter, search, and analyze them. This architecture enables centralized visibility because it standardizes diverse log formats (MikroTik syslog → structured JSON), applies security rules to detect issues, and provides a searchable interface for security monitoring across your infrastructure—transforming raw network device logs into actionable security intelligence.

# Task 3. Use cases 

The cases I selected:

```
b. Simulate a brute force attack against your infrastructure and demonstrate how you 
would detect the attack on each of the devices within your infrastructure. Are you 
able to detect the attack? If not, ensure you are able to.
c. Demonstrate how you would use the SIEM to detect existing CVEs within devices in 
your infrastructure. i.e vulnerability detection. Ensure you remediate at least 1 
vulnerability on each device and prove this in an updated scan
```

## b. Bruteforce

### Ubuntu Linux 24.04

I implemented script for bruteforce

<img width="1930" height="342" alt="image" src="https://github.com/user-attachments/assets/96d76d50-40ff-46a2-90d5-b2c0bf5cd1d3" />

Let's execute:

<img width="942" height="570" alt="image" src="https://github.com/user-attachments/assets/5ee23630-07fc-448c-9570-2574c7b91080" />

Bruteforce was detected:

<img width="3088" height="1660" alt="image" src="https://github.com/user-attachments/assets/ddf19e25-f4ee-49c6-b930-cdb4f6feb807" />

### Windows 10

Unfortunately, I installed home version of Windows, because of this there no RDP. I used SAMBA instead. I turned it on and configured user:

<img width="884" height="142" alt="image" src="https://github.com/user-attachments/assets/79528adc-333c-4d2c-ae5a-767a82dbe81e" />

I also implemented script for bruteforce^

<img width="1158" height="282" alt="image" src="https://github.com/user-attachments/assets/a8d6d0a5-462f-420e-a209-8bdd2a3121a0" />

Let's execute:

<img width="1066" height="446" alt="image" src="https://github.com/user-attachments/assets/0e33e7c3-57c6-46e7-9dd8-b710a49c1d67" />

As we can see, Wazuh rules detected bruteforce and windows locked account:

<img width="3114" height="1708" alt="image" src="https://github.com/user-attachments/assets/7ba38e0b-e0a6-45a7-a166-26be5cd810a8" />

## c. CVEs detection

I started with setting up vulnerability detector

<img width="1136" height="942" alt="image" src="https://github.com/user-attachments/assets/c321bb95-5c6f-4315-8339-2c0c6ad57419" />

After manager reboot, scan is started:

<img width="2110" height="132" alt="image" src="https://github.com/user-attachments/assets/7a81d653-cc1b-4f79-a58f-770925c9e665" />

Wazuh made scan. As we can see, there are a lot of vilnerabilities:

<img width="3118" height="1686" alt="image" src="https://github.com/user-attachments/assets/fe1a12c8-12ac-486b-be7d-4f3e03aa55e8" />

The reason is I downloaded stock image and forgot to run `apt upgrade`. Let's check curl vulnerabilities:

<img width="1280" height="280" alt="image" src="https://github.com/user-attachments/assets/934d8949-f1cc-4630-b10a-c606ea442a89" />

I think it is possible to fix them with `apt upgrade curl`. And it worked for me:

<img width="3116" height="1128" alt="image" src="https://github.com/user-attachments/assets/9a9e3ec7-33b2-480c-84fe-e461cd61b5b5" />

<img width="3106" height="724" alt="image" src="https://github.com/user-attachments/assets/5979b285-4822-4014-88bd-dcecca86f254" />

# Part B

I've chosen task 5.c

```
c. Integrate the SIEM with a WAF and simulate a scenario to test the integration. e.g 
use of Modsecurity
```

## 5.c Integrate the SIEM with a WAF and simulate a scenario to test the integration. e.g use of Modsecurity

I downloaded apache2 and modsecurity 

```
sudo apt install apache2 libapache2-mod-security2 -y
```

Then, I turned modsecurity on. I used recommened configuration, except the SecRuleEngine mode:

```
sudo cp /etc/modsecurity/modsecurity.conf-recommended /etc/modsecurity/modsecurity.conf
sudo sed -i 's/SecRuleEngine DetectionOnly/SecRuleEngine On/' /etc/modsecurity/modsecurity.conf
```

Next step is to configure Wazuh Agent, to provide it apache logs access:

<img width="1072" height="364" alt="image" src="https://github.com/user-attachments/assets/024c0fc2-f314-425b-8f15-842ca8912fe5" />

Finally, I can make an attack. I've written simple script for attack:

<img width="1072" height="730" alt="image" src="https://github.com/user-attachments/assets/0af82802-b030-4c24-82a1-1988de97fc4e" />

<img width="1640" height="1426" alt="image" src="https://github.com/user-attachments/assets/063a8747-60d7-4946-a2ae-567b1ca75610" />

After attack we can see logs in Wazuh:

<img width="3106" height="1860" alt="image" src="https://github.com/user-attachments/assets/51f3339e-a1d5-470e-bc7f-81315859b714" />

Finally, modsecurity works properly. It detected attack and blocked the following requests.

# References

1) https://documentation.wazuh.com/current/installation-guide/index.html
2) https://wazuh.com/blog/monitoring-network-devices/
3) https://learn.microsoft.com/ru-ru/windows-server/storage/file-server/troubleshoot/detect-enable-and-disable-smbv1-v2-v3?tabs=server
4) https://documentation.wazuh.com/current/user-manual/capabilities/vulnerability-detection/how-it-works.html
5) https://medium.com/@alexxmacenas/wazuhs-rules-and-decoders-with-modsecurity-waf-5fb8f5aaa6a4
