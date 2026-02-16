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

Let's connect devices for our SIEM.

### Connecting Ubuntu Linux

