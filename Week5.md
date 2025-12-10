# Week 5 – Advanced Security and Monitoring Infrastructure

## 1. Access Control with AppArmor
Ubuntu 24.04 uses AppArmor as its mandatory access control system. In Week 5, I verified that it was enabled and placed the SSH and web server processes under enforced profiles.

```bash
sudo aa-status
```
<img width="862" height="1341" alt="image" src="https://github.com/user-attachments/assets/fb086a59-be09-4515-8733-8cf9c51815c8" />

```bash
sudo aa-enforce /etc/apparmor.d/usr.sbin.rsyslogd
```
<img width="698" height="45" alt="image" src="https://github.com/user-attachments/assets/802957df-b416-43d1-94da-06a5deeaf9e1" />

```bash
sudo aa-status
```
<img width="914" height="575" alt="image" src="https://github.com/user-attachments/assets/f1c95997-0189-45e4-9729-234b1bc19256" />

To track and report on access control status, I used aa-status to capture a snapshot of loaded profiles and journalctl to inspect kernel messages relating to AppArmor:
```bash
sudo aa-status > apparmor-status-week5.txt
sudo journalctl -k | grep -i apparmor | tail -n 10
```
<img width="1903" height="327" alt="image" src="https://github.com/user-attachments/assets/1f44e2c5-83c1-4bf2-a840-b63c2df18e18" />
This evidence shows that access to `sshd` and `nginx` is now governed by AppArmor profiles in enforce mode and that I can report on their state using standard tools.

## 2. Automatic Security Updates
To ensure the server receives ongoing security patches without manual intervention, I configured `unattended-upgrades`.

```bash
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure unattended-upgrades
cat /etc/apt/apt.conf.d/20auto-upgrades
systemctl status unattended-upgrades.service
systemctl list-timers '*apt*'
```
<img width="1349" height="594" alt="image" src="https://github.com/user-attachments/assets/b126a5ba-e474-479d-8e79-0a94c601f824" />

The 20auto-upgrades file shows that daily package list updates and unattended upgrades are enabled. The systemctl output confirms that the unattended-upgrades service is running. I verified activity by inspecting the log file:
```bash
sudo tail -n 20 /var/log/unattended-upgrades/unattended-upgrades.log
```
<img width="1898" height="154" alt="image" src="https://github.com/user-attachments/assets/45589eb5-0398-4221-87b5-2a91378d43fe" />

## 3. Fail2ban Intrusion Detection
I deployed `fail2ban` to monitor SSH login attempts and automatically ban IP addresses that repeatedly fail authentication.

```bash
sudo apt install -y fail2ban
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```
<img width="869" height="133" alt="image" src="https://github.com/user-attachments/assets/bf95e253-78fc-4511-a4bb-ed5cf9ec5f84" />

```bash
sudo nano /etc/fail2ban/jail.local   
sudo systemctl restart fail2ban
sudo systemctl status fail2ban
sudo fail2ban-client status
sudo fail2ban-client status sshd
```
<img width="1919" height="423" alt="Screenshot 2025-12-09 093038" src="https://github.com/user-attachments/assets/3e5d93f2-16b7-4a65-a8a9-d829c4519d29" />

Demonstrating a ban
```bash
ssh wronguser@192.168.1.171   
ssh ashish@192.168.1.171  
```
<img width="1182" height="869" alt="image" src="https://github.com/user-attachments/assets/5c09f18e-48e7-47c4-bfe4-23530a1d8c75" />

After generating several failed SSH attempts from the workstation, I confirmed that the sshd jail registered the attempts and, when thresholds were exceeded, added the IP to the banned list.

## 4. Security Baseline Verification Script (`security-baseline.sh`)
To verify that all security controls from Phases 4 and 5 remain correctly configured, I wrote a `security-baseline.sh` script which runs on the server and prints a series of PASS/WARN checks.

Key checks include:

- SSH configuration (`PermitRootLogin no`, `PasswordAuthentication no`, `PubkeyAuthentication yes`, `AllowUsers ashish`)
- Firewall configuration (`ufw` active, default deny inbound, SSH allowed only from the workstation IP)
- AppArmor enabled with profiles for `sshd` and `nginx`
- `unattended-upgrades` service active and periodic configuration set
- `fail2ban` service active and `sshd` jail configured

The script is stored in my home directory and executed via SSH from the workstation:

```bash
ssh ashish@192.168.1.171 "./security-baseline.sh"
```
<img width="1789" height="673" alt="image" src="https://github.com/user-attachments/assets/61653637-ecfc-4bdb-b936-2b2e6a60e4a3" />

Every code block in the script is documented with comments explaining what each check does and why it matters for the security baseline.

## 5. Remote Monitoring Script (`monitor-server.sh`)
For performance monitoring, I created a `monitor-server.sh` script that runs on my workstation using Ubuntu WSL. The script:

1. Connects to the server via SSH using my existing key.
2. Runs `vmstat`, `iostat`, `free -h` and `df -h` for a fixed duration.
3. Stores the results in timestamped log files under a local `server-metrics` directory.

Example execution from WSL:

```bash
./monitor-server.sh
```
<img width="1111" height="707" alt="image" src="https://github.com/user-attachments/assets/f5e72999-1e6d-48ae-bc1a-43dcae939d4e" />

Listing the files in server-metrics.
```bash
head "$HOME/server-metrics"/*_vmstat.txt
```
<img width="931" height="244" alt="image" src="https://github.com/user-attachments/assets/ee81a798-9677-4b4d-a00c-0f457dad97c1" />

The script is fully commented, explaining each variable (server IP, username, key path, duration, interval) and each SSH command used to collect metrics. These logs will be used in Week 6 to build graphs and analyse system behaviour under different workloads.

## 6. Reflection

I​‍​‌‍​‍‌ spent this week picking up from essential hardening to continuous protection and monitoring. With the help of AppArmor and fail2ban, I got a deeper understanding of how the kernel and user-space tools can be combined to limit the harm of a compromised service and a brute force attack. The creation of `security-baseline.sh` made me turn my Week 4 and 5 configurations into a set of checks that can be repeated, thus regression testing will be made easier if I change the settings later. Moreover, the `monitor-server.sh` script is, therefore, the instrument that gathers performance data from the workstation, which will be very important when I start real workloads in Week ​‍​‌‍​‍‌6.
