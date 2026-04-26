# Conditional Access Troubleshooting Runbook
### For IT and Security Teams Managing Mixed Device Environments

---

## The Business Problem

Picture this. A staff member works from home on their personal MacBook. They try to access company email and get blocked. They call IT. IT does not have a documented process for this. Someone spends 45 minutes poking around the Entra portal before finding the right setting. The employee loses half a morning of productivity. And if this happens to ten people in a month, that is a real cost — in time, in frustration, and in the risk of someone making a hasty policy change that opens a security gap.

This runbook exists to prevent that.

It documents a real troubleshooting session from start to finish — the errors, the investigation steps, the dead ends, and the fixes. The goal is that the next time this happens, anyone on the team can pick this up and work through it systematically rather than starting from scratch.

---

## Who This Is For

This is written for IT administrators, security analysts, and SOC engineers who manage Microsoft Entra ID and Intune in environments where employees use a mix of corporate and personal devices — particularly where macOS or unmanaged devices need access to Microsoft 365 resources.

---

## The Scenario

| Detail | Info |
|--------|------|
| User | johnsmith@eaglesecureit.com |
| Device | Personal MacBook, Firefox browser |
| Target | Microsoft 365 (Outlook Web, Azure resources) |
| Secondary issue | RDP access to an Azure VM (soclab) from the same MacBook |
| Tools used | Entra ID Portal, Azure Run Command, Microsoft Sentinel KQL |

The user was completely locked out. Sign-in logs showed a block but the reason was not immediately obvious. This runbook walks through exactly how it was diagnosed and resolved.

---

## What Was Actually Happening

There were two separate problems layered on top of each other, which made it look more complicated than it was.

The first problem was the Entra ID sign-in block. The corporate network Conditional Access policy required a compliant device. The MacBook was not enrolled in Intune, so Entra ID refused to issue a token regardless of the fact that the user's credentials were correct and MFA passed. Adding the IPv6 address to the trusted Named Location helped get past the initial network check, but the device compliance requirement still blocked the token.

The second problem was the RDP connection to the Azure VM. Even after fixing the Entra sign-in, the VM was rejecting the connection because Network Level Authentication was enforced and the local account did not have the right to log on through Remote Desktop Services.

Both problems are common. Neither is obvious until you know what to look for.

---

## Business Impact

| Area | Impact |
|------|--------|
| Productivity | Employee locked out of all M365 resources during working hours |
| IT time cost | ~45 minutes to diagnose without a runbook, ~10 minutes with one |
| Security risk | Rushed CA policy changes without testing can create gaps or cause wider lockouts |
| Compliance | Unmanaged device accessing corporate data without proper controls flagged in audit logs |
| Scalability | Without documentation, every repeat incident costs the same time to resolve |

---

## The Most Important Thing to Understand

Conditional Access policies evaluate the device making the authentication request, not the device the user is trying to reach. A user sitting at an unmanaged Mac trying to access a fully compliant corporate VM will still be blocked because the Mac is what Entra ID sees. That distinction is obvious in hindsight but costs hours when you do not know it.

---

## Part 1 — Entra ID Sign-In Blocked

### What the user reported

The user tried to sign into Microsoft 365 and received a generic "having trouble signing you in" error with error code AADSTS900561.

![Sign-in error](./screenshots/01-sign-in-error-aadsts900561.png)

AADSTS900561 means the authentication endpoint received a GET request when it expected a POST. This is a redirect issue that shows up when the auth flow gets interrupted mid-session. It is a symptom of something blocking the login before it completes, not the root cause itself.

---

### Step 1 — Read the sign-in logs

The first place to look is always the Entra ID sign-in logs. Navigate to the Entra portal, go to Monitoring and then Sign-in logs, and filter by the affected user.

Running this in Sentinel gets the same result faster:

```kql
SigninLogs
| where UserPrincipalName contains "johnsmith"
| project TimeGenerated, ResultType, ResultDescription, AppDisplayName
```

The result came back with ResultType 53003.

![Sentinel sign-in log showing CA block](./screenshots/05-signin-log-ca-blocked-53003.png)

ResultType 53003 means one thing: a Conditional Access policy blocked the token from being issued. The full description read:

> Access has been blocked by Conditional Access policies. The access policy does not allow token issuance.

This tells you the user got past credential verification and MFA but was stopped before receiving a token. The block happened at the authorization layer, not the authentication layer.

---

### Step 2 — Find which policy is blocking

Open the individual sign-in event and go to the Conditional Access tab. This shows every policy that evaluated the sign-in and what the result was.

![CA policy failure tab](./screenshots/06-ca-policy-failure-tab.png)

The blocking policy was Corporate Network - MFA Req... and the grant controls listed were Require compliant device and Require MFA. The result was Failure. MFA likely passed. The compliant device requirement did not.

---

### Step 3 — Understand why the device failed

The user's corporate Windows VM was fully enrolled in Intune and marked compliant. But the sign-in was happening from a MacBook running Firefox. Entra ID evaluates the device that is making the authentication request, not the device the user is trying to access.

The MacBook was not enrolled in Intune. From Entra ID's perspective it was an unknown, unmanaged device. So the compliant device check failed and the token was never issued.

When the user clicked Continue on the device prompt, they saw this:

![Device compliance prompt](./screenshots/02-device-compliance-prompt.png)

The message was essentially: enroll this device or you cannot proceed.

---

### Step 4 — Fix the Named Location first

Before touching the CA policy there was a separate issue. The corporate network Named Location in Entra ID only had the IPv4 range defined. The user's connection was coming over IPv6, so Entra ID did not recognize it as a trusted corporate network location at all.

Navigate to Entra ID, then Protection, then Named Locations, and open the Corporate Network trusted location entry. Add the IPv6 range alongside the existing IPv4 entry.

![Named location with IPv6 added](./screenshots/03-named-location-ipv6-added.png)

This alone did not fix the block but it was a necessary step. Without the correct IP ranges, the network condition in the policy would never match correctly.

---

### Step 5 — Modify the CA policy for macOS

With the Named Location corrected, the remaining block was the compliant device requirement applied to the MacBook.

The cleanest long-term solution is enrolling the MacBook in Intune via Company Portal. The faster fix for a situation where that is not immediately possible is modifying the CA policy to treat macOS browser sessions differently.

How to do it safely: go to Entra ID, then Protection, then Conditional Access, then Policies. Open the blocking policy. Before making any changes, switch the policy to Report-only mode. This lets you make changes without the risk of locking anyone out while you work.

Go to Conditions and then Device platforms. Set Include to Any device and add macOS to the Exclude list. Save the policy and switch it back to On. Test the sign-in before closing anything.

If you want to be more precise, create a second CA policy scoped specifically to macOS that requires MFA only with no device compliance requirement. This gives macOS users access while keeping them under a defined policy rather than having no controls at all.

---

### Result

After the policy change, the sign-in completed successfully. The "Stay signed in?" prompt confirmed that all CA checks had passed and a token was issued.

![Successful Entra sign-in](./screenshots/09-entra-signin-success.png)

---

### What to do long term

Disabling device compliance for macOS is a pragmatic fix but it does reduce the security posture. If employees regularly work from personal Macs, the proper fix is enrolling those devices in Intune through Company Portal. Once enrolled and compliant, the CA policy works as designed without any platform exclusions.

Either way, document the decision. The next engineer who reviews this policy needs to understand why macOS is excluded.

---

### Quick reference for this error pattern

If a user is blocked with ResultType 53003 and the CA policy shows a compliant device failure, ask these questions before touching anything.

What device is the user authenticating from, and is that device enrolled in Intune? The device they are trying to access is irrelevant to this check.

Is the user's IP in the correct Named Location, including IPv6 ranges? A missing IPv6 entry is one of the most common causes of unexpected CA policy mismatches.

Is the policy set to require all grant controls or any grant control? If it requires all, then both MFA and device compliance must pass.

Is there a break-glass admin account excluded from this policy? There should always be one. Add one before making any changes.

---

## Part 2 — RDP Access Failure from macOS

### What the user reported

After fixing the Entra ID sign-in, the user tried to RDP from their MacBook into the Azure VM at 4.239.125.45. The connection attempt failed with error code 0x1307.

![RDP unable to connect error 0x1307](./screenshots/07-rdp-error-0x1307.png)

> We couldn't connect to the remote PC because the admin has restricted the type of logon that you may use.

Error 0x1307 is a logon restriction error. It means the user's security token does not have the privileges required for the type of logon being attempted. It is not a password error and it is not a firewall error.

---

### Step 1 — Check Sentinel for logon failures

Before touching anything on the VM, look at what Sentinel captured.

```kql
DeviceLogonEvents
| where DeviceName contains "soclab"
| project TimeGenerated, AccountName, DeviceName, ActionType, LogonType, Protocol
```

![Sentinel NTLM logon failure](./screenshots/04-sentinel-ntlm-logonfailed.png)

The result showed a LogonFailed event for labuser on soclab with LogonType Network and Protocol NTLM.

This tells you three things. The connection reached the machine, otherwise there would be no logon event at all. The authentication protocol being used was NTLM, not Kerberos. And the logon type was Network, which is what RDP uses when NLA is enabled.

---

### Step 2 — Get PowerShell access without RDP

This is where most people get stuck. RDP is broken, so how do you fix what is on the VM?

The answer for Azure VMs is Azure Run Command. Go to the Azure portal, find the VM, and look for Run Command under Operations in the left menu. This gives you a PowerShell prompt that executes directly on the VM through the Azure agent with no network dependency. It is invaluable for situations like this and worth knowing well before you need it.

---

### Step 3 — Check the Remote Desktop Users group

```powershell
net localgroup "Remote Desktop Users"
```

![Remote Desktop Users group empty](./screenshots/10-rdp-users-group-empty.png)

The group showed no members. That looked like the issue, but when we tried to add labuser, the command came back saying the account was already a member. This is a display quirk that can happen with domain-joined machines. The membership was fine.

---

### Step 4 — Check Network Level Authentication

NLA is the real culprit for this class of error when connecting from a Mac with a local Windows account.

```powershell
Get-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp" -Name "UserAuthentication"
```

![NLA enabled showing value 1](./screenshots/11-nla-enabled.png)

UserAuthentication: 1 — NLA was enforced.

When NLA is on, Windows requires the connecting client to authenticate at the network layer before the RDP session even starts. On a domain-joined Windows machine connecting to another domain machine, this works seamlessly via Kerberos. From a Mac connecting with a local Windows account over NTLM, the handshake cannot complete. Windows sees an incomplete authentication attempt and refuses the connection with 0x1307.

---

### Step 5 — Check the labuser account

```powershell
net user labuser
```

![labuser account details](./screenshots/12-labuser-account-details.png)

The account was clean. Active, no expiry, password set today, member of both Administrators and Remote Desktop Users. Nothing wrong with the account itself.

---

### Step 6 — Disable NLA

```powershell
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp" -Name "UserAuthentication" -Value 0
```

Verify the change took effect:

```powershell
Get-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp" -Name "UserAuthentication"
```

![NLA disabled showing value 0](./screenshots/13-nla-disabled.png)

UserAuthentication: 0 — NLA is now off.

---

### Step 7 — Fix the Remote Desktop Services logon right

After disabling NLA, the next attempt reached the Windows login screen but showed this:

![Other user screen on soclab](./screenshots/14-other-user-screen.png)

> To sign in remotely, you need the right to sign in through Remote Desktop Services.

This is a Group Policy user rights issue. The SeRemoteInteractiveLogonRight had been removed or overwritten, likely by a domain Group Policy. Being a member of the Remote Desktop Users group is not enough on its own if the group's logon right has been stripped at the policy level.

Fix it by exporting the local security policy, adding labuser, and reimporting:

```powershell
$sid = (New-Object System.Security.Principal.NTAccount("labuser")).Translate([System.Security.Principal.SecurityIdentifier]).Value

secedit /export /cfg "C:\Windows\Temp\secpol.cfg"
(Get-Content "C:\Windows\Temp\secpol.cfg") -replace 'SeRemoteInteractiveLogonRight = ', "SeRemoteInteractiveLogonRight = *$sid," | Set-Content "C:\Windows\Temp\secpol.cfg"
secedit /configure /db secedit.sdb /cfg "C:\Windows\Temp\secpol.cfg" /areas USER_RIGHTS
gpupdate /force
```

Verify the right was applied:

```powershell
secedit /export /cfg "C:\Windows\Temp\check.cfg"
Get-Content "C:\Windows\Temp\check.cfg" | Select-String "SeRemoteInteractiveLogonRight"
```

![SeRemoteInteractiveLogonRight granted](./screenshots/15-seremoteinteractive-granted.png)

The output confirmed labuser was explicitly listed alongside the Administrators group and the Remote Desktop Users group.

---

### Username format on Mac RDP clients

When connecting from Microsoft Remote Desktop on macOS, always use `.\labuser` or `soclab\labuser` to explicitly target the local account. Using just `labuser` can cause the client to attempt a domain lookup which will fail if the account is local.

---

### Production considerations

Disabling NLA removes a layer of protection. In a production environment, the better path is using Entra ID joined machines and connecting with Entra credentials rather than local accounts, or setting up Azure Bastion which handles the authentication layer separately and avoids the NTLM problem entirely.

---

## KQL Queries Reference

**All sign-in events for a specific user**

```kql
SigninLogs
| where UserPrincipalName contains "johnsmith"
| project TimeGenerated, ResultType, ResultDescription, AppDisplayName
| order by TimeGenerated desc
```

**Failed sign-ins only**

```kql
SigninLogs
| where UserPrincipalName contains "johnsmith"
| where ResultType != "0"
| project TimeGenerated, ResultType, ResultDescription, AppDisplayName, IPAddress, DeviceDetail
| order by TimeGenerated desc
```

**All CA-blocked sign-ins across the tenant**

```kql
SigninLogs
| where ResultType == "53003"
| project TimeGenerated, UserPrincipalName, AppDisplayName, ResultDescription, IPAddress
| order by TimeGenerated desc
```

**Which CA policies are failing per sign-in**

```kql
SigninLogs
| where UserPrincipalName contains "johnsmith"
| mv-expand ConditionalAccessPolicies
| project TimeGenerated,
    PolicyName = ConditionalAccessPolicies.displayName,
    PolicyResult = ConditionalAccessPolicies.result,
    GrantControls = ConditionalAccessPolicies.enforcedGrantControls
| where PolicyResult == "failure"
```

**All logon activity on a specific machine**

```kql
DeviceLogonEvents
| where DeviceName contains "soclab"
| project TimeGenerated, AccountName, DeviceName, ActionType, LogonType, Protocol
| order by TimeGenerated desc
```

**All NTLM failures across the environment**

```kql
DeviceLogonEvents
| where Protocol == "NTLM"
| where ActionType == "LogonFailed"
| project TimeGenerated, AccountName, DeviceName, LogonType, RemoteIP
| order by TimeGenerated desc
```

**Common ResultType codes**

| Code | Meaning |
|------|---------|
| 0 | Success |
| 50058 | Silent sign-in interrupted |
| 50074 | Strong authentication required but not satisfied |
| 53003 | Blocked by Conditional Access, token not issued |
| 53020 | Device compliance check failed |
| 70011 | Invalid scope requested |
| 900561 | Endpoint only accepts POST, received a GET |

---

## Final Word

Build your runbooks before the incident, not during it. The 45 minutes spent firefighting this without documentation is 45 minutes that compounds every time it happens again. A runbook like this, kept up to date as your environment changes, is one of the highest-value things an IT or security team can maintain.

---

> This runbook was built from a real troubleshooting session in a lab environment (EagleSecureIT). Some fixes applied here, such as disabling NLA, are appropriate for lab use but should be reviewed carefully before applying in production.
