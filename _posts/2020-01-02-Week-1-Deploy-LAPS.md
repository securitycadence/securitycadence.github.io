---
layout: posts
title: "Week 1: Deploy LAPS"
date: 2020-01-02
tags: windows passwords
---

## What?

Microsoft LAPS is a free tool from Microsoft for automatically generating and setting random passwords
to Windows Local Administrator accounts. The passwords are stored in a protected Active Directory attribute
of the computer object. You can control who has the ability to access the passwords. The passwords are randomly generated based on complexity requirements that you supply and they are rotated on a specified schedule.  
This gives you an easy, free, and fully supported method to have unique local administrator passwords
on all of your Windows endpoints.

## Why?

Without an automated mechanism for setting local administrator passwords, organizations are left with little
control over these important accounts. It is not uncommon to find organizations that have the same local
administrator password set on all end points. Further these passwords are rarely, if ever changed.  
This creates an environment that makes for easy lateral movement from endpoint to endpoint. An attacker
gaining access to this single account is in a position to move from system to system with full administrator
rights. This is also at high risk of pass-the-hash attacks.

## How?

1. Download LAPS  

LAPS is available in 64bit and 32bit versions and is supported on Windows 7 through Server 2019.  You will need a management computer, which is often a Domain Controller.  The software can be downloaded here: [https://www.microsoft.com/en-us/download/details.aspx?id=46899](https://www.microsoft.com/en-us/download/details.aspx?id=46899)

2. Management Installation

On the management system launch the installer. Choose which features you wish to install when prompted:
...AdmPWD GPO Extension: GPO extension used by LAPS for changing the local administrator password. This portion of the MSI will need to be installed on all endpoints where you wish to change the password.
...Fat client UI: Simple GUI interface for viewing passwords. This is honestly not super useful as you will be able to view passwords either with Powershell or via the Active Directory Users and Computers MMC.
...PowerShell module: Installs the Get-AdmPwdPassword commandlet for viewing passwords
...GPO editor templates: ADMX templates for configuring LAPS GPOs

3. Client Installation

The LAPS msi defaults to installing just the GPO extension when launched silently.  As such, deployment of LAPS can be as simple as a GPO that executes:
`msiexec /i <file location>\LAPS.x64.msi /quiet`

4. Extend the Active Directory Schema to support LAPS.

LAPS requires the following attributes to be added to the AD Schema for computer objects in order to function:

...ms-Mcs-AdmPwd: Stores the password in clear text
...ms-Mcs-AdmPwdExpirationTime: Stores the time to reset the password

To extend the schema, launch Powershell on your management server with a user account that is a member of the Schema Admins AD group.  Run the following commands:

`import-module AdmPwd.ps`
`Update-AdmPwdADSchema`

Next we need to grant the computer objects permissions to set their own password and expiration attributes:

`Set-AdmPwdComputerSelfPermission -OrgUnit <OU containing computer objects>`

Repeat the above for all relevant OUs in your AD structure.

5. Ensure that only the proper users can view local admin passwords

Next in Powershell we will take a look at who has the abilitity to view the ms-Mcs-AdmPwd attribute in Active Directory.  By default Domain and Enterprise admins will have access to view the passwords. Issue the following command:

`Find-AdmPwdExtendedRights -OrgUnit <OU containing computer objects>`

The output of this command will likely only show NT Authority\System and Domain Admins. If there are users/groups in this output that should not have permission to see the stored passwords, then they need to have the "All extended rights" permission removed from the OU in question. This website has a good writeup on how to remove these permissions [https://4sysops.com/archives/set-up-microsoft-laps-local-administrator-password-solution-in-active-directory/#update-user-group-ad-permissions](https://4sysops.com/archives/set-up-microsoft-laps-local-administrator-password-solution-in-active-directory/#update-user-group-ad-permissions)

You can use the Set-AdmPwdReadPasswordPermission cmdlet to provide additional users/groups the ability to view passwords:

`Set-AdmPwdReadPasswordPermission -OrgUnit <OU containing computer objects> -AllowedPrincipals <Name of group to delegate permissions to>`

6. Setup the LAPS GPO

Finally we are ready to configure a basic group policy for configuring LAPS. On the your management system load the Group Policy Management Console. You will want to create a new GPO or modify an existing GPO that is linked to your computer OU.

Under Computer Configurstion -> Policies -> Administrative Templates you should now have a section for LAPS:

![LAPS GPO Location](https://securitycadence.github.io/imgs/2020-01-02/GPO-LAPS-1.png)

Here you will find four different settings:

...Password Settings: Allows you to configure the complexity an dage of the passwords that LAPS sets.
...Name of administrator account to manage: Name of the local administrator account that you wish to mange. Only use this setting if you are using an account other than the built-in (RID 500) local admin account, even if you've renamed the built-in local admin account.
...Do not allow password expiration time longer than required by policy: When enabled the password of a local administrator account is changed immediately when it's password has expired.
...Enable local admin password management: Enables LAPS for the OU

7. Viewing PasswordState

In order to view a password set by LAPS, you will need an account that has been granted permissions to view the ms-Mcs-AdmPwd attribute.  There are three basic ways to view the password:

...Powershell: Use the Get-AdmPwdPassword cmdlet:
`Get-AdmPwdPassword -ComputerName "myWorkstation"

...LAPS Fat Client: From your management server launch C:\program files\LAPS\AdmPwd.UI.  This will execute a basic program where you can type in the computer name you wish to retrieve the local admin password for.

...Active Directory Users and Computers: Enable the viewing of Advanced Features in the ADUC MMC snap-in which will then expose the Attribute Editor tab when you view a computer object's properties. From here you can scroll down to the ms-Mcs-AdmPwd attribute to view its value.

## Gotchas?

1. If your organization has disabled the built-in (RID 500) local administrator account and replaced it with
an alternative you should know that this creates a risk. While LAPS can target a uniquely named Local administrator
account, it will lose track of it if the account is renamed. It is for this reason that Microsoft recommends
that organizations make exclusive use of the built-in Administrator account. When using the built-in account LAPS
will target RID 500 instead of the Administrator username.  Note that you can still rename the Administrator
account if you should so choose. This will not impact LAPS.  
2. Organizations should understand that LAPS does store the local administrator password in Active Directory in
clear text. Access to the passwords are controlled via ACLs in Active Directory. While this may seem like a
poor design, one should keep in mind that if an attacker manages to gain the necessary privileges to access these
Active Directory attributes, then they are very likely already Domain Admin and therefore no longer need to
worry about local administrator accounts.  
3. There is a chance that compliance might complain about the passwords being stored in clear text. This setup is
technically against PCI requirements, for example. However, this author has never heard of an organization
having issues with PCI auditors barking over LAPS. Compensating controls of ACLS combined with the obviously
stronger footprint over password reuse should appease any auditor.  
4. You can potentially run into issues with LAPS and technologies such as snapshotting where a virtual machine
has a snapshot applied, LAPS changes the password, and the virtual machine is reverted. In this scenario the
local administrator password will be lost until the next LAPS password change. The chances of this having an
impact are relatively low, as domain accounts will still have access to the system.

## Additional Thoughts And External Links

If you are concerned about storing passwords in clear text and/or wish to control passwords for non-Windows
accounts, there are several excellent options out there:

* [SHIPS](https://www.trustedsec.com/tools/ships/)
* [PasswordState](https://www.clickstudios.com.au/)
* [SecretServer](https://www.thycotic.com)

More information regarding LAPS:  

* [4SysOps LAPS Introduction](https://4sysops.com/archives/introduction-to-microsoft-laps-local-administrator-password-solution/)
* [Download LAPS](https://www.microsoft.com/en-us/download/details.aspx?id=46899)
* [Microsoft LAPS Deployment Step-By-Step Guide](https://gallery.technet.microsoft.com/Step-by-Step-Deploy-Local-7c9ef772/file/150657/1/Step%20by%20Step%20Guide%20to%20Deploy%20Microsoft%20LAPS.pdf)
