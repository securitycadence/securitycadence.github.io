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

##How?

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

* [Download LAPS](https://www.microsoft.com/en-us/download/details.aspx?id=46899)
* [Microsoft LAPS Deployment Step-By-Step Guide](https://gallery.technet.microsoft.com/Step-by-Step-Deploy-Local-7c9ef772/file/150657/1/Step%20by%20Step%20Guide%20to%20Deploy%20Microsoft%20LAPS.pdf)
