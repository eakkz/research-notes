# SPNs, keytabs and service accounts

Three AD-side objects decide whether Kerberos works: the SPN that names SQL Server, the service account the app runs as, and the keytab that holds that account's key. Most failed PoCs fail on one of these.

## Service principal name (SPN)

An SPN is the name a client uses when it asks the KDC for a service ticket. For SQL Server over TCP it has this form ([Microsoft: Register a Service Principal Name](https://learn.microsoft.com/en-us/sql/database-engine/configure-windows/register-a-service-principal-name-for-kerberos-connections)):

```text
MSSQLSvc/<fully qualified host name>:<port>
MSSQLSvc/sql01.corp.example.com:1433
```

The SPN is an attribute on an AD account, the account that runs the SQL Server service. When the KDC gets a request for this SPN, it looks up which account carries it and encrypts the ticket with that account's key. SQL Server then decrypts it with the same key.

That leads to three failure cases:

- No account has the SPN. The KDC answers `Server not found in Kerberos database`. A Windows client would fall back to NTLM without an error; the Linux ODBC driver just fails.
- Two accounts have it (a duplicate). The KDC cannot choose and the login fails.
- The wrong account has it, for example an old service account. The ticket is encrypted with a key SQL Server does not have, and the login fails with errors such as "Cannot generate SSPI context".

SPNs are managed with `setspn` on Windows. `setspn -L <account>` lists an account's SPNs, `setspn -Q <spn>` finds who owns one, and `setspn -S` adds one after checking for duplicates. SQL Server registers its own SPN at startup only when it runs under certain built-in accounts or has permission to write SPNs. Otherwise a domain admin adds it by hand.

The port in the SPN means SQL Server must use a static port. A named instance on a dynamic port changes port on restart, and the SPN stops matching.

## Service account

The app needs its own AD identity. In this design it is a normal AD user account, for example `svc-app`, created in the same domain as SQL Server. It has no SPN of its own; it is only ever a client.

Same domain matters because a client in one realm asking for a service in another has to follow a cross-realm referral through a trust. Linux clients can do it, but it needs extra `krb5.conf` mapping and testing. Keeping both in one domain removes that work.

SQL Server sees this account, not the people who use the app. Auditing in SQL Server therefore shows `CORP\svc-app` for every query the app runs.

## Keytab

A keytab is a file holding an account's long-term keys. Each entry holds the principal, a key version number (kvno), an encryption type and the key itself ([MIT: keytab](https://web.mit.edu/kerberos/krb5-latest/doc/basic/keytab_def.html)). `kinit -kt <file> <principal>` uses it to get a TGT without typing a password.

The keys in a keytab are derived from the password, not the password itself. They still work as a password for Kerberos: anyone with the file can act as the account. Treat a keytab like a password in every respect, including storage, access control and rotation.

### Encryption types

AD can derive keys with AES256, AES128 or the old RC4. Microsoft has been removing RC4:

- Since the April 2026 Windows update, domain controllers assume AES-only for accounts that have no explicit `msDS-SupportedEncryptionTypes` value. The July 2026 updates removed the rollback setting, so this is now enforced ([KB5073381](https://support.microsoft.com/en-us/topic/how-to-manage-kerberos-kdc-usage-of-rc4-for-service-account-ticket-issuance-changes-related-to-cve-2026-20833-1ebcda33-720a-4da8-93c1-b0496e1910dc)).
- Windows Server 2025 domain controllers do not issue RC4 TGTs at all ([Microsoft: Detect and remediate RC4 usage](https://learn.microsoft.com/en-us/windows-server/security/kerberos/detect-remediate-rc4-kerberos)).

The KB names the most common non-Windows failure: a keytab exported with only RC4 keys. So ask for AES256 (and AES128) entries, and have the AES options ticked on the account. `klist -kte <keytab>` shows what a keytab contains.

### How keytabs are made

| Tool | Runs on | Notes |
| --- | --- | --- |
| `ktpass` | Windows, by an AD admin | Microsoft-documented. Always pass `/crypto` (for example `AES256-SHA1`) and `/ptype KRB5_NT_PRINCIPAL`. Options that set a password change the account password ([ktpass](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/ktpass)) |
| `ktutil` | Linux | MIT tool. Builds entries from the password you type; needs the right kvno and salt ([MIT: ktutil](https://web.mit.edu/kerberos/krb5-latest/doc/admin/admin_commands/ktutil.html)) |
| `adutil` | Linux joined to AD | Microsoft tool for SQL Server on Linux. Documented for SQL Server's own keytab, not for client keytabs ([adutil](https://learn.microsoft.com/en-us/sql/linux/security/authentication/adutil-introduction)) |
| `msktutil` | Linux | Third-party; can also rotate the password on a schedule |

Which tool is used is the AD team's choice. Whichever they pick, the keytab must have the right principal, AES keys, and the current kvno.

### kvno and password changes

Each password change creates new keys with a higher kvno. A keytab built before the change holds the old keys, so `kinit` fails with `Preauthentication failed` once AD switches. Microsoft's tutorial says that when the password changes, "you must update the keytab with the new password and Key Version Number (KVNO)" ([Microsoft: AD authentication for SQL Server on Linux tutorial](https://learn.microsoft.com/en-us/sql/linux/security/authentication/active-directory-tutorial)).

There are two ways to live with this:

- An AD policy exception so the password never expires, with manual rotation on a schedule.
- Planned rotation: add the next kvno's keys to the keytab first, update the Secret, then change the password in AD. [02-architecture.md, section 6](../02-architecture.md#6-keytab-rotation) shows the order.

## gMSA, and why it does not apply here

A group Managed Service Account (gMSA) is an AD account whose password AD generates and rotates (every 30 days by default). Allowed Windows hosts fetch the current password from a domain controller, so nobody handles it ([Microsoft: gMSA overview](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/group-managed-service-accounts/group-managed-service-accounts/group-managed-service-accounts-overview)). It removes the keytab rotation problem completely, which is why security teams like it.

Only Windows components fetch gMSA passwords. Kubernetes supports gMSA for Windows pods only, and ROSA has no Windows container support. AWS built a Linux daemon, `credentials-fetcher`, that does it for Amazon ECS. It runs on the host, and ROSA customers cannot change ROSA hosts.

## Where to go next

- [04-linux-kerberos-client.md](04-linux-kerberos-client.md) for what the container does with the keytab.
- [Microsoft: Understanding AD authentication for SQL Server on Linux and containers](https://learn.microsoft.com/en-us/sql/linux/security/authentication/understand-active-directory) explains keytabs and the ticket flow with diagrams. It is written for SQL Server running on Linux, but the concepts are the same.
