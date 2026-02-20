# Lab 5: Mandatory Access Control

## Completed by Fedorov Alexey (tg: @ullibniss)

---

```
This lab is designed to introduce students to system hardening. The students will
interact with 2 fundamental technologies used in Linux security; AppArmor and SElinux
```

## PART A - AppArmor

### 1. Using the SIEM used in the previous lab, explain how are CIS benchmarks checked on an endpoint?

Wazuh SIEM uses the Security Configuration Assessment (SCA) module to automatically check CIS benchmarks on endpoints. SCA policies are stored in /var/ossec/ruleset/sca/ on the Wazuh agent in YAML format. Each policy contains multiple checks that validate specific CIS benchmark requirements. The agent runs these checks periodically (default: every 12 hours) and on startup. Results are sent to the Wazuh Manager and indexed in OpenSearch.

CIS Check Example:

```
- id: 1234
  title: "Ensure AppArmor is installed"
  description: "AppArmor provides Mandatory Access Control"
  rationale: "AppArmor protects the system by confining programs"
  remediation: "Run: apt install apparmor"
  compliance:
    - cis: "1.6.1.1"
  rules:
    - 'c:dpkg -s apparmor -> r:Status: install ok installed'
```
Two Operating Modes:
- Enforce mode: Actively blocks violations and logs denials
- Complain mode: Logs violations without blocking (for testing)

Ubuntu includes profiles for common services:
1) Web servers (Apache, Nginx)
2) Database servers
3) DNS servers
4) Email servers

These profiles are automatically activated when services are installed, providing immediate protection without manual configuration.

Let's use Wazuh to show CIS benchmark, it has the following checks:

<img width="2028" height="1650" alt="image" src="https://github.com/user-attachments/assets/f23bbdf8-bd37-4c37-8b08-86b03978f661" />

Wazuh is also has settings related to SCA in `ossec.conf`:

<img width="1526" height="286" alt="image" src="https://github.com/user-attachments/assets/daabaa57-1e3b-4597-b463-2fd0b88bbaaa" />

There is option `scan_on_start = yes`, so we can see logs on agent restart:

<img width="2160" height="244" alt="image" src="https://github.com/user-attachments/assets/1fe455ae-8936-4e99-90b9-6ab652ce75d7" />

Of course we can see results in Wazuh Dashboard:

<img width="3086" height="1688" alt="image" src="https://github.com/user-attachments/assets/71371bc8-f067-4071-afd0-d11e6663167a" />

We can also see, that AppArmor is not installed:

<img width="2288" height="840" alt="image" src="https://github.com/user-attachments/assets/c35b98e9-d817-4551-b76e-8ae2906ac43d" />

### 2. Based on a Linux distribution of your choice, fulfill the MAC section of the latest respective CIS benchmark. link: CIS benchmark download

I installed AppArmor and configured GRUB:

<img width="3102" height="388" alt="image" src="https://github.com/user-attachments/assets/39bd13da-01aa-4ea8-bf88-5fbfe93c2691" />

<img width="1390" height="718" alt="image" src="https://github.com/user-attachments/assets/bd5aeb81-cfb0-4ec0-b0a8-18f691b2f4bf" />

Let's make all profiles enforced:

<img width="1144" height="276" alt="image" src="https://github.com/user-attachments/assets/32488214-b678-43b3-8e8b-cc7cd4159ce1" />

As we can see, checks that related to `1.3 MAC of CIS Benchmark` are passed:

<img width="2260" height="534" alt="image" src="https://github.com/user-attachments/assets/ef4da5a0-36b5-4eb3-86a5-13a2c2d2b191" />

### 3. Configure a Webapp to serve static files from two directories and configure AppArmor to confine the Webapp to one of the two directories.

I will use apache2 as a web server, because i installed it in previous lab. So i created 2 directories:

<img width="654" height="116" alt="image" src="https://github.com/user-attachments/assets/90b25865-a4b4-4c56-a519-cbb46a52ab67" />

I configured apache2 to share two folders with the same handlers:

<img width="1054" height="790" alt="image" src="https://github.com/user-attachments/assets/a87b134d-07e3-454d-a4b4-49f8b9c9bcb9" />

By default, I can access them from outside:

<img width="2612" height="248" alt="image" src="https://github.com/user-attachments/assets/456a3a02-905f-4875-8a04-aec31e815837" />

Let's create apparmor profile for apache. I took ready template:

<img width="930" height="1462" alt="image" src="https://github.com/user-attachments/assets/aedab937-a8f5-4174-9e8d-985ebffadec7" />

I also loaded it in apparmor. Now i can see, that this profile in complain mode:

<img width="1496" height="340" alt="image" src="https://github.com/user-attachments/assets/b492757f-1f8d-45db-b8c0-6a835839b5ff" />

We can see that in complain mode it not restrict any files, but notifies about access to un expected files

<img width="2432" height="174" alt="image" src="https://github.com/user-attachments/assets/158821dd-bf9a-472d-b16d-2a2661cafe23" />

<img width="3082" height="530" alt="image" src="https://github.com/user-attachments/assets/57958ff5-d131-4add-8ed9-cb69c6a84ff0" />

I switched mode to enforce:

<img width="724" height="126" alt="image" src="https://github.com/user-attachments/assets/cbdfb433-2a0a-4c23-98dd-54c1c8a3db21" />

<img width="1652" height="542" alt="image" src="https://github.com/user-attachments/assets/55971c97-3354-4118-9f03-52c0f889f162" />

<img width="3044" height="512" alt="image" src="https://github.com/user-attachments/assets/dcd9ed9e-03bc-44d8-bed3-e82bbcf94415" />

As we can see, file was blocked by AppArmor.

### 4. Briefly explain how AppArmor uses default profiles to secure your services

AppArmor implements Mandatory Access Control (MAC) through application-specific profiles located in /etc/apparmor.d/. These profiles define exactly what system resources each application can access.

It controls file access:

```
/etc/apache2/** rwx, # File glob + permissions        
```

It controls capabilities:

```
capability net_bind_service
```

And it also use abstractions, that are reusable common rules:

```
#include <abstractions/base>
```

Ubuntu provides pre-configured profiles for common services. These activate automatically on installation, providing immediate protection without manual configuration. The principle is "default deny" - only explicitly permitted operations are allowed in enforce mode.

### 5. In a situation where your Webapp fails to start or misbehaving after the Apparmor profile has been enforced i.e AppArmor confinement, how would you rectify this? What steps would you take to troubleshoot this?

When a webapp fails after AppArmor enforcement, I follow this troubleshooting approach:

1. I try to verify that AppArmor is cause by checking enforce mode and logs

```
sudo aa-status | grep <profile>
sudo tail -50 /var/log/syslog | grep "apparmor.*DENIED"
```

2. If problem in AppArmor, trying to switch in complain mode:

```
sudo aa-complain /usr/sbin/apache2
```

If service starts to work, the common issue is in system files it tries to create. For example: logs, locks, pids.

3. Updating profile.
4. Turning back enforcing mode.
5. If it failed again, i start with step 1.

## PART B - SElinux

### 1. Give a short explanation on SElinux

SELinux (Security-Enhanced Linux) is a Mandatory Access Control (MAC) system implemented in the Linux kernel. Originally developed by the NSA and Red Hat, it provides fine-grained access control beyond traditional Unix permissions.

Key Concept:

1) Security Contexts (Labels). Every process, file, and resource has a security context with format:

```
user:role:type:level
```
Example: `system_u:object_r:httpd_sys_content_t:s0`

2) Type Enforcement (TE). Access decisions based on types. A process with type httpd_t can only access files with allowed types (e.g., httpd_sys_content_t).
3) Policies. Rules defining what types can interact. SELinux denies everything by default unless explicitly allowed by policy.
4) Operating Modes:
- Enforcing: Blocks violations
- Permissive: Logs violations without blocking
- Disabled: SELinux completely off

SELinux provides defense-in-depth by adding kernel-level MAC on top of traditional DAC, reducing attack surface.

### 2. Deploy a simple webapp or DB on a Linux server




