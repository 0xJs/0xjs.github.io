---
future: true
published: true


title: "Identifying highly privileged identities and checking MFA status"
date: 2022-12-18 20:35:00 +0100
categories: [Ethical Hacking, Azure]
tags: [ethical hacking, azure]     # TAG names should always be lowercase
author:
  name: "Jony Schats"
---

# Introduction
This is the fourth blog in the series. If you haven't read the previous blog you can find it [here](https://jonyschats.nl/posts/Service-principals-did-you-know-they-can-have-owners-too/). In the previous blogs we discussed the different types of identities and how we can find high privileged identities. In those blogs PowerShell cmdlets were created which helped querying these different types of identities. In this blog everything will be tied together into one cmdlet and we will have a look at MFA.

A quick recap of what we have learned in the previous blogposts:
- Roles can be assigned these three identities and these identities:
	- Users
	- Groups 
		- Can't have nested groups when assigned to roles.
		- Can have owners (Users or service principals)
	- ServicePrincipals 
		- Can have Owners (Users)
		- Can have permissions

# Identifying highly privileged identities
Microsoft define [these](https://learn.microsoft.com/en-us/azure/active-directory/conditional-access/howto-conditional-access-policy-admin-mfa) 14 roles highly privileged. So to get a overview of all the highly privileged identities in the Azure Tenant we should query the following using the created cmdlets:

1 Users, we can query this with: `Get-AzureADPrivilegedRolesMembers`

```
Get-AzureADPrivilegedRolesMembers

ObjectId                             DisplayName UserPrincipalName       UserType
--------                             ----------- -----------------       --------
766787e8-82c1-4062-bfa9-5d4a4ca300f3 0xjs        0xjs@jonyschats.nl      Member
eb815e66-31a5-45ca-bed8-2b0f5e24f62f GroupUser   GroupUser@jonyschats.nl Member
```

2 Privileged group owners, we can query this with `Get-AzureADPrivilegedRolesMembers -ReturnGroup | Get-AzureADGroupOwner`.

```
Get-AzureADPrivilegedRolesMembers -ReturnGroup | Get-AzureADGroupOwner

ObjectId                             DisplayName    UserPrincipalName            UserType
--------                             -----------    -----------------            --------
2cc999ae-fe8e-4ce9-a18a-309d68f5bce2 GroupOwnerUser GroupOwnerUser@jonyschats.nl Member
```

3  Privileged service principals, we can query this with `Get-AzureADPrivilegedRolesMembers -ReturnServicePrincipals `

```
Get-AzureADPrivilegedRolesMembers -ReturnServicePrincipals

ObjectId                             AppId                                DisplayName
--------                             -----                                -----------
5530a9cf-a45a-4662-9179-eaa8d9089605 1a93dd32-5ade-4656-9ada-6a285676eb92 Test_enterpriseapp
```

4 Privileged service principal owners, we can query this with `Get-AzureADPrivilegedRolesMembers -ReturnServicePrincipals | Get-AzureADServicePrincipalOwner`

```
Get-AzureADPrivilegedRolesMembers -ReturnServicePrincipals | Get-AzureADServicePrincipalOwner

ObjectId                             DisplayName           UserPrincipalName                   UserType
--------                             -----------           -----------------                   --------
59b28e90-d96b-410b-acc6-fa9ee823bfbd ServicePrincipalOwner ServicePrincipalOwner@jonyschats.nl Member
```

5 Service principals with permissions configured/consented. Well... that is something for a whole new blog.

Since nobody likes to run multiple commands I created another PowerShell cmdlet, which is `Get-AzureADPrivilegedIdentities`. This cmdlet queries this information and gathers the user and service principal objects and returns them:

```powershell
Function Get-AzureADPrivilegedIdentities{
<#
.SYNOPSIS
Author: Jony Schats - 0xjs
Required Dependencies: Get-AzureADPrivilegedRolesMembers, Get-AzureADDirectoryRole, Get-AzureADDirectoryRoleMember, Get-AzureADGroupMember, Get-AzureADGroupMemberRecursive
Optional Dependencies: None

.DESCRIPTION
Recursively search through privileged roles and return users and service principal identities and their owners

.EXAMPLE
Get-AzureADPrivilegedIdentities

#>
	[cmdletbinding()]
	param(

	)
	
    Begin{
		# Check if Azure AD is loaded
		If(-not(Get-Command *Get-AzureADCurrentSessionInfo*)){
			Write-Host -ForegroundColor Red "AzureAD Module not imported, stopping"
			break
		}
        
		# Check connection with AzureAD
		try {
			$var = Get-AzureADTenantDetail
		}
		catch {
			Write-Host -ForegroundColor Red "You're not connected with AzureAD, Connect with Connect-AzureAD"
			break
		}
		
		$AllUsers = @()
		$AllServicePrincipals = @()
		$Output = @()
    }
	
	Process {
		# Retrieving privileged users member of role
		$Users = Get-AzureADPrivilegedRolesMembers
		$AllUsers += $users
		$UsersCount = ($Users | Measure-Object).count
		Write-Host "[+] Discovered $UsersCount users"
		
		# Retrieving privileged group owners
		$GroupOwners = Get-AzureADPrivilegedRolesMembers -ReturnGroup | Get-AzureADGroupOwner
		$AllUsers += $GroupOwners | Where-Object -Property ObjectType -Match User
		$AllServicePrincipals += $GroupOwners | Where-Object -Property ObjectType -Match ServicePrincipal
		$CountGroupOwners = ($Users | Measure-Object).count
		Write-Host "[+] Discovered $CountGroupOwners group owners"
		
		# Retrieving privileged service principals
		$ServicePrincipals = Get-AzureADPrivilegedRolesMembers -ReturnServicePrincipals
		$AllServicePrincipals += $ServicePrincipals
		$CountServicePrincipals = ($ServicePrincipals | Measure-Object).count
		Write-Host "[+] Discovered $CountServicePrincipals service principals"
		
		# Retrieving privileged service principal owners
		$ServicePrincipalOwners = Get-AzureADPrivilegedRolesMembers -ReturnServicePrincipals | Get-AzureADServicePrincipalOwner
		$AllUsers += $ServicePrincipalOwners
		$CountServicePrincipalOwners = ($ServicePrincipalOwners | Measure-Object).count
		Write-Host "[+] Discovered $CountServicePrincipalOwners service principal owners"
		
		$CountAllUsers = ($AllUsers | Measure-Object).count
		Write-Host "[+] Found $CountAllUsers highly privileged users"
		$CountAllServicePrincipals = ($AllServicePrincipals | Measure-Object).count
		Write-Host "[+] Found $CountAllServicePrincipals highly privileged service principals"
		
		$AllUsers = $AllUsers | Sort-Object -Unique
		$AllServicePrincipals = $AllServicePrincipals | Sort-Object -Unique
		
		$Output += $AllUsers
		$Output += $AllServicePrincipals
	}

	end {
		return $Output | ft -Force
    }
}
```

The output:

```
Get-AzureADPrivilegedIdentities

[+] Discovered 2 users
[+] Discovered 2 group owners
[+] Discovered 1 service principals
[+] Discovered 1 service principal owners
[+] Found 4 highly privileged users
[+] Found 2 highly privileged service principals

ObjectId                             DisplayName              UserPrincipalName                   UserType
--------                             -----------              -----------------                   --------
2cc999ae-fe8e-4ce9-a18a-309d68f5bce2 GroupOwnerUser           GroupOwnerUser@jonyschats.nl        Member
59b28e90-d96b-410b-acc6-fa9ee823bfbd ServicePrincipalOwner    ServicePrincipalOwner@jonyschats.nl Member
766787e8-82c1-4062-bfa9-5d4a4ca300f3 0xjs                     0xjs@jonyschats.nl                  Member
eb815e66-31a5-45ca-bed8-2b0f5e24f62f GroupUser                GroupUser@jonyschats.nl             Member
5530a9cf-a45a-4662-9179-eaa8d9089605 Test_enterpriseapp
5fef25d5-9886-42df-9d98-de37f6ffd299 Test_enterpriseapp_owner
```

# MFA Configuration
One good thing to check is if all users have MFA enabled. To check this the command `Get-MsolUser` from the [MSOnline](https://learn.microsoft.com/en-us/powershell/module/msonline/?view=azureadps-1.0) module can be used. The interesting attributes related to MFA are: `StrongAuthenticationMethods` and `StrongAuthenticationRequirements`. When we query the user `0xjs`  from my tenant we can see the following:

```
Get-MsolUser -ObjectId 766787e8-82c1-4062-bfa9-5d4a4ca300f3 | Select-Object Displayname, StrongAuthenticationMethods, StrongAuthenticationRequirements |fl


DisplayName                      : 0xjs
StrongAuthenticationMethods      : {Microsoft.Online.Administration.StrongAuthenticationMethod, Microsoft.Online.Administration.StrongAuthenticationMethod,
                                   Microsoft.Online.Administration.StrongAuthenticationMethod, Microsoft.Online.Administration.StrongAuthenticationMethod}
StrongAuthenticationRequirements : {Microsoft.Online.Administration.StrongAuthenticationRequirement}
```

The attribute `StrongAuthenticationMethods` holds another PowerShell object and when we open it up we retrieve every authentication method configured and which one is the default. In the example below the PhoneAppNotification is the default MFA method but SMS, Phonecall and a OTP method is also enabled for the user `0xjs`. This is because I configured every authentication method possible on my account.

```
(Get-MsolUser -ObjectId 766787e8-82c1-4062-bfa9-5d4a4ca300f3 | Select-Object Displayname, StrongAuthenticationMethods, StrongAuthenticationRequirements).StrongAuthenticationMethods

ExtensionData                                    IsDefault MethodType
-------------                                    --------- ----------
System.Runtime.Serialization.ExtensionDataObject     False OneWaySMS
System.Runtime.Serialization.ExtensionDataObject     False TwoWayVoiceMobile
System.Runtime.Serialization.ExtensionDataObject     False PhoneAppOTP
System.Runtime.Serialization.ExtensionDataObject      True PhoneAppNotification
```

When we query the user `SecurityReader` where I didn't configure every method we only see two methods enabled:

```
(Get-MsolUser -ObjectId fb8a7905-e32c-4431-9e66-2968013f924f | Select-Object Displayname, StrongAuthenticationMethods, StrongAuthenticationRequirements).StrongAuthenticationMethods

ExtensionData                                    IsDefault MethodType
-------------                                    --------- ----------
System.Runtime.Serialization.ExtensionDataObject      True PhoneAppNotification
System.Runtime.Serialization.ExtensionDataObject     False PhoneAppOTP
```

The attribute `StrongAuthenticationRequirements` holds the Per-User MFA configuration. This configuration should not be used if Conditional Access policies exist since it will overwrite them and will always require MFA even if the policies say it doesn't need to.

```
(Get-MsolUser -ObjectId 766787e8-82c1-4062-bfa9-5d4a4ca300f3 | Select-Object Displayname, StrongAuthenticationMethods, StrongAuthenticationRequirements).StrongAuthenticationRequirements

ExtensionData                                    RelyingParty RememberDevicesNotIssuedBefore State
-------------                                    ------------ ------------------------------ -----
System.Runtime.Serialization.ExtensionDataObject *            11/11/2022 10:36:14 AM         Enforced
```

To combine this information I created the cmdlet `Get-AzureADUserMFAConfiguration`. This cmdlet takes a user from `Get-MsolUser` or `Get-AzureAdUser`. By default it will display if MFA is configured and what the default MFA method is and what the status is of Per-User MFA.

```powershell
Function Get-AzureADUserMFAConfiguration{
<#
.SYNOPSIS
Author: Jony Schats - 0xjs
Required Dependencies: Get-MsolUser
Optional Dependencies: None

.DESCRIPTION
Get MFA configuration data for the user. Requires a user as input.

.PARAMETER Detailed
If specified will create detailed MFA configuration objects

.EXAMPLE
Get-AzureADUser | Get-AzureADUserMFAConfiguration
Get MFA configuration data of all users

.EXAMPLE
Get-MsolUser | Get-AzureADUserMFAConfiguration
Get MFA configuration data of all users

.EXAMPLE
Get-MsolUser | Get-AzureADUserMFAConfiguration -Detailed
Get detailed MFA configuration data of all users

.EXAMPLE
Get-AzureADPrivilegedRolesMembers | Get-AzureADUserMFAConfiguration
Get MFA configuration data for all users of privileges roles

.EXAMPLE
Get-AzureADPrivilegedRolesMembers | Get-AzureADUserMFAConfiguration -Detailed
Get detailed MFA configuration data for all users of privileges roles
#>
	[OutputType('System.Management.Automation.PSCustomObject')]
	[cmdletbinding()]
	param(
	[parameter(Mandatory=$True,ValueFromPipeline=$true)]
	$User,
	[Parameter(Mandatory = $false)]
	[Switch]
	$Detailed
	)
		Begin{
			# Check if MSOnline is loaded
			If(-not(Get-Command *Get-MsolCompanyInformation*)){
				Write-Host -ForegroundColor Red "MSOnline Module not imported, stopping"
				break
			}
			
			# Check connection with MSOnline
			try {
				$var = Get-MsolDomain -ErrorAction Stop > $null
			}
			catch {
				Write-Host -ForegroundColor Red "You're not connected with MSOnline, Connect with Connect-MsolService"
				break
			}
		
			$Output = @()
		}
		
		Process {
			$User = Get-MsolUser -ObjectId $_.ObjectId 
			
			$MFADefault = ""
			$MFAConfigured = ""
			$MFADefaultMethod = ""
				
			$MFADefault = $User.StrongAuthenticationMethods | Where-Object -Property IsDefault -EQ $True | Select-Object -ExpandProperty MethodType
			
			if ($User.StrongAuthenticationMethods) {
				$MFAConfigured = $true
			}
			else {
				$MFAConfigured = $false
			}

			if ($MFADefault -eq "PhoneAppNotification") {
				$MFADefaultMethod = "Microsoft Authenticator"
			}
			elseif ($MFADefault -eq "PhoneAppOTP") {
				$MFADefaultMethod = "HW token / Authenticator"
			}
			elseif ($MFADefault -eq "OneWaySMS") {
				$MFADefaultMethod = "SMS"
			}
			elseif ($MFADefault -eq "TwoWayVoiceMobile") {
				$MFADefaultMethod = "Voice"
			}
			
			$item = New-Object PSObject
			$item | Add-Member -type NoteProperty -Name 'UserPrincipalName' -Value $User.UserPrincipalName
			$item | Add-Member -type NoteProperty -Name 'MFA Configured' -Value $MFAConfigured
			$item | Add-Member -type NoteProperty -Name 'MFA Default' -Value $MFADefaultMethod
			
			if ($User.StrongAuthenticationRequirements) {
					$item | Add-Member -type NoteProperty -Name 'Per-User MFA' -Value $User.StrongAuthenticationRequirements.State
				} else {
					$item | Add-Member -type NoteProperty -Name 'Per-User MFA' -Value "-"
			}
			
			if ($Detailed){
				if ($User.StrongAuthenticationMethods.MethodType -contains "OneWaySMS") {
				$item | Add-Member -type NoteProperty -Name 'OneWaySMS' -Value $true
				} else {
					$item | Add-Member -type NoteProperty -Name 'OneWaySMS' -Value "-"
				}
				
				if ($User.StrongAuthenticationMethods.MethodType -contains "TwoWayVoiceMobile") {
					$item | Add-Member -type NoteProperty -Name 'TwoWayVoiceMobile' -Value $true
				} else {
					$item | Add-Member -type NoteProperty -Name 'TwoWayVoiceMobile' -Value "-"
				}
				
				if ($User.StrongAuthenticationMethods.MethodType -contains "PhoneAppOTP") {
					$item | Add-Member -type NoteProperty -Name 'PhoneAppOTP' -Value $true
				} else {
					$item | Add-Member -type NoteProperty -Name 'PhoneAppOTP' -Value "-"
				}
				
				if ($User.StrongAuthenticationMethods.MethodType -contains "PhoneAppNotification") {
					$item | Add-Member -type NoteProperty -Name 'PhoneAppNotification' -Value $true
				} else {
					$item | Add-Member -type NoteProperty -Name 'PhoneAppNotification' -Value "-"
				}
				
				if ($User.StrongAuthenticationUserDetails.Email) {
					$item | Add-Member -type NoteProperty -Name 'Registered Email' -Value $User.StrongAuthenticationUserDetails.Email
				} else {
					$item | Add-Member -type NoteProperty -Name 'Registered Email' -Value "-"
				}
				
				if ($User.StrongAuthenticationUserDetails.PhoneNumber) {
					$item | Add-Member -type NoteProperty -Name 'Registered Phone' -Value $User.StrongAuthenticationUserDetails.PhoneNumber
				} else {
					$item | Add-Member -type NoteProperty -Name 'Registered Phone' -Value "-"
				}
			}
			
			$Output += $item
		}

		end {
			$Output
		}
}
```

Example output:

```
Get-MsolUser -ObjectId 766787e8-82c1-4062-bfa9-5d4a4ca300f3 | Get-AzureADUserMFAConfiguration

UserPrincipalName  MFA Configured MFA Default             Per-User MFA
-----------------  -------------- -----------             ------------
0xjs@jonyschats.nl           True Microsoft Authenticator Enforced
```

It is also possible to pipe all users to this cmdlet (might take a while in big environments):

```
Get-MsolUser | Get-AzureADUserMFAConfiguration

UserPrincipalName                   MFA Configured MFA Default              Per-User MFA
-----------------                   -------------- -----------              ------------
NestedGroupUser@jonyschats.nl                 True HW token / Authenticator -
GroupOwnerUser@jonyschats.nl                 False                          -
ServicePrincipalOwner@jonyschats.nl          False                          -
0xjs@jonyschats.nl                            True Microsoft Authenticator  Enforced
GroupUser@jonyschats.nl                       True Microsoft Authenticator  -
SecurityReader@jonyschats.nl                  True Microsoft Authenticator  -
```

If detailed information is prefered there is a flag for that, use the `-Detailed` flag and the output will be:

```
Get-MsolUser | Get-AzureADUserMFAConfiguration -Detailed


UserPrincipalName    : NestedGroupUser@jonyschats.nl
MFA Configured       : True
MFA Default          : HW token / Authenticator
Per-User MFA         : -
OneWaySMS            : -
TwoWayVoiceMobile    : -
PhoneAppOTP          : True
PhoneAppNotification : -
Registered Email     : -
Registered Phone     : -

...snip...

UserPrincipalName    : 0xjs@jonyschats.nl
MFA Configured       : True
MFA Default          : Microsoft Authenticator
Per-User MFA         : Enforced
OneWaySMS            : True
TwoWayVoiceMobile    : True
PhoneAppOTP          : True
PhoneAppNotification : True
Registered Email     : fakeemail@jonyschats.nl
Registered Phone     : +31 06123456789

...snip...

UserPrincipalName    : SecurityReader@jonyschats.nl
MFA Configured       : True
MFA Default          : Microsoft Authenticator
Per-User MFA         : -
OneWaySMS            : -
TwoWayVoiceMobile    : -
PhoneAppOTP          : True
PhoneAppNotification : True
Registered Email     : -
Registered Phone     : -
```

Remember when we were talking about privileged users and objects etc. We can also pipe these users to the cmdlet and quickly retrieve the MFA configuration of all privileged users within the tenant. Serviceprincipals can't have MFA configured!

```
Get-AzureADPrivilegedRolesMembers | Get-AzureADUserMFAConfiguration

UserPrincipalName       MFA Configured MFA Default             Per-User MFA
-----------------       -------------- -----------             ------------
0xjs@jonyschats.nl                True Microsoft Authenticator Enforced
GroupUser@jonyschats.nl           True Microsoft Authenticator -
```

We can quickly use the created cmdlets to get a overview of all privileged objects within the tenant and their MFA status and configuration and use this data to protect the Azure tenant.

# GitHub
All the cmdlets can be found in my GitHub project [AzurePowerCommands](https://github.com/0xJs/AzurePowerCommands).