# Week 1 – System Planning and Distribution Selection

## 1. System Architecture
The lab environment uses a dual-system architecture as required in the coursework brief. A single physical Windows 11 host runs Oracle VirtualBox. In VirtualBox, I deployed a single Ubuntu Server 24.04.3 LTS virtual machine that serves as the headless server. All administration is performed on the Windows host via SSH, treating Windows as the workstation. 
                                               
#### Architecture Diagram 
<img width="981" height="1319" alt="week1-architecture" src="https://github.com/user-attachments/assets/d5118c4e-7707-440d-bc68-ad0dfcd15b49" />

## 2. Distribution Selection Justification

I chose **Ubuntu Server 24.04.3 LTS** as the server operating system. The reasons are:

- **Long-term support:** LTS releases receive security updates for several years, which aligns with the coursework requirement to implement security best practices on a stable platform.
- **Package ecosystem:** Ubuntu uses the `apt` package manager with a very large repository of packages (e.g. OpenSSH, fail2ban, monitoring tools), which simplifies installing everything needed for later phases.
- **Documentation and community:** Ubuntu has extensive official documentation and community guides, which reduces time spent troubleshooting basic configuration issues.

Alternative distributions considered:

- **Debian:** very stable but slightly older packages; Ubuntu builds on Debian and offers more up-to-date defaults.
- **CentOS / Rocky Linux:** good for enterprise environments, but use RPM/YUM tooling I am less familiar with. For this coursework, familiarity and speed of learning are more important than matching a specific company distribution.

Overall, Ubuntu Server balances stability, security updates, and ease of use, making it a suitable choice for this project.

## 3. Workstation Configuration Decision

For the workstation, I chose **Option B – using the host machine with an SSH client**. The Windows 11 host runs the OpenSSH client via Windows Terminal / PowerShell. This satisfies the requirement for a separate workstation system while keeping resource usage low, because only one VM needs to run at a time.

Advantages of Option B:

- Lower CPU and RAM usage on the host compared with running both a server VM and a desktop VM.
- Simpler workflow: I can keep documentation, browser, and SSH terminal on the same Windows desktop.
- Matches real-world scenarios where administrators connect from a laptop to a remote server over SSH.

This workstation connects to the Ubuntu server exclusively using SSH for all administration tasks, as demonstrated in the Week 1 CLI evidence.

## 4. Network Configuration Documentation
The Ubuntu Server VM uses two virtual network adapters:

- **Adapter 1 – Bridged Adapter (MediaTek Wi-Fi card)**  
  - IP address: `192.168.1.171/24` on interface `enp0s3`  
  - Obtains an address from the home router, placing the VM on the same LAN as the Windows host.  
  - Used for SSH access and internet connectivity.

- **Adapter 2 – Host-only Adapter (VirtualBox Host-Only Ethernet Adapter 2)**  
  - Currently unused but reserved for future isolated testing if required.

- **VirtualBox network summary:**

```text
Adapter 1: Intel PRO/1000 MT Desktop (Bridged Adapter, MediaTek Wi-Fi 6 MT7921)
Adapter 2: Intel PRO/1000 MT Desktop (Host-only Adapter, 'VirtualBox Host-Only Ethernet Adapter 2')
```
<img width="570" height="726" alt="image" src="https://github.com/user-attachments/assets/d66c7908-628c-4912-b6e3-1be696e16bbd" />


```text
$ ip addr
```
<img width="847" height="334" alt="image" src="https://github.com/user-attachments/assets/4549f1a2-4f17-44a2-9240-03080ab3e104" />

```text
ssh ashish@192.168.1.171
```
<img width="723" height="674" alt="image" src="https://github.com/user-attachments/assets/b3863d35-8966-42bf-bbd7-281b5597c8ee" />


## 5. Command-Line System Specification Evidence
All commands in this section were executed from the Windows workstation using SSH, demonstrating remote administration of the headless server.

### 5.1 Kernel and architecture – `uname -a`
```bash
$ uname -a
```
<img width="1496" height="54" alt="image" src="https://github.com/user-attachments/assets/259b1ee8-701f-475a-adec-858c71723b35" />

### 5.2 Memory usage – `free -h`
```bash
$ free -h
```
<img width="999" height="93" alt="image" src="https://github.com/user-attachments/assets/9204f9e4-0404-47fb-898a-f7e7d720fc06" />

### 5.3 Disk usage – `df -h`
```bash
$ df -h
```
<img width="920" height="189" alt="image" src="https://github.com/user-attachments/assets/cab899ff-5bcb-49dc-aaec-2a3dfc879f06" />

### 5.4 Network interfaces – `ip addr`
```bash
$ ip addr
```
<img width="1253" height="441" alt="image" src="https://github.com/user-attachments/assets/85824dc6-8095-4691-a1dd-17ee3b4900d9" />

### 5.5 OS release information – `/etc/os-release`
```bash
$ cat /etc/os-release
```
<img width="1043" height="330" alt="image" src="https://github.com/user-attachments/assets/78e30ac1-02b0-4add-952a-959971f2129e" />

### 5.6 Distribution information – `lsb_release -a`
```bash
$ lsb_release -a
```
<img width="788" height="326" alt="image" src="https://github.com/user-attachments/assets/cda6cfd8-ecbc-4a8e-b59c-000fef2420e7" />


## 6. Reflection
Throughout​‍​‌‍​‍‌ this week, my main focus was on implementing the fundamental dual-system architecture, along with getting comfortable with using SSH. The toughest part was setting up the network so that the Windows host could connect to the Ubuntu server in a stable way; changing VirtualBox to bridged mode and looking at `ip addr` made me realize how virtual NICs are linked to the actual hardware. Additionally, I had to figure out the cause of `apt` repository errors, which helped me understand the way Ubuntu package sources are set up. In fact, the first week has provided me with a strong foundation of VM setup, remote access, and basic system inspection commands, which I will be able to extend in the following ​‍​‌‍​‍‌stages.

> Note: To​‍​‌‍​‍‌ plan and structure this journal, I utilized the help of an AI assistant. I was in full control of the execution of all commands and the verification of the same, and I also looked over and made changes to the entire ​‍​‌‍​‍‌explanations.
