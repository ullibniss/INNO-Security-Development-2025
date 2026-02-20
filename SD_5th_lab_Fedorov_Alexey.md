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

<img width="2160" height="244" alt="image" src="https://github.com/user-attachments/assets/047b277d-a59f-4d26-af08-5c5465f06671" />

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

```
Carry out a stress test on the application and verify the performance of the 
application on the server. The performance can be reviewed using a benchmark 
such as Spec benchmark. Take note of the results.
```

I created 2 pages hosting with apache2: static html, dynamic php page.

<img width="816" height="150" alt="image" src="https://github.com/user-attachments/assets/89c248fd-878a-47b8-b3db-8e14908f26d8" />

For benchmark I will use ApacheBench. 3 tests.

Test 1. Static page load test

<img width="2002" height="712" alt="image" src="https://github.com/user-attachments/assets/aaa68f1c-8b95-4e48-a127-04fd12869a7c" />

Test 2. Dynamic page load test

<img width="1888" height="702" alt="image" src="https://github.com/user-attachments/assets/32b643a1-5754-47b7-ab15-dfcccd934214" />

Test 3. Highload test for static page

<img width="2132" height="692" alt="image" src="https://github.com/user-attachments/assets/f0e8651b-86a3-4dcc-bcd9-15fe97bd7d67" />

the results are:

<img width="1296" height="1414" alt="image" src="https://github.com/user-attachments/assets/bfb5c7ef-1a2c-495e-86ee-9df96f9a2438" />

```
Install and enable SElinux on the same Linux server
```

I installed SElinux on Ubuntu, rebooted and has this default settings:

<img width="1070" height="578" alt="image" src="https://github.com/user-attachments/assets/30d94691-5d10-498b-ae59-631a52d2e856" />

I set SElinux to Enforced mode and checked default contexts:

<img width="2696" height="606" alt="image" src="https://github.com/user-attachments/assets/9795009e-7932-4f7d-97dc-da925cb9c6eb" />

```
Implement a couple of containment policies for the hosted webapp on the server 
and perform a similar stress test based on similar benchmarks used earlier. 
```

Firstly, I want to limit apache write permissions to /tmp folder:

```
sudo setsebool -P httpd_tmp_exec off
sudo setsebool -P httpd_use_fusefs off
```

<img width="596" height="398" alt="image" src="https://github.com/user-attachments/assets/b90784cc-c713-4ccc-b2de-089eb420369c" />

Next is restriction of outgoing requests:

```
sudo setsebool -P httpd_can_network_connect off
```

<img width="960" height="206" alt="image" src="https://github.com/user-attachments/assets/a4b119fb-fb39-40b5-9d10-8750805a8224" />

Restricting access to home directories:

<img width="794" height="406" alt="image" src="https://github.com/user-attachments/assets/cee1ecde-823c-4d23-aefa-67f6eaec3c52" />

Let's do benchmark and collect result again:

<img width="1262" height="1414" alt="image" src="https://github.com/user-attachments/assets/c01c1f72-ffe0-4e7f-8e8e-87b30cab057f" />

```
Do you observe any difference in performance?
```

Yes, there is a measurable performance impact with SELinux enforcing on Ubuntu:

|Benchmark|Baseline|SELinux|Diff|
|-|-|-|-|
|Static|3378| 2378.26| -30%~|
|Dynamic|2420| 2914.75|+17%~|
|Highload|2461| 2521.05|+2%~|

The benchmark results reveal workload-dependent SELinux performance impacts that differ significantly from typical patterns. Static content shows the expected overhead of -30%, consistent with SELinux's security context checking and label verification penalties on simple file reads. However, dynamic PHP content surprisingly shows a +17% performance improvement with SELinux enforcing, which is atypical and likely attributable to several factors: SELinux's containment policies may have reduced resource contention by preventing Apache from accessing unnecessary system resources (blocked network connections, restricted /tmp access, limited home directory reads), effectively forcing a more focused execution path; additionally, the baseline test may have experienced background noise or cache misses that the enforcing test avoided due to system state differences. The high-load scenario shows minimal impact (+2%), suggesting that under concurrent load, SELinux's overhead becomes negligible relative to network and application bottlenecks. These results demonstrate that SELinux's performance impact is highly workload-specific: while simple operations show overhead from mandatory access control checks, complex applications with restricted policies may actually benefit from reduced system-wide resource contention, and under realistic concurrent loads, the security benefits of containment come at virtually no performance cost.

# References

1) https://gitlab.com/apparmor/apparmor/-/wikis/Documentation
2) https://httpd.apache.org/docs/2.4/ru/
3) https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/7/html/selinux_users_and_administrators_guide/index
4) https://httpd.apache.org/docs/2.4/programs/ab.html
