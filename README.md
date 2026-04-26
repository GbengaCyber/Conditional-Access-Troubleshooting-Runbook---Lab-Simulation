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
| Target | Microsoft 365 — Outlook Web and Azure resources |
| Tools used | Entra ID Portal, Microsoft Sentinel KQL |

The user was completely locked out. Sign-in logs showed a block but the reason was not immediately obvious. This runbook walks through exactly how it was diagnosed and resolved.

---

## What Was Actually Happening

The corporate network Conditional Access policy required a compliant device. The MacBook was not enrolled in Intune, so Entra ID refused to issue a token regardless of the fact that the user's credentials were correct and MFA passed. Adding the IPv6 address to the trusted Named Location helped get past the initial network check, but the device compliance requirement still blocked the token.

The fix required two things: correcting the Named Location to include the IPv6 range, and modifying the CA policy so that macOS browser sessions were no longer held to the compliant device requirement.

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

## Step 1 — Read the Sign-In Logs

The first place to look is always the Entra ID sign-in logs. Navigate to the Entra portal, go to Monitoring and then Sign-in logs, and filter by the affected user.

Running this in Sentinel gets the same result faster:

```kql
SigninLogs
| where UserPrincipalName contains "johnsmith"
| project TimeGenerated, ResultType, ResultDescription, AppDisplayName
```

The result came back with ResultType 53003.

<img width="800" alt="image" src="https://github.com/user-attachments/assets/a85a85f9-cdee-4ac2-af5b-f2f1b37336b4" />


ResultType 53003 means one thing: a Conditional Access policy blocked the token from being issued. The full description read:

> Access has been blocked by Conditional Access policies. The access policy does not allow token issuance.

This tells you the user got past credential verification and MFA but was stopped before receiving a token. The block happened at the authorization layer, not the authentication layer.

The user had also seen this error on screen when the sign-in was first attempted:

![Sign-in error AADSTS900561](./screenshots/01-sign-in-error-aadsts900561.png)

AADSTS900561 is a redirect issue that appears when the auth flow gets interrupted mid-session. It is a symptom, not the root cause.

---

## Step 2 — Find Which Policy Is Blocking

Open the individual sign-in event and go to the Conditional Access tab. This shows every policy that evaluated the sign-in and what the result was.

![CA policy failure tab](./screenshots/06-ca-policy-failure-tab.png)

The blocking policy was Corporate Network — MFA Required and the grant controls listed were Require compliant device and Require MFA. The result was Failure. MFA likely passed. The compliant device requirement did not.

---

## Step 3 — Understand Why the Device Failed

The user's corporate Windows VM was fully enrolled in Intune and marked compliant. But the sign-in was happening from a MacBook running Firefox. Entra ID evaluates the device that is making the authentication request, not the device the user is trying to access.

The MacBook was not enrolled in Intune. From Entra ID's perspective it was an unknown, unmanaged device. So the compliant device check failed and the token was never issued.

When the user clicked Continue on the device prompt, they saw this:

![Device compliance prompt](./screenshots/02-device-compliance-prompt.png)

The message was essentially: enroll this device or you cannot proceed.

---

## Step 4 — Fix the Named Location First

Before touching the CA policy there was a separate issue to resolve. The corporate network Named Location in Entra ID only had the IPv4 range defined. The user's connection was coming over IPv6, so Entra ID did not recognize it as a trusted corporate network location at all.

Navigate to Entra ID, then Protection, then Named Locations, and open the Corporate Network trusted location entry. Add the IPv6 range alongside the existing IPv4 entry.

![Named location with IPv6 added](./screenshots/03-named-location-ipv6-added.png)

This alone did not fix the block but it was a necessary step. Without the correct IP ranges, the network condition in the policy would never match correctly.

---

## Step 5 — Modify the CA Policy for macOS

With the Named Location corrected, the remaining block was the compliant device requirement applied to the MacBook.

The cleanest long-term solution is enrolling the MacBook in Intune via Company Portal. The faster fix for a situation where that is not immediately possible is modifying the CA policy to treat macOS browser sessions differently.

How to do it safely: go to Entra ID, then Protection, then Conditional Access, then Policies. Open the blocking policy. Before making any changes, switch the policy to Report-only mode. This lets you make changes without the risk of locking anyone out while you work.

Go to Conditions and then Device platforms. Set Include to Any device and add macOS to the Exclude list. Save the policy and switch it back to On. Test the sign-in before closing anything.

If you want to be more precise, create a second CA policy scoped specifically to macOS that requires MFA only with no device compliance requirement. This gives macOS users access while keeping them under a defined policy rather than having no controls at all.

---

## Result

After the policy change, the sign-in completed successfully. The "Stay signed in?" prompt confirmed that all CA checks had passed and a token was issued.

![Successful Entra sign-in](./screenshots/09-entra-signin-success.png)

---

## What to Do Long Term

Disabling device compliance for macOS is a pragmatic fix but it does reduce the security posture. If employees regularly work from personal Macs, the proper fix is enrolling those devices in Intune through Company Portal. Once enrolled and compliant, the CA policy works as designed without any platform exclusions.

Either way, document the decision. The next engineer who reviews this policy needs to understand why macOS is excluded.

---

## Quick Reference for This Error Pattern

If a user is blocked with ResultType 53003 and the CA policy shows a compliant device failure, ask these questions before touching anything.

What device is the user authenticating from, and is that device enrolled in Intune? The device they are trying to access is irrelevant to this check.

Is the user's IP in the correct Named Location, including IPv6 ranges? A missing IPv6 entry is one of the most common causes of unexpected CA policy mismatches.

Is the policy set to require all grant controls or any grant control? If it requires all, then both MFA and device compliance must pass.

Is there a break-glass admin account excluded from this policy? There should always be one. Add one before making any changes.

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

> This runbook was built from a real troubleshooting session in a lab environment (EagleSecureIT). Settings and policy names have been kept as-is to reflect how the investigation actually unfolded.
