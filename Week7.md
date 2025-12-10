# Week 7 – Security Audit and System Evaluation

## 1. Introduction and Scope
This week, I carried out a final security audit of the CMPN202 Ubuntu server environment. The aim was to verify the effectiveness of the security measures implemented in previous phases (SSH hardening, firewall, AppArmor, automatic updates, fail2ban and monitoring) and to identify any remaining risks. The audit tasks included a host security scan with Lynis, a network security assessment with Nmap, access control verification, a service inventory and overall configuration review.

## 2. Infrastructure Security Assessment
The system consists of:

- A Windows 11 workstation acting as the only trusted administrative client.
- A VirtualBox Ubuntu Server VM connected via a bridged network adapter.
- SSH as the sole remote administration method, secured with key-based authentication.
- A host-based firewall (`ufw`) configured to allow SSH (and HTTP where required) only from the workstation IP.
- Mandatory access control using AppArmor and intrusion detection with fail2ban.

These measures reduce the external attack surface to a minimal set of network-facing services with additional layers of protection from firewalls and access control frameworks.


## 3. Lynis Security Audit (Before and After Remediation)

I used Lynis to perform a host-based security audit of the Ubuntu server.

### 3.1 Baseline Lynis Scan

```bash
sudo lynis audit system | tee lynis-week7-before.txt
```
<img width="993" height="1339" alt="Screenshot 2025-12-09 134144" src="https://github.com/user-attachments/assets/17a7d4e1-aed0-490a-b811-7e0878563d78" />

The initial hardening index reported by Lynis was `60`. Key findings included:

- Install a PAM module for password strength testing like pam_cracklib or pam_passwdqc [AUTH-9262]
- When possible set expire dates for all password-protected accounts [AUTH-9282]
- Configure password hashing rounds in /etc/login.defs [AUTH-9230]

The initial Lynis scan reported a hardening index of **60/100** and several suggestions. One example was control **AUTH-9328**, which recommended using a stricter default umask in `/etc/login.defs` (027 instead of 022) so that new files are not world-readable.

I implemented this change by updating:

- Before: `UMASK 022`
- After:  `UMASK 027`
<img width="684" height="94" alt="image" src="https://github.com/user-attachments/assets/c397152a-e680-40b5-b72a-369efaab5b88" />

and verified it with:

```bash
grep -i '^UMASK' /etc/login.defs
sudo lynis show details AUTH-9328
```
<img width="1230" height="451" alt="image" src="https://github.com/user-attachments/assets/577e4283-2fc2-4655-9d7a-ab8e3f36a34f" />

The follow-up Lynis scan still reported an overall score of 60/100, but the UMASK control is now marked as OK instead of SUGGESTION. This shows that individual findings were resolved even though the coarse hardening index did not change. The remaining suggestions mainly relate to more advanced hardening (password policy tuning, kernel parameters, logging changes, separate /home partition, etc.), which are beyond the scope of this lab environment.
 
#### Lynis baseline hardening.
 <img width="1040" height="656" alt="image" src="https://github.com/user-attachments/assets/bff7d9c2-0a96-43e3-b2a6-e99167031276" />

#### Lynis, after remediation, is still 60, but UMASK control is OK.
```bash
sudo lynis audit system | tee lynis-week7-after.txt
```
<img width="1033" height="633" alt="image" src="https://github.com/user-attachments/assets/99153943-d006-451a-af0f-91d408656b75" />

## 4. Network Security Testing (Nmap)
From the workstation (Ubuntu WSL), I scanned the server with Nmap to verify that only the expected ports are reachable.

```bash
sudo apt update
sudo apt install -y nmap                      #This shows only the expected ports.
nmap -sV 192.168.1.171 | tee nmap-week7-basic.txt 
```
<img width="1151" height="278" alt="image" src="https://github.com/user-attachments/assets/797ee6cf-4b1a-49d1-9631-605948315f83" />

```bash
nmap -sV -p- 192.168.1.171 | tee nmap-week7-full.txt        #This checks all 65535 TCP ports.
```
<img width="1073" height="296" alt="image" src="https://github.com/user-attachments/assets/93885b3a-2f4e-421e-a6c9-3c44d3069bd2" />

The Nmap results showed:
- Port 22/tcp (ssh) open and responding with OpenSSH.
- Port 80/tcp (http) open when nginx is running.
- No additional unexpected ports open.

This confirms that the combination of service configuration and the `ufw` firewall is successfully restricting network exposure to the necessary services only.

## 5. SSH and Access Control Verification
To confirm that access control mechanisms from previous phases were still in effect, I re-checked SSH, firewall, AppArmor and fail2ban:

```bash
grep -E 'PermitRootLogin|PasswordAuthentication|PubkeyAuthentication|AllowUsers' /etc/ssh/sshd_config
````
<img width="1499" height="164" alt="image" src="https://github.com/user-attachments/assets/6375eace-1d05-4af9-8148-29787edc3b98" />

```bash
sudo ufw status verbose
```
<img width="777" height="98" alt="image" src="https://github.com/user-attachments/assets/cfb00dc9-8e88-48ae-9c07-427d744e83db" />

```bash
sudo aa-status
```
<img width="854" height="477" alt="image" src="https://github.com/user-attachments/assets/5496fbfa-796a-456f-9855-de67ecc3944b" />

```bash
sudo fail2ban-client status
sudo fail2ban-client status sshd
```
<img width="1919" height="423" alt="Screenshot 2025-12-09 093038" src="https://github.com/user-attachments/assets/cfe639ec-5732-4292-af70-9f0d5446ac3b" />

The results confirm that:
- SSH does not allow root logins and requires public key authentication for the `ashish` user.
- `ufw` is active, with a default deny policy for incoming traffic and rules that allow SSH (and HTTP) only from the workstation IP.
- AppArmor is enabled, and profiles for `sshd` and `nginx` are loaded in enforce mode.
- fail2ban is actively monitoring SSH login attempts via the `sshd` jail.

Together, these controls provide layered protection against unauthorised access and brute-force attacks.

## 6. Service Inventory and Justification
I examined network-facing services and enabled systemd services.

```bash
sudo ss -tulpen
```
<img width="1767" height="532" alt="image" src="https://github.com/user-attachments/assets/c55a852e-73a0-4ee5-8bb1-bbbff0656b02" />

```bash
systemctl list-unit-files --type=service --state=enabled
```
<img width="746" height="1038" alt="image" src="https://github.com/user-attachments/assets/5c0134e9-5f4a-45b4-8ebf-2a680f9e1c0e" />


```bash
systemctl --type=service --state=running               #List all enabled services from systemctl
```
<img width="1345" height="677" alt="image" src="https://github.com/user-attachments/assets/1f0fe526-7ce3-4928-9546-2c75be9977a6" />

##### No unnecessary network services are exposed, and all enabled services have a clear justification within the context of this coursework.

## 7. System Configuration Review
As a final verification step, I re-ran my Week 5 `security-baseline.sh` script, which checks:

- SSH configuration (root login, password authentication, key-based auth, `AllowUsers`)
- Firewall status and SSH rule for the workstation IP
- AppArmor enablement and profiles
- Automatic security updates
- fail2ban service and sshd jail

```bash
ssh ashish@192.168.1.171 "./security-baseline.sh"
```
<img width="1705" height="593" alt="image" src="https://github.com/user-attachments/assets/8feae69e-7283-46c7-a21a-bdebd31d6b3c" />

## 8. Remaining Risks and Recommendations
Although the system is significantly hardened compared to a default installation, some residual risks remain:

- The server still runs on a single host with no redundancy or backup strategy, so hardware or virtualisation failures could cause downtime.
- The password policy for local user accounts could be further strengthened, for example, by integrating stricter PAM password quality rules.
- Application-specific hardening, for example, the detailed nginx configuration, TLS, and web application security, is minimal because the server is used only for lab workloads.
- Centralised logging and alerting are not implemented, so detection of subtle attacks may be delayed.

If this system were promoted to a production environment, I would recommend:

- Implementing regular backups and disaster recovery plans.
- Enforcing a stricter password and account policy.
- Adding TLS, web application hardening and possibly a reverse proxy/WAF in front of nginx.
- Forwarding logs to a central log server with automated alerting on suspicious patterns.

## 9. Reflection

This final week shifted the focus from configuration to verification. Running Lynis and Nmap made me look at the system from an attacker’s perspective and validate that my firewall and service configuration actually limit the attack surface. The access control checks showed how SSH, AppArmor and fail2ban work together to protect the system. Writing up the service inventory and remaining risks helped me understand that a secure system is not just about enabling individual features, but about how they fit together into a coherent security posture. Overall, the audit confirmed that the lab server is appropriately hardened for its purpose, while also highlighting areas that would need further work in a real production deployment.
