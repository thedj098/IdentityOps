# IdentityOps Module Documentation

**Version:** 1.0.1  
**Author:** IdentityOps Contributors  
**License:** MIT  
**PowerShell:** 7.2+  
**Dependency:** Microsoft.Graph.Authentication 2.0.0+ (auto-installed)
https://www.powershellgallery.com/packages/IdentityOps/1.0.1
---

## Table of Contents

1. [Installation](#installation)
2. [Authentication](#authentication)
   - [Interactive (Browser)](#1-interactive-browser--default)
   - [Device Code](#2-device-code)
   - [Client Secret](#3-client-secret)
   - [Certificate Thumbprint](#4-certificate-thumbprint)
   - [Certificate File](#5-certificate-file)
   - [System Managed Identity](#6-system-managed-identity)
   - [User-Assigned Managed Identity](#7-user-assigned-managed-identity)
   - [Access Token](#8-access-token)
3. [Required Permissions](#required-permissions)
4. [Commands Reference](#commands-reference)
   - [Connect-IdentityOps](#connect-identityops)
   - [Disconnect-IdentityOps](#disconnect-identityops)
   - [Get-IOExpiringSecrets](#get-ioexpiringsecrets)
   - [Get-IOExpiringSSOCerts](#get-ioexpiringssoerts)
   - [Get-IOStaleGuests](#get-iostaleguests)
   - [Get-IOOrphanedApps](#get-ioorphanedapps)
   - [Get-IOOverPrivilegedApps](#get-iooverprivilegedapps)
   - [Get-IOUsersWithoutMFA](#get-iouserswithoutmfa)
   - [Get-IODormantApps](#get-iodormantapps)
   - [Get-IOOrphanedRoleAssignments](#get-ioorphanedroleassignments)
   - [Get-IOOwnerlessGroups](#get-ioownerlessgroups)
   - [Get-IOPendingAdminConsent](#get-iopendingadminconsent)
   - [Get-IOServicePrincipalSignInFailures](#get-ioserviceprincipalsigninfailures)
   - [Get-IOPasswordOnlyAccounts](#get-iopasswordonlyaccounts)
   - [Get-IOCrossTenantAccessReport](#get-iocrosstenantaccessreport)
   - [Export-IOConditionalAccessReport](#export-ioconditionalaccessreport)
   - [Grant-IOManagedIdentityPermission](#grant-iomanagedidentitypermission)
5. [CSV Export](#csv-export)
6. [Error Handling & Resilience](#error-handling--resilience)
7. [Security Features](#security-features)
8. [License Requirements](#license-requirements)
9. [Troubleshooting](#troubleshooting)

---

## Installation

### From PowerShell Gallery

```powershell
Install-Module -Name IdentityOps -Repository PSGallery -Scope CurrentUser
```

`Microsoft.Graph.Authentication` is automatically installed as a dependency — no separate install needed.

### Update to Latest Version

```powershell
Update-Module -Name IdentityOps
```

### Verify Installation

```powershell
Get-Module -Name IdentityOps -ListAvailable
Get-Command -Module IdentityOps
```

---

## Authentication

IdentityOps supports **8 authentication flows**. Choose the right one based on your environment.

### 1. Interactive (Browser) — Default

Opens a browser window for sign-in. Best for admin workstations.

```powershell
Connect-IdentityOps
```

### 2. Device Code

Displays a code to enter at https://microsoft.com/devicelogin. Best for SSH sessions, headless servers, or remote terminals.

```powershell
Connect-IdentityOps -DeviceCode
```

### 3. Client Secret

App-only authentication using a client secret. The secret **must** be a `SecureString`.

```powershell
$secret = Read-Host "Enter client secret" -AsSecureString
Connect-IdentityOps -TenantId "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" `
                    -ClientId "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" `
                    -ClientSecret $secret
```

### 4. Certificate Thumbprint

App-only authentication using a certificate installed in the local cert store.

```powershell
Connect-IdentityOps -TenantId "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" `
                    -ClientId "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" `
                    -CertificateThumbprint "AB12CD34EF56..."
```

> The thumbprint must be a 40-character hex string.

### 5. Certificate File

App-only authentication using a `.pfx` certificate file on disk. The private key password must be a `SecureString`.

```powershell
$certPwd = Read-Host "Enter cert password" -AsSecureString
Connect-IdentityOps -TenantId "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" `
                    -ClientId "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" `
                    -CertificatePath "C:\certs\app.pfx" `
                    -CertificatePassword $certPwd
```

> The certificate is loaded with `EphemeralKeySet` and disposed after use. The BSTR password is zero-freed from memory.

### 6. System Managed Identity

For workloads running in Azure (VMs, App Service, Functions, etc.) with a system-assigned managed identity.

```powershell
Connect-IdentityOps -ManagedIdentity
```

### 7. User-Assigned Managed Identity

For Azure workloads with a user-assigned managed identity.

```powershell
Connect-IdentityOps -ManagedIdentity `
                    -ClientId "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
```

### 8. Access Token

Use a pre-acquired OAuth2 access token. Must be a `SecureString`.

```powershell
$token = ConvertTo-SecureString "eyJ0eX..." -AsPlainText -Force
Connect-IdentityOps -AccessToken $token
```

### Common Parameters

| Parameter   | Type       | Description                                                    |
|-------------|------------|----------------------------------------------------------------|
| `-Scopes`   | `string[]` | Override the default permission scopes for delegated flows     |
| `-NoBanner` | `switch`   | Suppress the ASCII art welcome banner on connection            |

### Disconnecting

```powershell
Disconnect-IdentityOps
```

Clears the Microsoft Graph session and all module-scoped connection state.

---

## Required Permissions

### Delegated (Interactive / Device Code)

The module requests these scopes by default:

| Scope                                   | Used By                                                    |
|-----------------------------------------|------------------------------------------------------------|
| `Application.Read.All`                  | Expiring secrets, orphaned apps, dormant apps              |
| `AuditLog.Read.All`                     | SP sign-in failures, password-only accounts                |
| `Directory.Read.All`                    | Role assignments, cross-tenant, stale guests               |
| `Group.Read.All`                        | Ownerless groups                                           |
| `Policy.Read.All`                       | Conditional Access, cross-tenant policies                  |
| `RoleManagement.Read.Directory`         | Orphaned role assignments, admin MFA checks                |
| `User.Read.All`                         | Users without MFA, stale guests                            |
| `UserAuthenticationMethod.Read.All`     | MFA method checks, password-only detection                 |
| `CrossTenantInformation.ReadBasic.All`  | Cross-tenant access report                                 |

### Application (Client Secret / Certificate / Managed Identity)

For app-only flows, grant these as **Application permissions** in the Entra ID app registration, then admin-consent them.

> **For Grant-IOManagedIdentityPermission**, you additionally need `AppRoleAssignment.ReadWrite.All` to assign permissions.

---

## Commands Reference

---

### Connect-IdentityOps

Connects to Microsoft Graph. Must be called before any other command.

| Parameter               | Type           | Required | Description                                              |
|-------------------------|----------------|----------|----------------------------------------------------------|
| `-Interactive`          | `switch`       | No       | Use browser sign-in (default)                            |
| `-DeviceCode`           | `switch`       | Yes*     | Use device code flow                                     |
| `-TenantId`             | `string`       | Yes*     | Tenant GUID (required for app-only flows)                |
| `-ClientId`             | `string`       | Yes*     | App registration GUID                                    |
| `-ClientSecret`         | `SecureString` | Yes*     | App client secret                                        |
| `-CertificateThumbprint`| `string`       | Yes*     | 40-char hex certificate thumbprint                       |
| `-CertificatePath`      | `string`       | Yes*     | Path to .pfx certificate file                            |
| `-CertificatePassword`  | `SecureString` | No       | Password for the .pfx file                               |
| `-ManagedIdentity`      | `switch`       | Yes*     | Use managed identity (system or user-assigned)           |
| `-AccessToken`          | `SecureString` | Yes*     | Pre-acquired OAuth2 token                                |
| `-Scopes`               | `string[]`     | No       | Custom scopes (overrides defaults)                       |
| `-NoBanner`             | `switch`       | No       | Suppress welcome banner                                  |

*Required within their parameter set (auth flow).

**Validation:**
- `TenantId` and `ClientId` must be valid GUIDs
- `CertificateThumbprint` must be exactly 40 hex characters
- `CertificatePath` must point to an existing file
- `ClientSecret` and `AccessToken` accept only `SecureString` (never plain text)

---

### Disconnect-IdentityOps

Disconnects from Microsoft Graph and clears all session state.

```powershell
Disconnect-IdentityOps
```

No parameters.

---

### Get-IOExpiringSecrets

Finds app registrations with client secrets or certificates expiring within a specified window.

| Parameter          | Type     | Default | Description                                       |
|--------------------|----------|---------|---------------------------------------------------|
| `-DaysUntilExpiry` | `int`    | `30`    | Number of days ahead to check (1–3650)            |
| `-IncludeExpired`  | `switch` | —       | Also include already-expired credentials          |
| `-ToCsv`           | `string` | —       | Export results to a CSV file                      |

**Output columns:** ApplicationName, ApplicationId, ObjectId, CredentialType (ClientSecret/Certificate), CredentialName, KeyId, ExpiryDate, DaysRemaining, Status (EXPIRED/EXPIRING)

**Examples:**

```powershell
# Secrets expiring in next 30 days (default)
Get-IOExpiringSecrets

# Secrets expiring in next 90 days, including already expired
Get-IOExpiringSecrets -DaysUntilExpiry 90 -IncludeExpired

# Export to CSV
Get-IOExpiringSecrets -DaysUntilExpiry 60 -ToCsv "expiring-secrets.csv"
```

---

### Get-IOExpiringSSOCerts

Scans enterprise apps (service principals) for SAML/SSO signing certificates that are expiring or expired.

| Parameter          | Type     | Default | Description                                       |
|--------------------|----------|---------|---------------------------------------------------|
| `-DaysUntilExpiry` | `int`    | `30`    | Number of days ahead to check (1–3650)            |
| `-IncludeExpired`  | `switch` | —       | Also include already-expired certificates         |
| `-ToCsv`           | `string` | —       | Export results to a CSV file                      |

**Output columns:** ApplicationName, ApplicationId, ObjectId, SSOMode, CredentialType, Usage, KeyId, ExpiryDate, DaysRemaining, Status (EXPIRED/EXPIRING)

**Examples:**

```powershell
# SSO certs expiring in next 60 days
Get-IOExpiringSSOCerts -DaysUntilExpiry 60

# Include expired, export to CSV
Get-IOExpiringSSOCerts -DaysUntilExpiry 30 -IncludeExpired -ToCsv "sso-certs.csv"
```

---

### Get-IOStaleGuests

Lists guest users who haven't signed in for a specified number of days.

| Parameter              | Type     | Default | Description                                         |
|------------------------|----------|---------|-----------------------------------------------------|
| `-InactiveDays`        | `int`    | `90`    | Days of inactivity threshold (1–3650)               |
| `-IncludeNeverSignedIn`| `switch` | —       | Include guests who have never signed in             |
| `-ToCsv`               | `string` | —       | Export results to a CSV file                        |

**Output columns:** DisplayName, UserPrincipalName, Mail, ObjectId, AccountEnabled, CreatedDate, LastSignIn, InactiveDays, Status (INACTIVE/NEVER_SIGNED_IN)

**Examples:**

```powershell
# Guests inactive for 90+ days
Get-IOStaleGuests

# Guests inactive 180+ days or never signed in
Get-IOStaleGuests -InactiveDays 180 -IncludeNeverSignedIn

# Export to CSV
Get-IOStaleGuests -InactiveDays 90 -IncludeNeverSignedIn -ToCsv "stale-guests.csv"
```

---

### Get-IOOrphanedApps

Finds app registrations (and optionally enterprise apps) with no owners assigned.

| Parameter              | Type     | Default | Description                                         |
|------------------------|----------|---------|-----------------------------------------------------|
| `-IncludeEnterpriseApps`| `switch`| —       | Also scan service principals for missing owners     |
| `-ToCsv`               | `string` | —       | Export results to a CSV file                        |

**Output columns:** DisplayName, AppId, ObjectId, Type (AppRegistration/EnterpriseApp), CreatedDate, OwnerCount

**Examples:**

```powershell
# App registrations with no owners
Get-IOOrphanedApps

# Include enterprise apps too
Get-IOOrphanedApps -IncludeEnterpriseApps

# Export to CSV
Get-IOOrphanedApps -IncludeEnterpriseApps -ToCsv "orphaned-apps.csv"
```

---

### Get-IOOverPrivilegedApps

Flags applications that have been granted high-privilege Microsoft Graph permissions.

| Parameter                  | Type       | Default                        | Description                                             |
|----------------------------|------------|--------------------------------|---------------------------------------------------------|
| `-CustomHighPrivilegeRoles`| `string[]` | Built-in list (see below)     | Override the default list of dangerous permissions       |
| `-ToCsv`                  | `string`   | —                              | Export results to a CSV file                            |

**Default high-privilege permissions detected:**

| Permission                            | Risk Level |
|---------------------------------------|------------|
| `Directory.ReadWrite.All`             | Critical   |
| `RoleManagement.ReadWrite.Directory`  | Critical   |
| `Application.ReadWrite.All`          | Critical   |
| `AppRoleAssignment.ReadWrite.All`    | Critical   |
| `Mail.ReadWrite`                      | High       |
| `Mail.Send`                          | High       |
| `Files.ReadWrite.All`                | Critical   |
| `Sites.FullControl.All`              | Critical   |
| `User.ReadWrite.All`                 | Critical   |
| `Group.ReadWrite.All`                | Critical   |
| `Calendars.ReadWrite`                | High       |
| `Contacts.ReadWrite`                 | High       |
| `MailboxSettings.ReadWrite`          | High       |
| `Chat.ReadWrite.All`                 | High       |
| `TeamSettings.ReadWrite.All`         | High       |
| `Policy.ReadWrite.ConditionalAccess` | Critical   |

**Output columns:** ApplicationName, ApplicationId, ObjectId, Permission, ResourceApp, RiskLevel (Critical/High)

**Examples:**

```powershell
# Scan with built-in list
Get-IOOverPrivilegedApps

# Scan with custom dangerous permissions
Get-IOOverPrivilegedApps -CustomHighPrivilegeRoles @('Mail.Send', 'Files.ReadWrite.All')

# Export to CSV
Get-IOOverPrivilegedApps -ToCsv "over-privileged.csv"
```

---

### Get-IOUsersWithoutMFA

Lists users who have no MFA authentication methods registered (password-only).

| Parameter        | Type     | Default | Description                                             |
|------------------|----------|---------|-------------------------------------------------------|
| `-IncludeGuests` | `switch` | —       | Include guest users (members only by default)          |
| `-AdminsOnly`    | `switch` | —       | Only check users who hold directory admin roles        |
| `-BatchSize`     | `int`    | `5`     | Users per batch (1–20). Lower = less throttling risk   |
| `-ToCsv`         | `string` | —       | Export results to a CSV file                           |

**Output columns:** DisplayName, UserPrincipalName, ObjectId, UserType, RegisteredMethods, MFAMethodCount, Risk (Critical for admins, High for others)

**Features:**
- Skips disabled accounts automatically
- Processes in configurable batches to avoid Graph API throttling
- Circuit breaker: stops if >10% of user checks fail (likely a permissions issue)
- Throttle-friendly 200ms delay between batches for large tenants (100+ users)

**Examples:**

```powershell
# Members without MFA
Get-IOUsersWithoutMFA

# Admin users without MFA (critical risk)
Get-IOUsersWithoutMFA -AdminsOnly

# All users including guests, larger batches
Get-IOUsersWithoutMFA -IncludeGuests -BatchSize 10

# Export to CSV
Get-IOUsersWithoutMFA -AdminsOnly -ToCsv "admins-no-mfa.csv"
```

---

### Get-IODormantApps

Finds service principals (enterprise apps) with no recent sign-in activity.

| Parameter       | Type     | Default | Description                                          |
|-----------------|----------|---------|------------------------------------------------------|
| `-InactiveDays` | `int`    | `90`    | Days of inactivity threshold (1–3650)                |
| `-ToCsv`        | `string` | —       | Export results to a CSV file                         |

**Output columns:** ApplicationName, ApplicationId, ObjectId, CreatedDate, LastActivity, InactiveDays, Status (DORMANT/NEVER_USED)

> Requires **Entra ID P1/P2** license for `signInActivity` data on service principals.

**Examples:**

```powershell
# Apps inactive for 90+ days
Get-IODormantApps

# Apps inactive for 180+ days
Get-IODormantApps -InactiveDays 180

# Export to CSV
Get-IODormantApps -InactiveDays 90 -ToCsv "dormant-apps.csv"
```

---

### Get-IOOrphanedRoleAssignments

Detects directory role assignments pointing at disabled or deleted users.

| Parameter | Type     | Default | Description                              |
|-----------|----------|---------|------------------------------------------|
| `-ToCsv`  | `string` | —       | Export results to a CSV file             |

**Output columns:** RoleName, RoleId, MemberName, MemberUPN, MemberObjectId, MemberStatus (Deleted/Disabled), MemberType

**Examples:**

```powershell
# Find orphaned role assignments
Get-IOOrphanedRoleAssignments

# Export to CSV
Get-IOOrphanedRoleAssignments -ToCsv "orphaned-roles.csv"
```

---

### Get-IOOwnerlessGroups

Finds Microsoft 365 and Security groups with no owners assigned.

| Parameter    | Type     | Default | Description                                             |
|--------------|----------|---------|-------------------------------------------------------|
| `-GroupType` | `string` | `All`   | Filter by group type: `All`, `M365`, `Security`, `MailEnabled` |
| `-ToCsv`     | `string` | —       | Export results to a CSV file                           |

**Output columns:** GroupName, GroupId, GroupType (Microsoft365/Security/MailEnabled/Other), Mail, IsDynamic, CreatedDate, OwnerCount

**Examples:**

```powershell
# All groups without owners
Get-IOOwnerlessGroups

# Only Microsoft 365 groups
Get-IOOwnerlessGroups -GroupType M365

# Only Security groups, export to CSV
Get-IOOwnerlessGroups -GroupType Security -ToCsv "ownerless-groups.csv"
```

---

### Get-IOPendingAdminConsent

Lists applications waiting for admin consent approval.

| Parameter | Type     | Default | Description                              |
|-----------|----------|---------|------------------------------------------|
| `-ToCsv`  | `string` | —       | Export results to a CSV file             |

**Output columns:** AppDisplayName, AppId, ConsentRequestId, UserRequestId, RequestStatus, RequestedBy, RequestedDate

> Requires the admin consent workflow to be enabled in your tenant. If not configured, the command will display a warning and return gracefully.

**Examples:**

```powershell
# View pending consent requests
Get-IOPendingAdminConsent

# Export to CSV
Get-IOPendingAdminConsent -ToCsv "pending-consent.csv"
```

---

### Get-IOServicePrincipalSignInFailures

Retrieves recent sign-in failures for service principals from audit logs.

| Parameter         | Type     | Default | Description                                          |
|-------------------|----------|---------|------------------------------------------------------|
| `-Days`           | `int`    | `7`     | Lookback period in days (1–30)                       |
| `-AppDisplayName` | `string` | —       | Filter to a specific application by display name     |
| `-ToCsv`          | `string` | —       | Export results to a CSV file                         |

**Output columns:** AppDisplayName, AppId, ServicePrincipal, SignInDateTime, ErrorCode, FailureReason, AdditionalDetails, IPAddress, ResourceApp, City, Country

> Requires **Entra ID P1/P2** license and `AuditLog.Read.All` permission.

**Examples:**

```powershell
# Last 7 days of SP sign-in failures
Get-IOServicePrincipalSignInFailures

# Last 30 days for a specific app
Get-IOServicePrincipalSignInFailures -Days 30 -AppDisplayName "My API App"

# Export to CSV
Get-IOServicePrincipalSignInFailures -Days 14 -ToCsv "sp-failures.csv"
```

---

### Get-IOPasswordOnlyAccounts

Identifies accounts that perform non-interactive sign-ins using only passwords — potential service accounts that should be migrated to certificate or managed identity authentication.

| Parameter       | Type     | Default | Description                                          |
|-----------------|----------|---------|------------------------------------------------------|
| `-LookbackDays` | `int`    | `30`    | How many days of sign-in history to analyze (1–365)  |
| `-ToCsv`        | `string` | —       | Export results to a CSV file                         |

**Output columns:** DisplayName, UserPrincipalName, UserId, NonInteractiveApps (up to 5), SignInCount, RegisteredMethods, Risk (Critical if >50 sign-ins, else High), Recommendation

> Requires **Entra ID P1/P2** license and `AuditLog.Read.All` permission.

**Examples:**

```powershell
# Default 30-day lookback
Get-IOPasswordOnlyAccounts

# 90-day lookback, export to CSV
Get-IOPasswordOnlyAccounts -LookbackDays 90 -ToCsv "password-only.csv"
```

---

### Get-IOCrossTenantAccessReport

Summarizes cross-tenant access policy settings and flags overly permissive configurations.

| Parameter | Type     | Default | Description                              |
|-----------|----------|---------|------------------------------------------|
| `-ToCsv`  | `string` | —       | Export results to a CSV file             |

**Output columns:** PartnerTenant, PartnerTenantId, Direction (Default Policy/Partner-Specific), B2BInbound, B2BOutbound, DirectConnect, InboundTrust, Risks, RiskCount

**Detected risks:**
- `InboundB2B_AllApps` — Default policy allows B2B collaboration inbound to all apps
- `InboundDirectConnect_AllApps` — Default policy allows direct connect to all apps
- `B2BInbound_AllApps` — Partner-specific policy allows inbound to all apps
- `B2BInbound_AllUsers` — Partner-specific policy allows all users inbound

**Examples:**

```powershell
# Analyze cross-tenant policies
Get-IOCrossTenantAccessReport

# Export to CSV
Get-IOCrossTenantAccessReport -ToCsv "cross-tenant.csv"
```

---

### Export-IOConditionalAccessReport

Exports all Conditional Access policies with automated gap analysis.

| Parameter      | Type     | Default | Description                                      |
|----------------|----------|---------|--------------------------------------------------|
| `-EnabledOnly` | `switch` | —       | Only include enabled policies (skip disabled)    |
| `-ToCsv`       | `string` | —       | Export results to a CSV file                     |

**Output columns:** PolicyName, PolicyId, State, CreatedDateTime, ModifiedDateTime, IncludeUsers, ExcludeUsers, IncludeGroups, IncludeApps, GrantControls, Gaps, GapCount

**Gap analysis detections:**
| Gap Flag            | What It Means                                              |
|---------------------|------------------------------------------------------------|
| `NarrowUserScope`   | Policy doesn't target "All" users and has no group targets |
| `HasUserExclusions`  | Users are excluded — potential bypass risk                 |
| `NoSessionControls`  | No sign-in frequency, persistent browser, or other session controls |
| `ReportOnlyMode`     | Policy is in report-only mode — not enforcing              |

**Examples:**

```powershell
# All CA policies with gap analysis
Export-IOConditionalAccessReport

# Only enabled policies
Export-IOConditionalAccessReport -EnabledOnly

# Export to CSV
Export-IOConditionalAccessReport -EnabledOnly -ToCsv "ca-policies.csv"
```

---

### Grant-IOManagedIdentityPermission

Grants Microsoft Graph (or other API) app role permissions to a managed identity's service principal. Supports `-WhatIf` and `-Confirm`.

| Parameter                  | Type       | Required | Description                                                |
|----------------------------|------------|----------|------------------------------------------------------------|
| `-ManagedIdentityName`     | `string`   | Yes*     | Display name of the managed identity                       |
| `-ManagedIdentityObjectId` | `string`   | Yes*     | Object ID (GUID) of the managed identity service principal |
| `-Permission`              | `string[]` | Yes      | One or more Graph permission names to grant                |
| `-ResourceAppId`           | `string`   | No       | Target API app ID (defaults to Microsoft Graph)            |
| `-ToCsv`                   | `string`   | No       | Export results to a CSV file                               |

*Use either `-ManagedIdentityName` or `-ManagedIdentityObjectId`, not both.

**Output columns:** ManagedIdentity, MIObjectId, Permission, ResourceApp, Status (GRANTED/ALREADY_ASSIGNED/NOT_FOUND/FAILED), Message

**Features:**
- Automatically skips permissions already assigned (no duplicates)
- Reports NOT_FOUND if the permission doesn't exist as an Application role
- Supports `-WhatIf` to preview changes without applying them
- Supports non-Graph APIs via `-ResourceAppId` (e.g., SharePoint Online)

**Examples:**

```powershell
# Grant Mail.Send to a managed identity by name
Grant-IOManagedIdentityPermission -ManagedIdentityName "my-func-app" -Permission "Mail.Send"

# Grant multiple permissions by object ID
Grant-IOManagedIdentityPermission -ManagedIdentityObjectId "aaaa-bbbb-cccc-dddd" `
                                  -Permission "User.Read.All", "Group.Read.All"

# Preview without applying (WhatIf)
Grant-IOManagedIdentityPermission -ManagedIdentityName "my-vm" -Permission "Sites.Read.All" -WhatIf

# Grant SharePoint Online permission instead of Graph
Grant-IOManagedIdentityPermission -ManagedIdentityName "my-app" `
                                  -Permission "Sites.Read.All" `
                                  -ResourceAppId "00000003-0000-0ff1-ce00-000000000000"

# Export results to CSV
Grant-IOManagedIdentityPermission -ManagedIdentityName "my-func" `
                                  -Permission "Mail.Send", "User.Read.All" `
                                  -ToCsv "permissions-granted.csv"
```

---

## CSV Export

**Every command** supports the `-ToCsv` parameter to export results to a CSV file.

```powershell
Get-IOExpiringSecrets -DaysUntilExpiry 60 -ToCsv "C:\Reports\expiring-secrets.csv"
```

### CSV Export Behavior

| Behavior                    | Details                                                       |
|-----------------------------|---------------------------------------------------------------|
| **Auto-extension**          | `.csv` is appended automatically if missing                  |
| **Auto-directory creation** | Parent directories are created if they don't exist           |
| **Encoding**                | UTF-8 with BOM for Excel compatibility                       |
| **Pipeline output**         | Results are still output to the pipeline even when exporting |
| **No results**              | CSV file is not created if there are no results              |

### CSV Security Restrictions

| Restriction              | Why                                                    |
|--------------------------|--------------------------------------------------------|
| **No UNC paths**         | Blocks `\\server\share` to prevent data exfiltration   |
| **No system directories**| Blocks writes to `C:\Windows`, `C:\Program Files`, etc.|
| **Filename validation**  | Rejects invalid filename characters                    |

### Combining with Pipeline

Results always flow to the pipeline, so you can chain commands:

```powershell
# Export to CSV AND filter in pipeline
Get-IOExpiringSecrets -DaysUntilExpiry 30 -ToCsv "all.csv" | Where-Object Status -eq 'EXPIRED'

# Pipe to Format-Table for display
Get-IOStaleGuests -InactiveDays 90 | Format-Table -AutoSize

# Pipe to Out-GridView for interactive filtering
Get-IOOrphanedApps -IncludeEnterpriseApps | Out-GridView
```

---

## Error Handling & Resilience

### Automatic Retry with Exponential Backoff

All Graph API calls go through a centralized handler that retries on:

| HTTP Status | Behavior                                                                 |
|-------------|--------------------------------------------------------------------------|
| **429**     | Throttled — waits `Retry-After` header × 2^attempt (max 120s), 3 retries |
| **500/502/503/504** | Server error — waits 2 × 2^attempt seconds (max 60s), 3 retries |

### Error Categories

| Error Type       | Module Behavior                                              |
|------------------|--------------------------------------------------------------|
| **403 Forbidden**| Throws `IO_InsufficientPermissions` with clear scope guidance|
| **404 Not Found**| Throws `IO_NotFound` with the resource URI                   |
| **Auth Failure** | Throws `IO_AuthenticationFailed` with the flow name          |
| **Other errors** | Throws `IO_GraphRequestFailed` with HTTP status code         |

### Circuit Breaker (Get-IOUsersWithoutMFA)

When checking MFA for large user sets, if more than 10% of per-user API calls fail, the command stops early and warns about missing permissions — rather than making thousands of failing calls.

### Batched Processing

`Get-IOUsersWithoutMFA` processes users in configurable batches (default 5) with 200ms pauses between batches for tenants with 100+ users, to avoid triggering throttling.

### Progress Bars

Long-running commands show `Write-Progress` bars (cleaned up properly in `finally` blocks even if errors occur).

---

## Security Features

| Feature                        | Description                                                          |
|--------------------------------|----------------------------------------------------------------------|
| **TLS 1.2/1.3 enforced**      | All connections use TLS 1.2 or 1.3 — older protocols are blocked    |
| **SecureString-only secrets**  | Client secrets, certificate passwords, and access tokens must be `SecureString` |
| **BSTR zero-free**             | Certificate passwords are freed from unmanaged memory via `ZeroFreeBSTR` |
| **Certificate disposal**       | X509 certificates are disposed in `finally` blocks to release private keys |
| **Secret redaction in logs**   | All log output redacts values matching patterns for secrets, passwords, tokens, API keys, Bearer tokens, and JWTs |
| **OData injection prevention** | User-supplied filter values (app names, etc.) are validated against `;$&\/<>{}` characters |
| **SSRF protection**            | Pagination `@odata.nextLink` URLs are validated to only follow `graph.microsoft.com/.us/.de/.cn` domains or relative `v1.0/beta` paths |
| **UNC path blocking**          | CSV export blocks `\\server\share` paths to prevent data exfiltration to network shares |
| **System path protection**     | CSV export blocks writes to `C:\Windows`, `C:\Program Files`, and other system directories |
| **GUID validation**            | `TenantId`, `ClientId`, and `ManagedIdentityObjectId` are regex-validated as proper GUIDs |
| **StrictMode Latest**          | The module runs under `Set-StrictMode -Version Latest` to catch undefined variables and property access issues |
| **EphemeralKeySet**            | Certificate files are loaded with `EphemeralKeySet` flag — private keys never persist to disk |

---

## License Requirements

| Feature / Command                          | License Needed       |
|--------------------------------------------|---------------------|
| Most commands                              | Entra ID Free       |
| `Get-IODormantApps` (signInActivity)       | Entra ID P1/P2      |
| `Get-IOStaleGuests` (signInActivity)       | Entra ID P1/P2      |
| `Get-IOServicePrincipalSignInFailures`     | Entra ID P1/P2      |
| `Get-IOPasswordOnlyAccounts`               | Entra ID P1/P2      |

> Commands that require P1/P2 will display a warning and return gracefully if the license is not available — they will not throw errors.

---

## Troubleshooting

### "Insufficient permissions" errors

Make sure all required scopes are consented. For delegated flows, the user must be an admin or scopes must be admin-consented. For app-only flows, grant Application permissions and admin-consent them.

```powershell
# Check current scopes
(Get-MgContext).Scopes
```

### "Circuit breaker triggered" on Get-IOUsersWithoutMFA

This means too many per-user auth method checks failed. Usually caused by missing `UserAuthenticationMethod.Read.All` permission.

### Sign-in log commands return nothing

`Get-IOServicePrincipalSignInFailures` and `Get-IOPasswordOnlyAccounts` require Entra ID P1/P2 license. They will display a warning if your tenant doesn't have the license.

### "Admin consent workflow is not configured"

`Get-IOPendingAdminConsent` requires the admin consent workflow to be enabled in the Entra ID portal under **Enterprise Applications > User Settings > Admin consent requests**.

### Throttling (429 errors)

The module handles 429 throttling automatically with retry. For very large tenants, consider:
- Running commands during off-peak hours
- Using a smaller `-BatchSize` for `Get-IOUsersWithoutMFA`
- Avoiding running multiple commands simultaneously

### Connection issues

```powershell
# Verify connection status
Get-MgContext

# Force reconnect
Disconnect-IdentityOps
Connect-IdentityOps
```

---

## Quick Reference Card

```
Connect-IdentityOps                           # Connect (interactive)
Connect-IdentityOps -DeviceCode               # Connect (device code)
Disconnect-IdentityOps                        # Disconnect

Get-IOExpiringSecrets                          # Expiring app secrets & certs
Get-IOExpiringSSOCerts                         # Expiring SSO/SAML certs
Get-IOStaleGuests                              # Inactive guest accounts
Get-IOOrphanedApps                             # Apps with no owners
Get-IOOverPrivilegedApps                       # Over-privileged app permissions
Get-IOUsersWithoutMFA                          # Users missing MFA
Get-IODormantApps                              # Apps with no sign-in activity
Get-IOOrphanedRoleAssignments                  # Roles assigned to deleted/disabled users
Get-IOOwnerlessGroups                          # Groups with no owners
Get-IOPendingAdminConsent                      # Apps awaiting admin consent
Get-IOServicePrincipalSignInFailures           # SP sign-in errors
Get-IOPasswordOnlyAccounts                     # Service accounts on passwords
Get-IOCrossTenantAccessReport                  # Cross-tenant policy risks
Export-IOConditionalAccessReport               # CA policies + gap analysis
Grant-IOManagedIdentityPermission              # Assign Graph perms to MI
```

Add `-ToCsv "filename.csv"` to any command to export results.
