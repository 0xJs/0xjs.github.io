---
future: true
published: true


title: "Service principals, did you know they can have owners too?"
date: 2022-11-23 12:40:00 +0100
categories: [Ethical Hacking, Azure]
tags: [ethical hacking, azure]     # TAG names should always be lowercase
author:
  name: "Jony Schats"
---

# Introduction
This is the third blog in the series. If you havent read the [first](https://jonyschats.nl/posts/What-is-the-MFA-status-of-your-users/) or [second](https://jonyschats.nl/posts/Do-you-check-for-group-owners-of-privileged-roles/) blog I recommend reading them. In the previous blogpost we focussed on retrieving group owners and that they should be considered as high privileged users if the group he owns is member of a high privileged role. In this blogpost we will have a look at Service Principals and they can have owners too!

# Service princpals
In the first blog we discussed that there are three identities that can access resources; Users, Groups and Service Principals. You might be wondering what is a service principal, [Microsoft](https://learn.microsoft.com/en-us/azure/active-directory/develop/app-objects-and-service-principals) defines; 

> A service principal is created in each tenant where the application is used and references the globally unique app object. The service principal object defines what the app can actually do in the specific tenant, who can access the app, and what resources the app can access.

As an example I created an app in my Azure Active Directory in "App registrations" with the name `Test_enterpriseapp`.

![Exam timeline](/assets/img/app_registration.png)

When creating the app a service principal is automatically registered and can be found in the "Enterprise application" section of Azure Active Directory.

![Exam timeline](/assets/img/service_principal.png)

If my app needs to access any resources within my Azure tenant it will use this service principal. For example; I could give this service principal permissions to read certain keys or certificates out of a Azure keyvault. When I give a app from somebody else permissions to access my tenant, a service principal is also created. This could be an app from somebody else or for Example [Microsoft Graph Explorer](https://developer.microsoft.com/en-us/graph/graph-explorer). After logging in I receive a prompt to give permissions:

![Exam timeline](/assets/img/graph.png)

After giving consent the service principal is added to our tenant with permissions and roles

![Exam timeline](/assets/img/service_principal2.png)

In a later blog we will dive into these permissions since its a complicated concept.

# Service princpals owners
Just like the groups, a service principal can also have an owner. So if a high priviliged role has a service principal in its member and that service principal has a owner, this user should also be considered highly privileged. In my tenant I added the service principal `Test_enterpriseapp` to the `Authentication Administrator` role and added a user `ServicePrincipalOwner` as owner of the service principal.

To accomodate retrieving the service principals in the cmdlets of [AzurePowerCommands](https://github.com/0xJs/AzurePowerCommands) I added the `-ReturnServicePrincipals` parameter which will return service principals in the same way I did for the `-ReturenGroups`.

When you run the cmdlet `Get-AzureADPrivilegedRolesMembers` with the parameter `-ReturnServicePrincipals` it will search recursivly through the 14 roles for service principals and return them:

```
Get-AzureADPrivilegedRolesMembers -ReturnServicePrincipals

ObjectId                             AppId                                DisplayName
--------                             -----                                -----------
5530a9cf-a45a-4662-9179-eaa8d9089605 1a93dd32-5ade-4656-9ada-6a285676eb92 Test_enterpriseapp
```

When we run `Get-AzureADPrivilegedRolesMembers -ReturnServicePrincipals` and pipe it to `Get-AzureADServicePrincipalOwner`. The following will happen:

- loop through the 14 privileged roles
- search for all service principals and also in groups and return service principals
- these service principals will be piped to retrieve the owners of all the service principals

```
Get-AzureADPrivilegedRolesMembers -ReturnServicePrincipals | Get-AzureADServicePrincipalOwner

ObjectId                             DisplayName           UserPrincipalName                   UserType
--------                             -----------           -----------------                   --------
59b28e90-d96b-410b-acc6-fa9ee823bfbd ServicePrincipalOwner ServicePrincipalOwner@jonyschats.nl Member
```

## Updating the overview cmdlet
To get a good overview again I changed the existing `Get-AzureADPrivilegedRolesOverview` cmdlet and added the SPsCount, SPs and SPsOwners attributes.

```PowerShell
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
				
				$AdminRoleMembersSPs = Get-AzureADDirectoryRole -ObjectId $AdminRoleData.ObjectId | Get-AzureADDirectoryRoleMemberRecursive -ReturnServicePrincipals
				$AdminRoleMembersSPsCount = $AdminRoleMembersSPs | Sort-Object -Unique | Measure-Object
				$ServicePrincipalOwners = Get-AzureADDirectoryRole -ObjectId $AdminRoleData.ObjectId | Get-AzureADDirectoryRoleMemberRecursive -ReturnServicePrincipals | Get-AzureADServicePrincipalOwner
				
				$item = New-Object PSObject
				$item | Add-Member -type NoteProperty -Name 'Role' -Value $AdminRoleData.DisplayName
				
				$item | Add-Member -type NoteProperty -Name 'UserCount' -Value $AdminRoleMembersUsersCount.Count
				$item | Add-Member -type NoteProperty -Name 'Users' -Value $AdminRoleMembersUsers.UserPrincipalName
				
				$item | Add-Member -type NoteProperty -Name 'GroupCount' -Value $AdminRoleMembersGroupsCount.Count
				$item | Add-Member -type NoteProperty -Name 'Groups' -Value $AdminRoleMembersGroups.DisplayName
				$item | Add-Member -type NoteProperty -Name 'GroupOwners' -Value $GroupOwners.UserPrincipalName
				
				$item | Add-Member -type NoteProperty -Name 'SPsCount' -Value $AdminRoleMembersSPsCount.Count
				$item | Add-Member -type NoteProperty -Name 'SPs' -Value $AdminRoleMembersSPs.DisplayName
				$item | Add-Member -type NoteProperty -Name 'SPsOwners' -Value $ServicePrincipalOwners.UserPrincipalName
				
				$Output += $item
			}
			else {
				$item = New-Object PSObject
				$item | Add-Member -type NoteProperty -Name 'Role' -Value $AdminRole
				$item | Add-Member -type NoteProperty -Name 'UserCount' -Value "0"
				$item | Add-Member -type NoteProperty -Name 'GroupCount' -Value "0"
				$item | Add-Member -type NoteProperty -Name 'SPsCount' -Value "0"
				$Output += $item
			}
		}
	}

	end {
        Return $Output | Sort-Object -Property UserCount -Descending
    }
}
```

The output:

```
Get-AzureADPrivilegedRolesOverview | ft

Role                                    UserCount Users                   GroupCount Groups        GroupOwners                         SPsCount SPs                SPsOwners
----                                    --------- -----                   ---------- ------        -----------                         -------- ---                ---------
Authentication Administrator                    1 GroupUser@jonyschats.nl          1 Administrator ServicePrincipalOwner@jonyschats.nl        1 Test_enterpriseapp ServicePrincipalOwner@jonyschats.nl
Global Administrator                            1 0xjs@jonyschats.nl               0                                                          0
Privileged Role Administrator                   0                                  0                                                          0
Privileged authentication administrator         0                                  0                                                          0
Password administrator                          0                                  0                                                          0
User Administrator                              0                                  0                                                          0
SharePoint administrator                        0                                  0                                                          0
Security administrator                          0                                  0                                                          0
Cloud application administrator                 0                                  0                                                          0
Billing administrator                           0                                  0                                                          0
Application administrator                       0                                  0                                                          0
Helpdesk administrator                          0                                  0                                                          0
Exchange administrator                          0                                  0                                                          0
Conditional Access administrator                0                                  0                                                          0
```

# Quick recap
A quick recap of what we have learned about Azure AD identities:
- Roles can be assigned these three identities and these identities:
	- Users
	- Groups 
		- Can't have nested groups when assigned to roles.
		- Can have owners
	- ServicePrincipals 
		- Can have Owners
		- Can have permissions

In the next blog we will dive into retrieving all the privileged identities and checking their MFA configuration.

# GitHub
All the cmdlets can be found in my GitHub project [AzurePowerCommands](https://github.com/0xJs/AzurePowerCommands).
