# Active Directory Home Lab — Part 2: Group Policy, Password Policy & Account Lockout

## Overview

Part 1 covered setting up the domain and joining a client machine. Part 2 is about Group Policy, specifically configuring a password policy, an account lockout policy and simulating a locked out user scenario from start to finish.

---

## Part 1 — Configuring the Password and Lockout Policy

Password and account lockout policies in Active Directory need to be configured in the **Default Domain Policy** GPO. This is a Microsoft recommendation, these policies must apply at the domain level to function correctly. A custom GPO linked at the domain level will be overridden by the Default Domain Policy unless specific steps are taken to change the link order.

On the DC, opened **Group Policy Management** → right clicked **Default Domain Policy** → **Edit**

Navigated to:
```
Computer Configuration → Policies → Windows Settings → Security Settings → Account Policies
```

### Password Policy settings applied:

| Setting | Value |
|---|---|
| Minimum password length | 8 characters |
| Password must meet complexity requirements | Enabled |
| Maximum password age | 30 days |

### Account Lockout Policy settings applied:

| Setting | Value |
|---|---|
| Account lockout threshold | 3 invalid logon attempts |
| Account lockout duration | 15 minutes |
| Reset account lockout counter after | 15 minutes |

After saving, ran `gpupdate /force` on the DC to push the policy out.

---

## Part 2 — Applying the Policy to the Client

On **DidYouRebootIT-PC1**, opened cmd as Administrator and ran:

```
gpupdate /force
```

To verify the policy was actually applied, ran:

```
gpresult /r /scope computer
```

Confirmed **Default Domain Policy** was listed under Applied Group Policy Objects on the client machine.

---

## Troubleshooting — Policy Not Applying Initially

The first attempt involved creating a separate custom GPO named `Password Policy` linked to the domain. Despite the GPO showing as applied in `gpresult`, the account lockout was not triggering after multiple failed login attempts.

**Diagnostic step:** Ran the following on the DC to check the actual bad logon count being recorded:

```powershell
Get-ADUser -Identity stefi -Properties LockedOut, BadLogonCount
```

Output showed `BadLogonCount: 17` with `LockedOut: False` — confirming the policy was not enforcing despite being listed as applied.

**Root cause:** The custom GPO was being overridden by the Default Domain Policy, which had higher link order priority. Account lockout and password policies need to be set directly in the Default Domain Policy to apply correctly at the domain level.

**Fix:** Moved all policy settings into the Default Domain Policy, then deleted the custom GPO. Policy applied correctly on the next `gpupdate /force` cycle on the client.

**Takeaway:** For password and account lockout policies in AD, always configure them in the Default Domain Policy. A separate GPO will lose to it in link order unless manually reordered — and even then, this is not the recommended approach.

---

## Part 3 — Simulating a Helpdesk Scenario: Locked Out User

With the policy correctly applied, simulated a common helpdesk scenario — a user locked out after too many failed login attempts.

On **DidYouRebootIT-PC1**, signed out of the domain account and entered the wrong password 3 times on the login screen. The account locked out as expected.

![Lockout message on the Win11 login screen](images/6.%20User_logged_out.png)

### Resolving the lockout from the DC

Opened **Active Directory Users and Computers (ADUC)** on the DC.

Navigated to the Users container → located **Stefanija Stefanovska** → right clicked → **Properties** → **Account** tab.

The "Unlock account" checkbox was visible, confirming the account was locked. Ticked the checkbox → **Apply** → **OK**.

![Account tab showing locked state and unlock option](images/7.%20Unlocking_account.png)

Returned to **DidYouRebootIT-PC1** and logged in successfully with the correct credentials.

![Successful login after unlock](images/8.%20Successful_login.png)

---

## Key Takeaways

- Password and lockout policies belong in the Default Domain Policy. Everything else I learned the hard way which is documented in the troubleshooting section above.
