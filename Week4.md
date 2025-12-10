# Week 4 – Initial System Configuration & Security Implementation

## 1. SSH Key-Based Authentication

In Week 4, I replaced password-based SSH logins with key-based authentication, as required.

On the **workstation** I generated an ED25519 key pair:

```bash
ssh-keygen -t ed25519 -C "ashish@windows"
```

```bash
Get-Content $env:USERPROFILE\.ssh\id_ed25519.pub
```
<img width="946" height="53" alt="image" src="https://github.com/user-attachments/assets/c4e2b1d8-a0ea-4229-a44a-ea5c046699e2" />

I then connected to the server via SSH and installed the public key into ~/.ssh/authorized_keys for the ashish account:

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
nano ~/.ssh/authorized_keys   
```
<img width="1200" height="120" alt="image" src="https://github.com/user-attachments/assets/8b89a806-e29a-4490-955b-43029b843a5f" />

```bash
chmod 600 ~/.ssh/authorized_keys
ls -ld ~/.ssh
ls -l ~/.ssh
```
<img width="821" height="190" alt="image" src="https://github.com/user-attachments/assets/2cd64218-ff2c-4e38-a3d8-da6e3bedd36d" />

Finally, I opened a new SSH session from the workstation and confirmed that authentication succeeded using the key:
```bash
ssh -i ~/.ssh/id_ed25519 -v ashish@192.168.1.171
```
<img width="1127" height="840" alt="image" src="https://github.com/user-attachments/assets/733121ad-49c8-44ff-b9c5-357625b40ea1" />

## 2. Firewall Configuration
To restrict remote access, I configured the Ubuntu firewall (`ufw`) to allow SSH only from my Windows workstation IP address, satisfying the requirement to control access via firewall rules.

First, I identified the workstation address using `ipconfig`:

# Windows workstation
```bash
ipconfig
```
<img width="1015" height="801" alt="image" src="https://github.com/user-attachments/assets/01afe159-0de8-4736-89d8-6c1365171dff" />

On the server (over SSH), I then enabled and configured `ufw`:
```bash
sudo apt install -y ufw
sudo ufw status
```
<img width="555" height="48" alt="image" src="https://github.com/user-attachments/assets/d9a87afb-671f-44b3-9478-05cfa36e7757" />

```bash
sudo ufw default deny incoming
```
```bash
sudo ufw default allow outgoing
```
<img width="711" height="146" alt="image" src="https://github.com/user-attachments/assets/527f6b8f-ac7b-4925-8ec8-57097e597f9e" />

```bash
sudo ufw allow from 192.168.1.100 to any port 22 proto tcp
```
<img width="1023" height="50" alt="image" src="https://github.com/user-attachments/assets/993b5862-67aa-4180-aa3e-1111477da0e7" />

```bash
sudo ufw enable
```
<img width="1045" height="117" alt="image" src="https://github.com/user-attachments/assets/436bd77d-06c1-4754-ba2d-bbe39cc2fe3c" />

```bash
sudo ufw status verbose
```
<img width="826" height="221" alt="image" src="https://github.com/user-attachments/assets/14718564-afcc-43a9-b45d-88acab8a6c6c" />

```bash
sudo ufw status numbered
```
<img width="734" height="169" alt="image" src="https://github.com/user-attachments/assets/2ddecb4a-34ed-4734-9c1a-7c2fd2bf7294" />

This ruleset implements a default-deny policy for inbound traffic while permitting SSH exclusively from the trusted workstation IP.


## 3. User and Privilege Management
The non-root administrative account `ashish` was created during installation and is used for all remote administration. In Week 4 I verified its privileges and ensured that direct root logins are disabled.

```bash
id ashish
groups ashish
getent passwd ashish
sudo -l
```
<img width="1684" height="290" alt="image" src="https://github.com/user-attachments/assets/ec2b8979-ffa3-4c34-9cda-00963abbe9f1" />

The ID and groups output shows that Ashish is a regular user (UID ≠ 0) and a member of the sudo group. sudo -l confirms the account can run administrative commands with password-based sudo escalation. Combined with PermitRootLogin no in sshd_config, this ensures that all administration occurs via a non-root user with controlled privilege escalation.

## 4. SSH Access Evidence
The screenshot below demonstrates successful key-based SSH access from the Windows workstation to the Ubuntu server after hardening:

- Authentication uses the ED25519 key.
- No password prompt is required.
- The command prompt clearly shows `ashish@osserver`.


## 5. Configuration File Before/After Comparisons
Before editing `sshd_config`, I created a backup:
```bash
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.preweek4
```

After applying the changes, I compared the files:
```bash
sudo diff -u /etc/ssh/sshd_config.preweek4 /etc/ssh/sshd_config
```

#### Key changes include:
- PermitRootLogin set to no to disable root SSH logins.
- PasswordAuthentication and ChallengeResponseAuthentication are set to no to enforce key-based authentication.
- AllowUsers ashish added to restrict SSH access to a single administrative user.


## 6. Firewall Documentation
The final `ufw` ruleset is shown below:

```bash
sudo ufw status verbose
sudo ufw status numbered
```
<img width="843" height="386" alt="image" src="https://github.com/user-attachments/assets/06ec71dd-fd2d-4a74-b332-4a5a8fe63255" />

This confirms:
- Default incoming policy: deny
- Default outgoing policy: allow
- Inbound rule: allow SSH (port 22) only from 192.168.1.100 (workstation IP)
- No other inbound ports are open at this stage.

## 7. Remote Administration Evidence
All configuration changes in this phase were performed via SSH from the workstation, in line with the administrative constraint. 

Example remote management commands:
```bash
sudo adduser testuser
id testuser
hostnamectl
```
<img width="997" height="463" alt="image" src="https://github.com/user-attachments/assets/00b43230-8e5f-4aca-bbd8-e8fb79965fa3" />

```bash
sudo usermod -aG sudo testuser
```
<img width="1001" height="418" alt="image" src="https://github.com/user-attachments/assets/450a5327-49a4-4a37-91cd-f3aa7ab8df9f" />

```bash
sudo systemctl status ssh
```
<img width="1901" height="873" alt="image" src="https://github.com/user-attachments/assets/1f4c104e-5335-4ad8-b52b-7d2bf23e5dc4" />


```bash
sudo ufw status
```
<img width="752" height="152" alt="image" src="https://github.com/user-attachments/assets/d8fe2abb-3b10-400e-aa45-dfc3f2e30ca6" />

```bash
journalctl -u ssh --since "today" | head
```
<img width="1485" height="269" alt="image" src="https://github.com/user-attachments/assets/8b5dde64-261f-4e3c-bdc9-da55be05c238" />

These commands demonstrate that I can inspect services, firewall status, and SSH logs without using the VirtualBox console, which is reserved only for recovery.

## 8. Reflection
In​‍​‌‍​‍‌ the fourth week, it was the first time I altered the server's security posture to a great extent. After setting up key-based authentication and subsequently turning off password logins, I had to take extra caution in not getting myself locked out; hence, I kept one SSH session open while I tested another. The computer felt like a real production host, whose access is very tightly controlled, when I configured `ufw` to permit SSH only from my workstation. Checking user privileges and turning off root logins also gave the least privileged principle more strength. These modifications are the basis for the more sophisticated controls that I will be able to install in Week 5, for instance, automatic updates, AppArmor profiles, and ​‍​‌‍​‍‌fail2ban.
