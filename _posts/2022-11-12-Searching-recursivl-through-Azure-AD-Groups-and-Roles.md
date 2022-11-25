---
future: true
published: true


title: "Searching recursivly through Azure AD Groups and Roles"
date: 2022-11-12 12:45:00 +0100
categories: [Ethical Hacking, Azure]
tags: [ethical hacking, azure]     # TAG names should always be lowercase
author:
  name: "Jony Schats"
---

# Introduction
During some courses I have been working with the AzureAD PowerShell Module to retrieve information from Users, Groups, Roles etc. While working with these cmdlets it was annoying that it doesn't have a functionality to recursivly search through group objects. So I wrote two modules to recursivly search through group objects and roles and only return the user objects. Then I also wanted to get a overview of the Azure AD roles and privileged roles and their users? This blogpost will be the first of the series about priviliged identities. In this blogpost I will describe and share the PowerShell cmdlets I created which will help with enumerating Azure AD Group and Role user memberships.

In Azure there are three identities that can have access to resources:
- Users
- Groups
	- Can have users, serviceprincipals or other groups as members
- ServicePrincipals

In the first blog we will focus on retrieving users out of groups and roles.

# The problem with Azure AD cmdlets when querying for memberships
When quering members of a group with the cmdlet `Get-AzureADGroupMember` the cmdlet returns user, serviceprincipal and group objects. In the output below an example is shown where it returns the user `GroupUser` and the group `NestedGroup`.

```
Get-AzureADGroupMember -ObjectId f5108639-9aca-4694-864e-c4e00186706b

ObjectId                             DisplayName UserPrincipalName       UserType
--------                             ----------- -----------------       --------
eb815e66-31a5-45ca-bed8-2b0f5e24f62f GroupUser   GroupUser@jonyschats.nl Member

DeletionTimestamp            :
ObjectId                     : 50c16bf1-5016-4dfe-8bcc-d273c3b02f02
ObjectType                   : Group
Description                  :
DirSyncEnabled               :
DisplayName                  : NestedGroup
LastDirSyncTime              :
Mail                         :
MailEnabled                  : False
MailNickName                 : 295cb161-8
OnPremisesSecurityIdentifier :
ProvisioningErrors           : {}
ProxyAddresses               : {}
SecurityEnabled              : True
```

But it is missing the user that is part of the group `NestedGroup` and I would like to only retrieve the userobjects. The same happens for roles since you can assign groups and users to a role.

```
Get-AzureADDirectoryRoleMember -ObjectId 598a6cfe-5d1a-42a7-81b6-76f4ab077152

ObjectId                             DisplayName    UserPrincipalName            UserType
--------                             -----------    -----------------            --------
fb8a7905-e32c-4431-9e66-2968013f924f SecurityReader SecurityReader@jonyschats.nl Member

DeletionTimestamp            :
ObjectId                     : 15c39279-6020-4a08-891f-8d76b6b86036
ObjectType                   : Group
Description                  :
DirSyncEnabled               :
DisplayName                  : Security Reader AD Group
LastDirSyncTime              :
Mail                         :
MailEnabled                  : False
MailNickName                 : 62ac515e-a
OnPremisesSecurityIdentifier :
ProvisioningErrors           : {}
ProxyAddresses               : {}
SecurityEnabled              : True
```

The output is missing the user of the group `Security Reader AD Group`.

# New cmdlets for recursively searching through groups and roles
### Get-AzureADGroupMemberRecursive
So I created the PowerShell cmdlet `Get-AzureADGroupMemberRecursive` which takes a group object as input and recursivly searches through all group objects and returns user objects. The code of the cmdlet is:

```powershell
Function Get-AzureADGroupMemberRecursive{
<#
.SYNOPSIS
Author: Jony Schats - 0xjs
Required Dependencies: Get-AzureADGroupMember
Optional Dependencies: None
.DESCRIPTION
Recursively search through groups and only return unique user objects. Requires the Get-AzureADGroup as input.
.EXAMPLE
Get-AzureADGroup -ObjectId <ID> | Get-AzureADGroupMemberRecursive
.EXAMPLE
Get-AzureADGroup | Where-Object -Property Displayname -eq "<GROUP>" | Get-AzureADGroupMemberRecursive
#>
	[cmdletbinding()]
	param(
	[parameter(Mandatory=$True,ValueFromPipeline=$true)]
	$AzureGroup
	)
		Begin{
			If(-not(Get-AzureADCurrentSessionInfo)){Connect-AzureAD}
			$Output = @()
		}
		Process {
			Write-Verbose -Message "Enumerating $($AzureGroup.DisplayName)"
			$Members = Get-AzureADGroupMember -ObjectId $AzureGroup.ObjectId -All $true
			$UserMembers = $Members | Where-Object{$_.ObjectType -eq 'User'}
			$Output += $UserMembers
			
			$GroupMembers = $Members | Where-Object{$_.ObjectType -eq 'Group'}
			If($GroupMembers){
				$UserMembers = $GroupMembers | ForEach-Object{ Get-AzureADGroupMemberRecursive -AzureGroup $_}
				$Output += $UserMembers
			}
		}
		end {
			Return $Output | Sort-Object -Unique
		}
}
```

Below is example output where it retrieves the user `GroupUser` of the group `test group`, but also the user `NestedGroupUser` from the group `NestedGroup` which is member of the group `test group`. The membership structure is as follows:

- Test group
	- GroupUser@jonyschats.nl
	- NestedGroup
		- NestedGroupUser@jonyschats.nl

```
Get-AzureADGroup -ObjectId f5108639-9aca-4694-864e-c4e00186706b | Get-AzureADGroupMemberRecursive

ObjectId                             DisplayName     UserPrincipalName             UserType
--------                             -----------     -----------------             --------
1a9a26f4-297a-4dec-95b3-e502ec8e9dfc NestedGroupUser NestedGroupUser@jonyschats.nl Member
eb815e66-31a5-45ca-bed8-2b0f5e24f62f GroupUser       GroupUser@jonyschats.nl       Member
```

### Get-AzureADDirectoryRoleMemberRecursive
Roles can also have user and group objects as members. So I created the PowerShell cmdlet `Get-AzureADDirectoryRoleMemberRecursive` which uses the `Get-AzureADGroupMemberRecursive` cmdlet to recursivly search through all the groups assigned to the role and returns all the users. The code of the cmdlet is:

```powershell
Function Get-AzureADDirectoryRoleMemberRecursive{
<#
.SYNOPSIS
Author: Jony Schats - 0xjs
Required Dependencies: Get-AzureADDirectoryRoleMember, Get-AzureADGroupMember, Get-AzureADGroupMemberRecursive
Optional Dependencies: None
.DESCRIPTION
Recursively search through roles and only return unique user objects. Requires the Get-AzureADDirectoryRole as input.
.EXAMPLE
Get-AzureADDirectoryRole -ObjectId <ID> | Get-AzureADDirectoryRoleMemberRecursive
.EXAMPLE
Get-AzureADDirectoryRole | Where-Object -Property Displayname -eq "<ROLE>" | Get-AzureADDirectoryRoleMemberRecursive
#>
	[cmdletbinding()]
	param(
	[parameter(Mandatory=$True,ValueFromPipeline=$true)]
	$RoleGroup
	)
		Begin{
			If(-not(Get-AzureADCurrentSessionInfo)){Connect-AzureAD}
			$Output = @()
		}
		Process {
			Write-Verbose -Message "Enumerating $($RoleGroup.DisplayName)"
			$Members = Get-AzureADDirectoryRoleMember -ObjectId $RoleGroup.ObjectId
			$UserMembers = $Members | Where-Object{$_.ObjectType -eq 'User'}
			$Output += $UserMembers
			
			$GroupMembers = $Members | Where-Object{$_.ObjectType -eq 'Group'}
			If($GroupMembers){
				$UserMembers = $GroupMembers | ForEach-Object{ Get-AzureADGroupMemberRecursive -AzureGroup $_}
				$Output += $UserMembers
			}
		}
		end {
			Return $Output | Sort-Object -Unique
		}
}
```

Below is example output where it retrieves the user `SecurityReader` of the role `Security Reader` but also the user `NestedGroupUser` from the group `NestedGroup` which is member of the role. The membership structure is as follows:

- Security Reader
	- SecurityReader@jonyschats.nl
	- NestedGroup
		- NestedGroupUser@jonyschats.nl

```
Get-AzureADDirectoryRole -ObjectId 598a6cfe-5d1a-42a7-81b6-76f4ab077152 | Get-AzureADDirectoryRoleMemberRecursive

ObjectId                             DisplayName     UserPrincipalName             UserType
--------                             -----------     -----------------             --------
1a9a26f4-297a-4dec-95b3-e502ec8e9dfc NestedGroupUser NestedGroupUser@jonyschats.nl Member
fb8a7905-e32c-4431-9e66-2968013f924f SecurityReader  SecurityReader@jonyschats.nl  Member
```

# New cmdlets for enumerating roles
### Get-AzureADDirectoryRoleOverview
AzureAD has a lot of [built-in roles](https://learn.microsoft.com/en-us/azure/active-directory/roles/permissions-reference) but the AzureAD cmdlet `Get-AzureADDirectoryRole` only returns the roles which are [activated](https://github.com/Azure/azure-docs-powershell-azuread/issues/245). Using this information we can easily request all activated roles and retrieve its users with the cmdlet I created earlier. But I woud like to get an overview of how many users each role has and who the users are. So I created the cmdlet `Get-AzureADDirectoryRoleOverview`:

```powershell
Function Get-AzureADDirectoryRoleOverview{
<#
.SYNOPSIS
Author: Jony Schats - 0xjs
Required Dependencies: Get-AzureADDirectoryRole, Get-AzureADDirectoryRoleMember, Get-AzureADGroupMember, Get-AzureADGroupMemberRecursive
Optional Dependencies: None
.DESCRIPTION
Recursively search through all active Azure AD roles and return a overview of the amount of members a role has and the members itself.
.EXAMPLE
Get-AzureADDirectoryRoleOverview
#>
    Begin{
        If(-not(Get-AzureADCurrentSessionInfo)){Connect-AzureAD}
		$Output = @()
    }
	
	Process {
		$AzureADRoles = Get-AzureADDirectoryRole

		foreach ($AzureADRole in $AzureADRoles) {
			Write-Verbose -Message "Enumerating $($AzureADRole.DisplayName)"
				
			# Retrieve members of the AdminRole
			$RoleMembers = Get-AzureADDirectoryRole -ObjectId $AzureADRole.ObjectId | Get-AzureADDirectoryRoleMemberRecursive
			$RoleMembersCount = $RoleMembers | Sort-Object -Unique | Measure-Object
			
			$item = New-Object PSObject
			$item | Add-Member -type NoteProperty -Name 'Role' -Value $AzureADRole.DisplayName
			$item | Add-Member -type NoteProperty -Name 'UserCount' -Value $RoleMembersCount.Count
			$item | Add-Member -type NoteProperty -Name 'Members' -Value $RoleMembers.UserPrincipalName
			$Output += $item
			}
	}

	end {
        Return $Output | Sort-Object -Property UserCount -Descending
    }
}
```

The cmdlet retrieves all active roles and their memberships, counts the userobjects after searching recursivly and creates new PowerShell objects with the attributes `Rolename`, `Usercount` and `Members`.

```
Get-AzureADDirectoryRoleOverview

Role                 UserCount Members
----                 --------- -------
Security Reader              2 {NestedGroupUser@jonyschats.nl, SecurityReader@jonyschats.nl}
Global Reader                1 SecurityReader@jonyschats.nl
Global Administrator         1 0xjs@jonyschats.nl
```

### Get-AzureADPrivilegedRolesMembers
We got an overview of all roles used within Azure, the member count and listing the members, but what about privileged roles? There is a warning in Microsoft Security Center which recommends MFA for atleast [these](https://learn.microsoft.com/en-us/azure/active-directory/conditional-access/howto-conditional-access-policy-admin-mfa) privileged roles. So I created the PowerShell cmdlet `Get-AzureADPrivilegedRolesMembers` which searches recursivly through all these roles and return all user object:

```powershell
Function Get-AzureADPrivilegedRolesMembers{
<#
.SYNOPSIS
Author: Jony Schats - 0xjs
Required Dependencies: Get-AzureADDirectoryRole, Get-AzureADDirectoryRoleMember, Get-AzureADGroupMember, Get-AzureADGroupMemberRecursive
Optional Dependencies: None
.DESCRIPTION
Recursively search through privileged roles and only return unique user objects.
.EXAMPLE
Get-AzureADPrivilegedRolesMembers
#>
    Begin{
        If(-not(Get-AzureADCurrentSessionInfo)){Connect-AzureAD}
		$AdminRoles = "Global administrator", "Application administrator", "Authentication Administrator", "Billing administrator", "Cloud application administrator", "Conditional Access administrator", "Exchange administrator", "Helpdesk administrator", "Password administrator", "Privileged authentication administrator", "Privileged Role Administrator", "Security administrator", "SharePoint administrator", "User administrator"
		$Output = @()
    }
	
	Process {
		foreach ($AdminRole in $AdminRoles) {			
			$AdminRoleData = Get-AzureADDirectoryRole | Where-Object -Property Displayname -eq $AdminRole
			Write-Verbose -Message "Enumerating $($AdminRoleData.DisplayName)"
			
			# If the role is populated
			if ($AdminRoleData -ne $null){
				
				# Retrieve members of the AdminRole
				$AdminRoleMembers = Get-AzureADDirectoryRole -ObjectId $AdminRoleData.ObjectId | Get-AzureADDirectoryRoleMemberRecursive
				$Output += $AdminRoleMembers
			}
		}
	}

	end {
        Return $Output | Sort-Object -Unique
    }
}
```

I only have one `Global Administrator` in my tenant so it returns just one object. But it will go through all 14 roles and recursivly search for all users.

```
Get-AzureADPrivilegedRolesMembers

ObjectId                             DisplayName UserPrincipalName  UserType
--------                             ----------- -----------------  --------
766787e8-82c1-4062-bfa9-5d4a4ca300f3 0xjs        0xjs@jonyschats.nl Member
```

### Get-AzureADPrivilegedRolesOverview
We already created a overview of all active roles and its members, but I also wanted to have this overview for the privileged roles which we discussed earlier. So I created `Get-AzureADPrivilegedRolesOverview`:

```powershell
Function Get-AzureADPrivilegedRolesOverview{
<#
.SYNOPSIS
Author: Jony Schats - 0xjs
Required Dependencies: Get-AzureADDirectoryRole, Get-AzureADDirectoryRoleMember, Get-AzureADGroupMember, Get-AzureADGroupMemberRecursive
Optional Dependencies: None
.DESCRIPTION
Recursively search through privileged Azure AD roles and return a overview of the amount of members a role has and the members itself.
.EXAMPLE
Get-AzureADPrivilegedRolesOverview
#>
    Begin{
        If(-not(Get-AzureADCurrentSessionInfo)){Connect-AzureAD}
		$AdminRoles = "Global administrator", "Application administrator", "Authentication Administrator", "Billing administrator", "Cloud application administrator", "Conditional Access administrator", "Exchange administrator", "Helpdesk administrator", "Password administrator", "Privileged authentication administrator", "Privileged Role Administrator", "Security administrator", "SharePoint administrator", "User administrator"
		$Output = @()
    }
	
	Process {		
		foreach ($AdminRole in $AdminRoles) {
			$AdminRoleData = Get-AzureADDirectoryRole | Where-Object -Property Displayname -eq $AdminRole
			Write-Verbose -Message "Enumerating $($AdminRoleData.DisplayName)"

			# If the role is populated
			if ($AdminRoleData -ne $null){
				
				# Retrieve members of the AdminRole
				$AdminRoleMembers = Get-AzureADDirectoryRole -ObjectId $AdminRoleData.ObjectId | Get-AzureADDirectoryRoleMemberRecursive
				$AdminRoleMembersCount = $AdminRoleMembers | Sort-Object -Unique | Measure-Object
				
				$item = New-Object PSObject
				$item | Add-Member -type NoteProperty -Name 'Role' -Value $AdminRoleData.DisplayName
				$item | Add-Member -type NoteProperty -Name 'UserCount' -Value $AdminRoleMembersCount.Count
				$item | Add-Member -type NoteProperty -Name 'Members' -Value $AdminRoleMembers.UserPrincipalName
				$Output += $item
			}
			else {
				$item = New-Object PSObject
				$item | Add-Member -type NoteProperty -Name 'Role' -Value $AdminRole
				$item | Add-Member -type NoteProperty -Name 'UserCount' -Value "0"
				$Output += $item
			}
		}
	}

	end {
        Return $Output | Sort-Object -Property UserCount -Descending
    }
}
```

The PowerShell cmdlet also adds a object for the role even if its empty, but you can easily filter this by piping it to a where-object statement such as `| Where-Object -Property UserCount -NE 0`. 


```
Get-AzureADPrivilegedRolesOverview

Role                                    UserCount Members
----                                    --------- -------
Global Administrator                            1 0xjs@jonyschats.nl
Privileged Role Administrator                   0
Privileged authentication administrator         0
Password administrator                          0
User administrator                              0
SharePoint administrator                        0
Security administrator                          0
Helpdesk administrator                          0
Billing administrator                           0
Authentication Administrator                    0
Application administrator                       0
Exchange administrator                          0
Conditional Access administrator                0
Cloud application administrator                 0


Get-AzureADPrivilegedRolesOverview | Where-Object -Property UserCount -NE 0

Role                 UserCount Members
----                 --------- -------
Global Administrator         1 0xjs@jonyschats.nl
```

# GitHub
All the cmdlets can be found in my GitHub project [AzurePowerCommand](https://github.com/0xJs/AzurePowerCommands).

The next blog in the series will be about GroupOwners.