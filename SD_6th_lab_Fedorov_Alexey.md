# Lab 6: Penetration Testing

## Completed by Fedorov Alexey (tg: @ullibniss)

---

## BackStory

```
Below is a virtual machine (VM) from an incorporation that runs a critical service for the
organization. This service handles cryptocurrency transactions, manages blockchain security,
and stores condential client data. The company prides itself on its cutting-edge security
infrastructure, but recent security audits have raised concerns about potential vulnerabilities in
its web applications and backend services.
Unbeknownst to this Incorporation., miscongurations and an insecure web application have
introduced security loopholes into the system. A vulnerability in the web application allows
Remote Code Execution (RCE), potentially exposing sensitive data and allowing attackers to
gain unauthorized access to the system. Additionally, weak credentials and miscongured
services have created further openings for exploitation.
As a security researcher, your task is to assess the security of this incorporation server, identify
its vulnerabilities, and attempt to exploit them to retrieve the hidden security ag:
halborn{ag}.
```

## Objective

```
This lab aims to introduce students to remote Linux enumeration, vulnerability analysis, and
exploitation techniques. Students will explore a given Ubuntu virtual machine (VM) from a
separate attacking VM, identify open ports, discover vulnerabilities within the logic of a web
application, and gain access to the web application.
```

## Prerequisites

```
● Basic knowledge of the Linux command line
● Familiarity with networking concepts (e.g., SSH, ports, scanning)
● Understanding of common security vulnerabilities (e.g., miscongured services, weak
credentials, session hijack)
● Experience with penetration testing tools (e.g., Nmap, netdiscover, Burpsuite etc)


```
