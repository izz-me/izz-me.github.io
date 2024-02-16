---
layout: post
title: Fixing Start Menu Issues in Windows Server 2016 and 2019
category:
- IT Troubleshooting
- Windows Server
tags: applocker gpo
date: 2024-02-16 14:30 -0700
---
Every once in a while, I run into an issue in Windows Server 2016 and 2019 where the Start Menu stops working. Right-clicking and opening the context menu will work. It's only left-clicking that won't work. Usually, this happens when either Citrix Profile Management or roaming profiles are enabled. It's difficult to correctly capture all the titles and files that make up the Start Menu in a profile. However, in this case, no profile management solution was enabled.

## Problem Identification
`Event Viewer > Windows logs > Applications` showed several `Apps` errors with Event ID 5973 related to `Microsoft.Windows.ShellExperienceHost` and `Microsoft.Windows.Cortana`. 

![cortana_shell_error](/assets/images/2024-02-16-pic1.png)
_Cortana and ShellExperienceHost were not working_

The `General` tab pointed to invalid handles and blocked processes. These processes are necessary to make the Start Menu work. Seeing the blocked error jogged my memory. That's when I remembered AppLocker.


## The Root Cause: AppLocker

Still in Event Viewer, I checked all sections under `Applications and Services logs > Microsoft > Windows > AppLocker`. Under `Packaged app-Execution`, I saw dozens of `No packaged apps can be executed while Exe rules are being enforced and Packaged app rules have been configured` errors. 

![cortana_shell_error](/assets/images/2024-02-16-pic2.png)
_AppLocker Packaged app errors_

This led me to believe that the AppLocker rules were not configured correctly or the GPOs were not being procecessed. Then I looked into `Windows logs > System`. There were several errors with Event ID 1130. One of the entries pointed to stale file errors related to the AppLocker startup script. Jackpot!

## Resolution Steps
The solution involved several steps to restore Start Menu functionality:

1. **Local Group Policy Test:** First, I tested with a Local Group Policy by launching gpedit.msc and navigating to `Computer > Application Control Policies > AppLocker`. Right-clicked Packaged app Rules, and then **Create Default Rules**. In this case, I looked at the default rules created and decided that they were good enough. A gpupdate /force command in CMD immediately fixed the Start Menu.

> Per Microsoft:
> You can use the default rules as a template when creating your own rules to allow files within the Windows folders to run. However, these rules are only meant to function as a starter policy when you are first testing AppLocker rules. The default rules can be modified in the same way as other AppLocker rule types.
{: .prompt-info }


1. **Fixed Stale GPO Files:** Corrected stale files contained in the GPO settings.

2. **Domain-Level GPO Adjustments:** Finally, the edits made to the Local machine were applied to the domain GPO.

## Conclusion
This process not only highlights the importance of correct AppLocker configurations but also the need for regular checks on GPO file integrity. For administrators, these insights are vital for maintaining a seamless user experience on Windows Server platforms.

> Before implementing any script or code in a production environment, always perform thorough testing in a controlled and non-critical setting. Understand that errors or unexpected behavior may occur, so validate and adapt the code to your specific use case.
{: .prompt-info}
