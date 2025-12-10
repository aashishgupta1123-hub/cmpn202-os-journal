# Week 2 – Security Planning and Testing Methodology

## 1. Performance Testing Plan
### 1.1 Objectives

My performance testing aims to understand how the Ubuntu server behaves under different workloads and how security controls affect resource usage. I plan to measure:

- CPU usage
- Memory usage
- Disk I/O performance
- Network performance
- Latency/response times for at least one server application

```bash
sudo apt install -y sysstat iotop iftop iperf3
```
<img width="957" height="1201" alt="image" src="https://github.com/user-attachments/assets/f9b3d784-f627-4adb-8fca-c2e2d5d7e3f3" />

### 1.2 Metrics and Tools
I will use the following tools to collect metrics:

| Metric             | Tool / Command                 | Notes |
|--------------------|-------------------------------|-------|
| CPU utilisation    | `top`, `htop`, `vmstat`       | Live and sampled CPU usage and run queue length |
| Memory usage       | `free -h`, `vmstat`           | Total/used memory and swap activity |
| Disk I/O           | `iostat -xz`, `iotop`         | Per-device throughput and per-process I/O wait |
| Network throughput | `iftop`, `iperf3`, `ss`       | Bandwidth and connection details |
| Filesystem usage   | `df -h`                       | Space usage for root and data partitions |

### 1.3 Remote Monitoring Methodology

All monitoring will be performed **remotely from the Windows workstation** using SSH. Instead of logging in interactively on the server console, I will run commands from the workstation, for example:

```bash
ssh ashish@192.168.1.171 "vmstat 1 5"
```
<img width="1069" height="353" alt="image" src="https://github.com/user-attachments/assets/1d4ad172-ae0c-4682-9e8c-bbc78b470559" />

```bash
ssh ashish@192.168.1.171 "iostat -xz 1 5"
```
<img width="1836" height="1000" alt="image" src="https://github.com/user-attachments/assets/e7acd6e7-5616-4865-871f-300d2b86a58b" />

### 1.4 Planned Testing Scenarios

For each application chosen in Week 3, I will run:
- Baseline test: Light or idle usage to capture normal system behaviour.
- Load test: Higher concurrency or data volume to stress the system.
- Optimisation test: Re-run after tuning configuration and compare metrics.

Each test run will have:
- A clear name.
- Recorded start and end times.
- A corresponding monitoring log captured from the workstation via SSH.

```bash
$ top
```
<img width="1387" height="984" alt="image" src="https://github.com/user-attachments/assets/320d73aa-27be-43d9-8ac7-5982567c4938" />

```bash
$ htop
```
<img width="2559" height="1439" alt="image" src="https://github.com/user-attachments/assets/a98b161d-d808-45dd-9749-943e335a07a5" />

## 2. Security Configuration Checklist
The table below describes the security controls I plan to implement in Phases 4 and 5. It focuses on SSH hardening, firewall configuration, mandatory access control, automatic updates, user privilege management, and network security.

| Area                        | Planned Control                                                                 | Evidence Week |
|-----------------------------|----------------------------------------------------------------------------------|--------------|
| SSH hardening               | Enable key-based authentication, disable password authentication, disable root login, and change the default SSH configuration to minimise the attack surface. | Week 4       |
| Firewall configuration      | Configure `ufw` (or equivalent) to allow SSH only from the workstation IP and restrict all other inbound traffic by default. | Week 4       |
| Mandatory access control    | Use AppArmor (Ubuntu default) to enforce profiles for key services (e.g. SSH, web server), verifying profiles are loaded and in enforcing mode. | Week 5       |
| Automatic updates           | Configure unattended security updates so that critical security patches are applied automatically, with logs collected as evidence. | Week 5       |
| User privilege management   | Use a non-root administrative user with `sudo` access; ensure direct root login is disabled and sudo usage is audited. | Week 4       |
| Network security/intrusion detection | Deploy `fail2ban` to monitor SSH logs and automatically block repeated login failures. Review ban logs as evidence. | Week 5       |

In later weeks, I will convert this checklist into concrete configuration steps and verify each control using the `security-baseline.sh` script and supporting CLI commands.

## 3. Threat Model
This threat model focuses on realistic risks to my small VirtualBox-based lab environment. Although the system is not internet-facing, the same threats apply to production servers and help justify the security controls planned for later phases.

| Threat ID | Threat Description | Asset(s) at Risk | Likely Attack Vector | Impact | Planned Mitigation |
|----------|--------------------|------------------|----------------------|--------|--------------------|
| T1 | Brute-force SSH attacks from the local network or wider internet. | SSH service, user accounts, server configuration. | Automated tools repeatedly guessing usernames/passwords on port 22. | Compromise of admin account, full control of server. | Enforce key-based authentication, disable password logins, disable root login, configure the firewall to allow SSH only from the workstation IP, and deploy `fail2ban` to block repeated failures. |
| T2 | Compromised workstation used as a pivot to attack the server. | Server OS, services, monitoring and baseline scripts. | Malware or an attacker on the Windows host reusing saved SSH keys or sessions. | Unauthorised commands run on the server, data corruption, and misconfiguration. | Protect SSH keys with proper file permissions, avoid storing plain-text passwords, limit SSH access to a single user account with the least privilege, and monitor logs for unusual activity. |
| T3 | Misconfiguration exposing additional services or ports. | Network-facing services (e.g. web server, database). | Enabling test services or leaving default configurations open to the network. | Information disclosure or remote code execution via poorly configured services. | Use a firewall default-deny policy, explicitly allow only required ports, and periodically scan the server from the workstation with `nmap` in Week 7 to verify open ports. |
| T4 | Unpatched vulnerabilities in OS or packages. | Kernel, daemons, third-party applications. | Exploiting known CVEs in outdated software. | Privilege escalation or remote exploitation. | Configure automatic security updates (`unattended-upgrades`), run `apt update && apt upgrade` regularly, and track package versions in the journal. |

## 4. Reflection
Over​‍​‌‍​‍‌ the past week, I transitioned from just setting up the server to actually detailing the security and performance planning. In fact, preparing the performance testing plan made me really consider which metrics are of actual importance (CPU, memory, disk, and network) and the way to get them from a distance via SSH without interacting with the server console. Working out the security checklist and threat model also gave me an insight into how the different measures mingled: securing SSH, firewall rules, AppArmor, automatic updates, and fail2ban were all steps that targeted specific risks respectively, instead of being random configuration steps. This planning should bring more organization to the subsequent implementation weeks and lessen the likelihood of missing out on important ​‍​‌‍​‍‌controls.
