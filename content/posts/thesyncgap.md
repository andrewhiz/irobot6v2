---
date: '2025-06-11T14:43:33-07:00'
draft: false
title: 'The sync gap'
---


# 🔄 Why Mobile Numbers Aren't Syncing from AD to Entra ID

> **TL;DR**: Your AD mobile numbers aren't updating in Entra ID Hybrid environment? It's likely that `BypassDirSyncOverridesEnabled` is not what you expect it to be.

---

## 🚨 The Problem

In hybrid Microsoft Entra ID environments, admins are **surprised** to find that mobile numbers in Active Directory don't sync to Entra for **some users** — even though all other attributes do perfectly. You also don't see any signs of failure on Entra Connect log.


**What you see:**
```
Active Directory          Entra ID
┌─────────────────┐      ┌─────────────────┐
│ 📱 +1234567890  │ ───► │ 📱 +1219999999  │
│ 👤 John Doe     │ ───► │ 👤 John Doe     │ ✅
│ 📧 john@co.com  │ ───► │ 📧 john@co.com  │ ✅
└─────────────────┘      └─────────────────┘
```

**The Impact:**
- 🔍 **Inconsistent data between EntraID and Local AD**
- 🔐 **MFA setup fails** (no or incorrect mobile number)
- 🔒 **SSPR doesn't work** (missing recovery method)  
- 📞 **Teams or outlook contact data incorrect**

---

## 🧠 Root Cause: BypassDirSyncOverridesEnabled (or is it)

The culprit is a tenant setting called **`BypassDirSyncOverridesEnabled`**:

| Setting Value | Behavior |
|---------------|----------|
| `false` (default) | 🚫 **Blocks** AD mobile sync - Entra keeps cloud values |
| `True` | ✅ **Allows** AD mobile sync - Traditional behavior |

This setting specifically impacts:
- 📱 `mobile` → `MobilePhone`
- 📱 `otherMobile` → `AlternateMobilePhones`

Note: `false` setting above is a bit mis-leading as Entra Connect will continue to sync Mobile number number from Local AD to EntraID, for majority of your users.

---

## 📅 When Did This Change?

Prior to mid-2023, Entra ID allowed updates to certain user object fields (like Mobile) for hybrid users. Users were able to update on their my profile page on Entra and Admins were also able to update on EntraID, without touching Local AD.

Later, Microsoft introduced `BypassDirSyncOverridesEnabled` setting with-in Entra Connect, that allowed organization to control this behavior. 

**Why admins are caught off-guard:**
- 🤐 **Silent rollout** - No widespread communication
- 🔍 **PowerShell-only** - Not visible in admin portals  
- 🎯 **Selective blocking** - Only mobile attributes affected
- ❌ **No error messages** - Sync appears to work fine and continued to sync upto EntraID (with a caveat)

---

## The REAL issue IMO

- **No information logged on Entra Connect.**
-   `BypassDirSyncOverridesEnabled` setting is Disabled, yet Mobile numbers are syncing as usual for **most users*
-   User that the mobile numbers dont sync, all other attributed are syncing

### My theory is: 
 - If you have users that had their mobile number updated, while it was allowed, EntraID value will be unchanged if  `BypassDirSyncOverridesEnabled` is FALSE
 - For user accounts where Entra Mobile fields were never directly updated, Entra Connect sync will continue to update - regardless of `BypassDirSyncOverridesEnabled` is TRUE or FALSE

If you are a traditional organization on hybrid, where you treat Local AD as your "source of truth", you want to ensure that Local AD values are synced to Entra ID. If that is the case, you probably want this setting to be Enabled, so you could continue to sync mobile number values from Local AD to Entra, for all users.

## Solution summary
1. **Status Check >** Identify the current status of `BypassDirSyncOverridesEnabled`
2. **Evaluate the difference >** Identify the user objects that has a different value (there is a command for that). 
3. **The Fix >** Enable the `BypassDirSyncOverridesEnabled` set to TRUE
4. **Sync >** Perform a Full Entra Connect Sync


## 🔍 1. Status Check

```powershell
# Connect to Microsoft Graph
Connect-MgGraph -Scopes "Directory.Read.All"

# Ensure to install ADSyncTools Module
Install-Module ADSyncTools

# Check current status
(Get-MgDirectoryOnPremiseSynchronization).Features.BypassDirSyncOverridesEnabled
```

**Result:**
- `True` = 🚫 Mobile sync blocked (but may be working with a caveat)
- `False` = ✅ Mobile sync enabled

---

## 📊 2. Evaluate the difference

Find users that have different mobile numbers between Loca AD vs EntraID, using the ADSyncTools Module. This is helpful in the event you need to capture that data to be updated to Local AD.

```powershell
# Get sync gaps between AD and Entra ID
Compare-ADSyncToolsDirSyncOverrides
```

---

## ⚡ 3. The Fix

To restore traditional mobile number synchronization:

```powershell
# Connect with write permissions
Connect-MgGraph -Scopes "Directory.ReadWrite.All"

# Disable the bypass (enable AD sync)
$directorySynchronization = Get-MgDirectoryOnPremiseSynchronization
$directorySynchronization.Features.BypassDirSyncOverridesEnabled = $true
Update-MgDirectoryOnPremiseSynchronization -OnPremisesDirectorySynchronizationId $directorySynchronization.Id -Features $directorySynchronization.Features

```

## 🔄 4. Sync
Start a Full Sync cycle in MS Entra Connect (on your AD Connect server)

```powershell
Start-ADSyncSyncCycle -PolicyType Initial

```

---

## 📚 Official Microsoft Resources

- **Primary Documentation**: [How to use the BypassDirSyncOverridesEnabled feature](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-bypassdirsyncoverrides)
- **Attribute Mapping**: [Attributes synchronized by Microsoft Entra Connect](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/reference-connect-sync-attributes-synchronized)  
- **Graph API Reference**: [onPremisesDirectorySynchronizationFeature resource](https://learn.microsoft.com/en-us/graph/api/resources/onpremisesdirectorysynchronizationfeature)

---
