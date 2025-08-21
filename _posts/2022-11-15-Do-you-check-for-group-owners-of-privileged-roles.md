---
future: true
published: true


title: "Do you check for group owners of privileged roles?"
date: 2022-11-15 19:40:00 +0100
categories: [Ethical Hacking, Azure]
tags: [ethical hacking, azure]     # TAG names should always be lowercase
author: jony
---

# Introduction
This is the second blog in the series. If you havent read the first one you can find it [here](https://jonyschats.nl/posts/What-is-the-MFA-status-of-your-users/). In the previous blogpost we focussed on retrieving userobjects from groups and roles and created a PowerShell cmdlet to get a overview of the users of 14 privileged roles. The created PowerShell cmdlet `Get-AzureADPrivilegedRolesMembers` searches recursivly through all these roles and return all the userobjects. This blog we will focus on group owners and enumerating them.

# Group Owners
A group owner can add members to the group without being part of the group. This is important when searching for privileged users since the following scenario could exist in Azure tenants:

A privileged role has one membership, which is a group. This group has one member Bob, but the user Alice own's this group. This owner can add members to this group, resulting in the assignment of the role to the user. In my tenant I configured the following: The `Authentication Administrator` role has one membership which is the group `Administrator`.  The group has one member which is `Groupuser` and the owner of the group is `GroupOwnerUser`. 

![Get-AzureADUserMFAConfiguration](/assets/img/groupowner.png)

When we run the created PowerShell cmdlet `Get-AzureADDirectoryRoleMemberRecursive` to list all the members of the role `Authenticated Administrators` it returns the user `Groupuser`: 

```
Get-AzureADDirectoryRole -ObjectId 594857e7-f7b4-4fd0-892b-6e1e66237b8f | Get-AzureADDirectoryRoleMemberRecursive

ObjectId                             DisplayName UserPrincipalName       UserType
--------                             ----------- -----------------       --------
eb815e66-31a5-45ca-bed8-2b0f5e24f62f GroupUser   GroupUser@jonyschats.nl Member
```

But it doesn't take into account the owner of the group. When we query the memberships of the role we see that the `Administrator` group is a member and when we query the owner of the group we see that `GroupOwnerUser` is owner of the group:

```
Get-AzureADDirectoryRoleMember -ObjectId 594857e7-f7b4-4fd0-892b-6e1e66237b8f

ObjectId                             DisplayName   Description
--------                             -----------   -----------
58b18ca8-2b9d-49b0-acfa-d0a68299a580 Administrator


Get-AzureADGroupOwner -ObjectId 58b18ca8-2b9d-49b0-acfa-d0a68299a580

ObjectId                             DisplayName    UserPrincipalName            UserType
--------                             -----------    -----------------            --------
2cc999ae-fe8e-4ce9-a18a-309d68f5bce2 GroupOwnerUser GroupOwnerUser@jonyschats.nl Member
```

# Changing the already created cmdlets
A group owner can add members to that group. This means that the user `GroupOwnerUser` can add members to the group `Administrator`, which will give the newly added user the role `Authentication administrator`. Group owners can add themself and should be considered as highly privileged users if they own a group which is a member of a high privileged role.

To retrieve group objects even if they are nested I changed the `Get-AzureADGroupMemberRecursive` function and added the `-ReturnGroups` parameter which will make the function return groups instead of users.

```powershell
...snip...
Write-Verbose -Message "Enumerating $($AzureGroup.DisplayName)"
$Members = Get-AzureADGroupMember -ObjectId $AzureGroup.ObjectId -All $true

if ($ReturnGroups){
	$UserMembers = $Members | Where-Object{$_.ObjectType -eq 'Group'}
	$Output += $UserMembers
	
	$GroupMembers = $Members | Where-Object{$_.ObjectType -eq 'Group'}
	If($GroupMembers){
		$UserMembers = $GroupMembers | ForEach-Object{ Get-AzureADGroupMemberRecursive -ReturnGroups -AzureGroup $_}
		$Output += $UserMembers
	}
}
else {
	$UserMembers = $Members | Where-Object{$_.ObjectType -eq 'User'}
	$Output += $UserMembers
	
	$GroupMembers = $Members | Where-Object{$_.ObjectType -eq 'Group'}
	If($GroupMembers){
		$UserMembers = $GroupMembers | ForEach-Object{ Get-AzureADGroupMemberRecursive -AzureGroup $_}
		$Output += $UserMembers
	}
}
...snip...
```

The cmdlet now returns only group objects when the parameter is given, to be sure it searched recusivly I created a extra group `NestedNestedgroup` and placed it inside the group `Nestedgroup`. The membership structure is as follows:
- Test Group
	- NestedGroup
		- NestedNestedgroup

```
Get-AzureADGroup -ObjectId f5108639-9aca-4694-864e-c4e00186706b | Get-AzureADGroupMemberRecursive -ReturnGroups

ObjectId                             DisplayName       Description
--------                             -----------       -----------
50c16bf1-5016-4dfe-8bcc-d273c3b02f02 NestedGroup
8510e6c0-fa05-49ca-ab5c-26d3d59a2e4d NestedNestedGroup
```

I also changed the `Get-AzureADGroupMemberRecursive` cmdlet and added the parameter `-ReturnGroups`. I made the same changes to `Get-AzureADPrivilegedRolesMembers`. When you run the cmdlet now with the parameter `-ReturnGroups` it will search recursivly through the 14 roles for group objects and return them:

```
Get-AzureADPrivilegedRolesMembers -ReturnGroups

ObjectId                             DisplayName   Description
--------                             -----------   -----------
58b18ca8-2b9d-49b0-acfa-d0a68299a580 Administrator
```

Then I tried adding a group to the `Administrator` group to create a nested group effect. But I figured out that is not possible since you can't create nested groups if the group is elligible to be assigned to a role. Well okay, atleast my function is future proof if add that ability. Facepalm!

When we run `Get-AzureADPrivilegedRolesMembers -ReturnGroups` and pipe it to `Get-AzureADGroupOwner`. The following will happen:
- loop through the 14 privileged roles
- list all the group objects and return group objects 
- these group objects will be piped to retrieve the owners of all the groups

```
Get-AzureADPrivilegedRolesMembers -ReturnGroups | Get-AzureADGroupOwner

ObjectId                             DisplayName    UserPrincipalName            UserType
--------                             -----------    -----------------            --------
2cc999ae-fe8e-4ce9-a18a-309d68f5bce2 GroupOwnerUser GroupOwnerUser@jonyschats.nl Member
```

So even though the user `GroupOwnerUser@jonyschats.nl` doesn't have a role that gives him high privileges, the user should be considered as a high privileged user since he owns a group which is member of a high privileged role. 

## Updating the overview cmdlet
To get a good overview again I changed the existing `Get-AzureADPrivilegedRolesOverview` cmdlet and added the groupcount, groups and groupowners attributes.

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
				$AdminRoleMembersUsers = Get-AzureADDirectoryRole -ObjectId $AdminRoleData.ObjectId | Get-AzureADDirectoryRoleMemberRecursive
				$AdminRoleMembersUsersCount = $AdminRoleMembersUsers | Sort-Object -Unique | Measure-Object
				
				$AdminRoleMembersGroups = Get-AzureADDirectoryRole -ObjectId $AdminRoleData.ObjectId | Get-AzureADDirectoryRoleMemberRecursive -ReturnGroups
				$AdminRoleMembersGroupsCount = $AdminRoleMembersGroups | Sort-Object -Unique | Measure-Object
				
				$GroupOwners = Get-AzureADDirectoryRole -ObjectId $AdminRoleData.ObjectId | Get-AzureADDirectoryRoleMemberRecursive -ReturnGroups | Get-AzureADGroupOwner
				
				$item = New-Object PSObject
				$item | Add-Member -type NoteProperty -Name 'Role' -Value $AdminRoleData.DisplayName
				$item | Add-Member -type NoteProperty -Name 'UserCount' -Value $AdminRoleMembersUsersCount.Count
				$item | Add-Member -type NoteProperty -Name 'Users' -Value $AdminRoleMembersUsers.UserPrincipalName
				$item | Add-Member -type NoteProperty -Name 'GroupCount' -Value $AdminRoleMembersGroupsCount.Count
				$item | Add-Member -type NoteProperty -Name 'Groups' -Value $AdminRoleMembersGroups.DisplayName
				$item | Add-Member -type NoteProperty -Name 'GroupOwners' -Value $GroupOwners.UserPrincipalName				
				$Output += $item
			}
			else {
				$item = New-Object PSObject
				$item | Add-Member -type NoteProperty -Name 'Role' -Value $AdminRole
				$item | Add-Member -type NoteProperty -Name 'UserCount' -Value "0"
				$item | Add-Member -type NoteProperty -Name 'GroupCount' -Value "0"
				$Output += $item
			}
		}
	}

	end {
        Return $Output | Sort-Object -Property UserCount -Descending
    }
}
```

The output of the new cmdlet looks like:
```
Get-AzureADPrivilegedRolesOverview | ft

Role                                    UserCount Users                   GroupCount Groups        GroupOwners
----                                    --------- -----                   ---------- ------        -----------
Authentication Administrator                    1 GroupUser@jonyschats.nl          1 Administrator GroupOwnerUser@jonyschats.nl
Global Administrator                            1 0xjs@jonyschats.nl               0
Privileged Role Administrator                   0                                  0
Privileged authentication administrator         0                                  0
Password administrator                          0                                  0
User Administrator                              0                                  0
SharePoint administrator                        0                                  0
Security administrator                          0                                  0
Cloud application administrator                 0                                  0
Billing administrator                           0                                  0
Application administrator                       0                                  0
Helpdesk administrator                          0                                  0
Exchange administrator                          0                                  0
Conditional Access administrator                0                                  0
```

# Quick recap
A quick recap of what I have learned about Azure AD identities:
- Roles can be assigned these three identities and these identities:
	- Users
	- Groups 
		- Can't have nested groups when assigned to roles.
		- Can have owners
	- ServicePrincipals 

In the next blog we will dive into ServicePrincipals and yes these can have owners too.

# GitHub
All the cmdlets can be found in my GitHub project [AzurePowerCommand](https://github.com/0xJs/AzurePowerCommands).
