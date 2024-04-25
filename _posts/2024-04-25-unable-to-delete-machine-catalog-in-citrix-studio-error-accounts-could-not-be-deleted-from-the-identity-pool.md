---
layout: post
title: 'Unable to delete Machine Catalog in Citrix Studio. Error: Accounts could not
  be deleted from the identity pool.'
category:
- IT Troubleshooting
- Citrix Studio
tags:
- citrix
- MCS
date: 2024-04-25 13:22 -0600
---
I recently ran across this error when attempting to delete an MCS Machine Catalog from Citrix Studio.

`Accounts could not be deleted from the identity pool. Note that this result can occur if you do not have the required Active Directory permissions.`

![ad_account_error](/assets/images/2024-04-25-pic1.png)

When I clicked on `View Details`, I saw the PowerShell commands Citrix Studio was using, along with the SIDs for each problematic account. The line will look something like this:

```powershell
Remove-AcctADAccount -ADAccountSid @("S-0-0-00-000000000-0000000000-00000") -AdminAddress "testdc01" -BearerToken ********* -Force -IdentityPoolUid "a00000a00000a0000 a00000a00000a0000"
```

Then I remembered messing around with this Machine Catalog using PowerShell a while back, trying to fix some naming convention quirks with MCS. That could've been the culprit.

So, I ran this command to check things out:

```powershell
Get-AcctADAccount -state Available -AdminAddress "testdc01"
```

Turned out, the SIDs in the error matched the ones in the output, and all the SIDs in the error had their `Lock` property set to `True`. Note: Don’t get confused with the `AccountLocked` property.

![account_properties](/assets/images/2024-04-25-pic2.png)

I unlocked each SID using this command:

```powershell
Unlock-AcctADAccount -ADAccountSid S-0-0-00-000000000-0000000000-00000
```

Once all the accounts were unlocked, I was finally able to delete the Machine Catalog.
