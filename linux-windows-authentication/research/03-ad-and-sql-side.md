# AD and SQL Server side requirements for Windows authentication from Linux

Researched 2026-11-23. This file covers what Active Directory (AD) and SQL Server need so that Linux containers, including pods on Red Hat OpenShift Service on AWS, can sign in to Microsoft SQL Server with Windows authentication. On Linux that means Kerberos through GSSAPI, or NTLM. Every claim links to the page that owns it. Anything marked "not confirmed" could not be verified in a primary source and is also listed near the end.

## 1. Kerberos basics for SQL Server

Microsoft names two conditions for Kerberos with SQL Server. The client and server must be in the same domain or in trusted domains. A Service Principal Name (SPN) must also be registered in AD, which acts as the Key Distribution Center (KDC). The registered SPN maps to the Windows account that started the SQL Server service. If the SPN is missing or registration failed, Kerberos is not used ([Register a Service Principal Name for Kerberos Connections](https://learn.microsoft.com/en-us/sql/database-engine/configure-windows/register-a-service-principal-name-for-kerberos-connections)). In an AD environment Kerberos is always attempted first. If the client cannot get a ticket, authentication uses NTLM ([same article](https://learn.microsoft.com/en-us/sql/database-engine/configure-windows/register-a-service-principal-name-for-kerberos-connections)).

### SPN formats

The SPN article lists these provider-generated formats ([source](https://learn.microsoft.com/en-us/sql/database-engine/configure-windows/register-a-service-principal-name-for-kerberos-connections)):

| SPN format | When the client builds it |
| --- | --- |
| `MSSQLSvc/<FQDN>:<port>` | TCP connections, for both default and named instances |
| `MSSQLSvc/<FQDN>` | Default instance over a protocol other than TCP |
| `MSSQLSvc/<FQDN>:<instancename>` | Named instance over a protocol other than TCP (named pipes, shared memory) |

`<FQDN>` is the server's fully qualified domain name. When the port is part of the SPN, SQL Server must have the TCP protocol enabled for Kerberos to work ([source](https://learn.microsoft.com/en-us/sql/database-engine/configure-windows/register-a-service-principal-name-for-kerberos-connections)). The Microsoft ODBC driver on Linux documents only the TCP form, `MSSQLSvc/<fqdn>:<port>`, as its default ([Use Integrated Authentication, ODBC Driver](https://learn.microsoft.com/en-us/sql/connect/odbc/linux-mac/using-integrated-authentication)). So for Linux clients the `<FQDN>:<port>` SPN is the one that has to exist.

### Which account holds the SPN

The SPN goes on the account that runs the SQL Server service. At startup SQL Server tries to register its own SPN. That works when it runs as Local System, NETWORK SERVICE, a virtual account, a managed service account (MSA), or an account allowed to write SPNs. Otherwise a domain administrator must register the SPN by hand ([source](https://learn.microsoft.com/en-us/sql/database-engine/configure-windows/register-a-service-principal-name-for-kerberos-connections)). To let the startup account register its own SPN, give it "Validated write to service principal name", "Read servicePrincipalName" and "Write servicePrincipalName" on the SQL Server computer object ([source](https://learn.microsoft.com/en-us/sql/database-engine/configure-windows/register-a-service-principal-name-for-kerberos-connections)). When SQL Server itself runs on Linux, Microsoft's tutorial puts the SPNs on a dedicated AD user account (example name `sqlsvc`) ([Tutorial: Use Active Directory Authentication for SQL Server on Linux](https://learn.microsoft.com/en-us/sql/linux/security/authentication/active-directory-tutorial)).

### setspn

`setspn.exe` is built into Windows. It reads, adds and deletes SPNs on an account. Microsoft's example registers a TCP SPN on a domain user account: `setspn -S MSSQLSvc/myhost.redmond.microsoft.com:1433 redmond\accountname`. An existing SPN must be deleted with `-D` before it can be registered again ([source](https://learn.microsoft.com/en-us/sql/database-engine/configure-windows/register-a-service-principal-name-for-kerberos-connections)). Microsoft's troubleshooting guidance collects SPNs with `SETSPN -L`, finds the account that holds an SPN with `SETSPN -Q`, and moves an SPN to the right account with `-D` then `-S` ([Overview of consistent authentication issues in SQL Server](https://learn.microsoft.com/en-us/troubleshoot/sql/database-engine/connect/consistent-authentication-connectivity-issues)). The Linux tutorial registers both `MSSQLSvc/<FQDN>:<port>` and `MSSQLSvc/<NetBIOS name>:<port>` with `setspn -A`. If the TCP port changes later, the SPN and the keytab must both be updated ([Linux tutorial](https://learn.microsoft.com/en-us/sql/linux/security/authentication/active-directory-tutorial)).

### What happens when the SPN is wrong

If the SPN maps to the wrong account, authentication fails. If the SPN lookup fails, both local and remote connections use NTLM ([SPN article](https://learn.microsoft.com/en-us/sql/database-engine/configure-windows/register-a-service-principal-name-for-kerberos-connections)). Microsoft lists missing SPNs, duplicate SPNs and SPNs on the wrong account as causes of "Cannot generate SSPI context" and "Login failed for user 'NT AUTHORITY\ANONYMOUS LOGON'" ([consistent authentication issues](https://learn.microsoft.com/en-us/troubleshoot/sql/database-engine/connect/consistent-authentication-connectivity-issues)).

### Checking whether a connection used Kerberos or NTLM

Run this on the connection under test. The `auth_scheme` column is `KERBEROS` when Kerberos was used ([SPN article](https://learn.microsoft.com/en-us/sql/database-engine/configure-windows/register-a-service-principal-name-for-kerberos-connections)):

```sql
SELECT net_transport, auth_scheme
FROM sys.dm_exec_connections
WHERE session_id = @@SPID;
```

The column "specifies SQL Server/Windows Authentication scheme used with this connection". Querying the view requires `VIEW SERVER STATE`, or `VIEW SERVER PERFORMANCE STATE` on SQL Server 2022 and later ([sys.dm_exec_connections](https://learn.microsoft.com/en-us/sql/relational-databases/system-dynamic-management-views/sys-dm-exec-connections-transact-sql)). AWS uses the same query and refers to an "NTLM connection" result when Kerberos was not used ([Joining your DB instance to self-managed Active Directory](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_SQLServer_SelfManagedActiveDirectory.Joining.html)).

### Kerberos Configuration Manager for SQL Server

Microsoft still documents the tool. The support article (KB 2985455, dated 2026-10-20) says it scans SQL Server, SSRS and SSAS instances and reports missing or duplicate SPNs and delegation problems. It can also generate a fix script or apply the fix. The same article says the tool is provided as-is, gets no support or updates, and was not updated for the WMI provider used by SQL Server 2022 and later. For those versions Microsoft points to SQLCheck instead ([Use Kerberos Configuration Manager Tool for SQL Server Issues](https://learn.microsoft.com/en-us/troubleshoot/sql/database-engine/connect/using-kerberosmngr-sqlserver)). The Fix and Generate commands work only for default instances and for named instances on static ports ([same article](https://learn.microsoft.com/en-us/troubleshoot/sql/database-engine/connect/using-kerberosmngr-sqlserver)). The Download Center lists version 4.2.1, published July 15, 2024. It supports Windows operating systems only, and SQL Server 2008 through SQL Server 2019 ([Download Center](https://www.microsoft.com/download/details.aspx?id=39046)). Whether it can inspect a SQL Server on Linux instance is not confirmed.

## 2. Client service account and the SQL Server login

### The application's AD account

Microsoft does not prescribe an account type for a Linux client. The ODBC driver guidance says the application gets a ticket-granting ticket (TGT) for its own principal in one of three ways: `kinit` with a password typed at the prompt, `kinit` with a keytab provisioned for that principal, or a Kerberos PAM login. The driver then authenticates as the default principal in the process's credential cache ([Use Integrated Authentication, ODBC Driver](https://learn.microsoft.com/en-us/sql/connect/odbc/linux-mac/using-integrated-authentication)). If several users share one service principal, the database login records that principal, not each person using the application ([same page](https://learn.microsoft.com/en-us/sql/connect/odbc/linux-mac/using-integrated-authentication)). The driver does not renew Kerberos credentials, so the deployment has to renew or reacquire the ticket before it expires ([same page](https://learn.microsoft.com/en-us/sql/connect/odbc/linux-mac/using-integrated-authentication)).

An ordinary AD user account works as the client identity. A user account without an SPN does not need a value in `msDS-SupportedEncryptionTypes`; the device configuration decides the encryption type for service tickets and session keys. An account whose password was last set before AES support existed has no AES keys until the password is changed ([Detect and Remediate RC4 Usage in Kerberos](https://learn.microsoft.com/en-us/windows-server/security/kerberos/detect-remediate-rc4-kerberos)).

### Creating the SQL Server login

The syntax is `CREATE LOGIN [<domainName>\<login_name>] FROM WINDOWS;`, with optional `DEFAULT_DATABASE` and `DEFAULT_LANGUAGE`. Use the pre-Windows 2000 format `domain\user`. The UPN format `user@domain` is not accepted. Logins based on AD users are limited to names of fewer than 21 characters ([CREATE LOGIN](https://learn.microsoft.com/en-us/sql/t-sql/statements/create-login-transact-sql)). In each database the application uses, a database user maps to the login with `CREATE USER ... FOR LOGIN ...` ([Contained User Access to Contained Databases](https://learn.microsoft.com/en-us/sql/relational-databases/security/contained-database-users-making-your-database-portable)).

A login can also be created for an AD group, and then every member can connect ([Linux tutorial](https://learn.microsoft.com/en-us/sql/linux/security/authentication/active-directory-tutorial)). On SQL Server on Linux, the `CONTOSO.COM\Username` form is not supported (use `CONTOSO\Username`), and logins from Domain Local groups are not supported (use Global security groups) ([Linux tutorial](https://learn.microsoft.com/en-us/sql/linux/security/authentication/active-directory-tutorial)).

On Amazon RDS for SQL Server, Windows logins can be created only with T-SQL, not with SSMS, and the creator needs `ALTER ANY LOGIN`. AWS also warns that if an AD user is deleted and recreated with the same `sAMAccountName`, the new login can pick up the old SID because of OS-level caching on the RDS host ([Setting up Windows Authentication for SQL Server DB instances](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_SQLServerWinAuth.SettingUp.html)).

## 3. Keytabs and encryption types

### What a keytab is

A keytab stores long-term keys for one or more principals. Each entry holds a timestamp, a principal name, a key version number (kvno), an encryption type and the key itself ([MIT: keytab](https://web.mit.edu/kerberos/krb5-latest/doc/basic/keytab_def.html)). Microsoft adds that the key is not the password itself, and that Linux server processes cannot run under a Windows service account, so they use a keytab instead ([Understanding Active Directory Authentication for SQL Server on Linux and Containers](https://learn.microsoft.com/en-us/sql/linux/security/authentication/understand-active-directory)).

Two different keytabs can be involved. SQL Server needs a server-side keytab (`mssql.keytab`) only when it runs on Linux; it reads the SPN key from that file to decrypt the client's ticket ([Understanding AD authentication for SQL Server on Linux](https://learn.microsoft.com/en-us/sql/linux/security/authentication/understand-active-directory)). A client-side keytab is one of the documented ways for the application to get its TGT ([ODBC integrated authentication](https://learn.microsoft.com/en-us/sql/connect/odbc/linux-mac/using-integrated-authentication)). Microsoft's SQL docs do not name a tool for building the client-side keytab.

### Tools Microsoft documents for SQL Server on Linux

adutil is Microsoft's CLI for managing AD for SQL Server on Linux and containers. Its support is limited to SQL Server use cases. It must run on a host that is already joined to AD, after `kinit` as an account allowed to create accounts and SPNs. Packages exist for RHEL 8, 9 and 10, Ubuntu 20.04, 22.04 and 24.04, and SLES 12 and 15, and installation requires accepting a EULA ([Introduction to adutil](https://learn.microsoft.com/en-us/sql/linux/security/authentication/adutil-introduction)). Microsoft calls `mssql-conf setup-ad-keytab` (which uses adutil) the preferred way to create the SQL Server service keytab. The manual route is `adutil keytab createauto` for the SPN entries plus `adutil keytab create` for the account principal. The `-e`/`--enctype` option sets the encryption types. Microsoft suggests `aes256-cts-hmac-sha1-96` and `aes128-cts-hmac-sha1-96` and says to avoid `arcfour-hmac` (RC4) in production ([Configure AD authentication with SQL Server on Linux using adutil](https://learn.microsoft.com/en-us/sql/linux/security/authentication/adutil-tutorial)). adutil can use LDAPS when `useLdaps` is set to `true` in `adutil.json` ([Introduction to adutil](https://learn.microsoft.com/en-us/sql/linux/security/authentication/adutil-introduction)).

ktpass runs on Windows. It configures the principal name for a host or service in AD DS and writes a `.keytab` file for non-Windows Kerberos services. `/crypto` accepts `DES-CBC-CRC`, `DES-CBC-MD5`, `RC4-HMAC-NT`, `AES256-SHA1` (AES256-CTS-HMAC-SHA1-96), `AES128-SHA1` and `All`. Microsoft says to always pass `/crypto` because the defaults are based on older MIT versions. `/ptype KRB5_NT_PRINCIPAL` is the recommended principal type, `/kvno` defaults to 1, `/princ` is case-sensitive, and `-setpass` sets the user's password ([ktpass](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/ktpass)). The Linux tutorial's ktpass commands write both `aes256-sha1` and `rc4-hmac-nt` entries for each SPN and for the account. It notes that RC4 is older and that the keytab can be built with AES only ([Linux tutorial](https://learn.microsoft.com/en-us/sql/linux/security/authentication/active-directory-tutorial)).

### Tools Microsoft does not document

MIT's ktutil reads, writes and edits keytab entries. `add_entry {-key|-password} -p principal -k kvno [-e enctype] [-f|-s salt]` adds an entry; `-f` fetches the salt from the KDC; `write_kt` saves the file ([MIT: ktutil](https://web.mit.edu/kerberos/krb5-latest/doc/admin/admin_commands/ktutil.html)). A GitHub code search of the public [MicrosoftDocs/sql-docs](https://github.com/MicrosoftDocs/sql-docs) repository on 2026-11-23 returned no matches for `ktutil`, while the same search did find `adutil` and `ktpass` pages. So Microsoft's SQL docs do not appear to reference ktutil, though a code search is not proof of absence.

msktutil is a third-party GPL-2.0 tool. It creates computer or service accounts in AD, creates keytabs, adds and removes principals, and changes the account password ([msktutil repository](https://github.com/msktutil/msktutil)). It manages machine accounts by default; `--use-service-account` switches it to user-type service accounts, and `--enctypes` writes the `msDS-SupportedEncryptionTypes` bitmask ([msktutil man page](https://github.com/msktutil/msktutil/blob/master/msktutil.1.in)). The latest GitHub release is 1.2.2, published 2024-09-11 ([releases](https://github.com/msktutil/msktutil/releases)). The same code search found no `msktutil` reference in Microsoft's SQL docs.

### AES versus RC4 and msDS-SupportedEncryptionTypes

`msDS-SupportedEncryptionTypes` is the AD attribute that records which encryption types an account supports. If it is not set, the KDC uses the domain default (`DefaultDomainSupportedEncTypes`). Relevant bits: `0x4` RC4, `0x8` AES128, `0x10` AES256, `0x18` AES128 plus AES256, `0x1C` RC4 plus both AES types ([Detect and Remediate RC4 Usage in Kerberos](https://learn.microsoft.com/en-us/windows-server/security/kerberos/detect-remediate-rc4-kerberos)). Microsoft's SQL on Linux tutorials ask for "This account supports Kerberos AES 128 bit encryption" and "This account supports Kerberos AES 256 bit encryption" to be checked on the SQL Server service account ([Linux tutorial](https://learn.microsoft.com/en-us/sql/linux/security/authentication/active-directory-tutorial), [adutil tutorial](https://learn.microsoft.com/en-us/sql/linux/security/authentication/adutil-tutorial)). Domain controllers log the requested and issued encryption types in events 4768 (TGT) and 4769 (service ticket) ([RC4 article](https://learn.microsoft.com/en-us/windows-server/security/kerberos/detect-remediate-rc4-kerberos)).

On the Linux side, MIT's default `permitted_enctypes` list starts with `aes256-cts-hmac-sha1-96` and still includes `arcfour-hmac-md5`. Since release 1.18 it is also the default for `default_tgs_enctypes` and `default_tkt_enctypes` ([MIT: krb5.conf](https://web.mit.edu/kerberos/krb5-latest/doc/admin/conf_files/krb5_conf.html)).

### Microsoft's RC4 deprecation: current statement and dates

- Microsoft's statement: "Microsoft plans to disable RC4 use as the default assumed supported encryption type for Active Directory domain controllers by the end of the second quarter of 2026" ([RC4 article, ms.date 2026-02-06](https://learn.microsoft.com/en-us/windows-server/security/kerberos/detect-remediate-rc4-kerberos)).
- KB5073381 (CVE-2026-20833) gives the phases. The January 13, 2026 updates added audit events 201, 202, 206 and 207. The April 14, 2026 updates moved DCs to Enforcement with a manual rollback; the KDC then assumes AES-SHA1 only (`0x18`) for accounts with no explicit `msDS-SupportedEncryptionTypes`. In July 2026 the `RC4DefaultDisablementPhase` rollback setting was removed ([KB5073381](https://support.microsoft.com/en-us/topic/how-to-manage-kerberos-kdc-usage-of-rc4-for-service-account-ticket-issuance-changes-related-to-cve-2026-20833-1ebcda33-720a-4da8-93c1-b0496e1910dc)).
- The KB names the most common non-Windows failure: a keytab exported with only RC4 keys for a service account that has AES keys and no `msDS-SupportedEncryptionTypes`. Failures usually show up at the service, not at the KDC. The long-term fix is AES-SHA1. As stopgaps the KB allows `0x24` on the affected account, or, as a last resort, on `DefaultDomainSupportedEncTypes`, which leaves the domain exposed to the CVE ([KB5073381](https://support.microsoft.com/en-us/topic/how-to-manage-kerberos-kdc-usage-of-rc4-for-service-account-ticket-issuance-changes-related-to-cve-2026-20833-1ebcda33-720a-4da8-93c1-b0496e1910dc)).
- Windows Server 2025 domain controllers do not issue RC4 TGTs ([RC4 article](https://learn.microsoft.com/en-us/windows-server/security/kerberos/detect-remediate-rc4-kerberos)), and RC4 use in Kerberos is on the Windows Server 2025 deprecated list ([Features Removed or No Longer Developed in Windows Server](https://learn.microsoft.com/en-us/windows-server/get-started/removed-deprecated-features-windows-server)).
- The Directory Services team wrote: "As of January 2026, there are currently no plans to remove RC4 from Windows." The change removes RC4 from the assumed encryption types; it does not remove the code ([What is going on with RC4 in Kerberos?](https://techcommunity.microsoft.com/blog/askds/what-is-going-on-with-rc4-in-kerberos/4489365)).
- On the standards side, RFC 8429 (October 2018, Best Current Practice) deprecates RC4 and 3DES in Kerberos and moved RFC 4757 (the RC4 enctypes) to Historic ([RFC 8429](https://www.rfc-editor.org/rfc/rfc8429)).

## 4. Key version numbers and password rotation

`kvno` gets a service ticket for each named principal and prints its key version number ([MIT: kvno](https://web.mit.edu/kerberos/krb5-latest/doc/user/user_commands/kvno.html)). Every keytab entry carries a kvno ([MIT: keytab](https://web.mit.edu/kerberos/krb5-latest/doc/basic/keytab_def.html)). Microsoft says the kvno is usually 2, but can be another number if the account's password has been changed several times ([Linux tutorial](https://learn.microsoft.com/en-us/sql/linux/security/authentication/active-directory-tutorial)).

For password changes, the tutorial says: "If the password for the Active Directory account is changed or the password for the account that the SPNs are assigned to is changed, you must update the keytab with the new password and Key Version Number (KVNO)." Microsoft adds that some services rotate passwords automatically, and that rotation policies should be lined up with maintenance windows ([Linux tutorial](https://learn.microsoft.com/en-us/sql/linux/security/authentication/active-directory-tutorial)). The same tutorial creates the SQL Server service account with `-PasswordNeverExpires $true` ([Linux tutorial](https://learn.microsoft.com/en-us/sql/linux/security/authentication/active-directory-tutorial)).

Microsoft's rotation procedure for SQL Server on Linux works like this:

1. Write the new password into the keytab under the next kvno (current plus one) before changing it in AD. If AD is changed first, SQL Server must be restarted.
2. Use `mssql-conf setup-ad-keytab <keytab> <account> --use-next-kvno`, or `adutil keytab createauto ... --kvno <n+1>` and `adutil keytab create ... --kvno <n+1>`.
3. Old and new kvno entries then sit side by side in the keytab.
4. Change the password in AD to the same value. Authentication keeps working without a restart.

Source: [Rotate SQL Server on Linux Keytabs](https://learn.microsoft.com/en-us/sql/linux/security/authentication/rotate-keytabs).

msktutil, the third-party tool, documents a scheduled-rotation pattern. Each update changes the account password and rewrites all keytab entries, and it leaves the previous entries in place so sessions on the old key keep working. `msktutil auto-update` only acts when `pwdLastSet` is at least 30 days old (configurable with `--auto-update-interval`) and is meant to run from a daily cron job. The man page notes that many domains expire passwords after 90 days, which breaks a keytab that is never updated. Its two options are a daily cron job (preferred) or disabling password expiry on the account (not preferred). For `--use-service-account`, the domain's minimum password age (often 1 day) also applies ([msktutil man page](https://github.com/msktutil/msktutil/blob/master/msktutil.1.in)).

No Microsoft or MIT page found describes rotating a client-side keytab mounted into containers. That part is not confirmed.

## 5. Group managed service accounts (gMSA)

A gMSA is a domain account that several servers can share. Windows manages its password, and SPN management is simplified. The Key Distribution Service (`kdssvc.dll`) supplies the key from which the DC computes the password, and member hosts get the current and previous password from a DC ([Group Managed Service Accounts overview](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/group-managed-service-accounts/group-managed-service-accounts/group-managed-service-accounts-overview)). Creating one needs a KDS root key and `New-ADServiceAccount ... -PrincipalsAllowedToRetrieveManagedPassword <group>`. The password interval defaults to 30 days and can be set only when the account is created ([Manage Group Managed Service Accounts](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/group-managed-service-accounts/group-managed-service-accounts/manage-group-managed-service-accounts)). gMSAs do not apply to Windows versions before Windows Server 2012 ([overview](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/group-managed-service-accounts/group-managed-service-accounts/group-managed-service-accounts-overview)).

### Why Linux cannot use a gMSA natively

Microsoft's principal table describes the gMSA as usable by "Any Windows Server domain-joined server", where "the domain controller manages, and the host retrieves" the password. The listed consumers are all Windows mechanisms: products that support the sMSA APIs, services configured through the Service Control Manager, IIS application pools, and Task Scheduler. The host-side commands (`Install-ADServiceAccount`, `Test-ADServiceAccount`) are Windows PowerShell cmdlets ([Manage gMSAs](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/group-managed-service-accounts/group-managed-service-accounts/manage-group-managed-service-accounts)). The same page says gMSA use is "scoped to any machine that is able to use the Lightweight Directory Access Protocol (LDAP) to retrieve the gMSA's credentials" ([same page](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/group-managed-service-accounts/group-managed-service-accounts/manage-group-managed-service-accounts)). Linux has no Microsoft component that does that retrieval.

### What Microsoft documents for containers

Microsoft documents gMSA for Windows containers on Windows Server 2019, 2022 and 2025. A Windows host process, `ccg.exe`, retrieves the gMSA password, optionally through a plug-in on a non-domain-joined host, and hands it to the container ([Create gMSAs for Windows containers](https://learn.microsoft.com/en-us/virtualization/windowscontainers/manage-containers/manage-serviceaccounts)). AKS enables gMSA "on your Windows Server nodes" ([AKS gMSA](https://learn.microsoft.com/en-us/azure/aks/use-group-managed-service-accounts)). Microsoft's on-premises container support policy covers gMSA for Windows Server containers. In its Kubernetes section it says Microsoft supports only Windows nodes and does not support "Setting up and configuring Linux nodes" or "Linux containers" ([Support policy for Windows containers](https://learn.microsoft.com/en-us/troubleshoot/windows-server/containers/support-for-windows-containers-docker-on-premises-scenarios)). No Microsoft page found documents gMSA support for Linux hosts or Linux containers; Linux support from Microsoft is not confirmed.

### What AWS documents

AWS documents gMSA for Linux containers on Amazon ECS (EC2 and Fargate). It works through the `credentials-fetcher` daemon on the host, which gets the gMSA credentials from a DC and passes them to the container. The AWS page states that "Linux containers can't be domain-joined". Creating the CredSpec still requires the CredentialSpec PowerShell module on a domain-joined Windows computer ([Using gMSA for EC2 Linux containers on Amazon ECS](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/linux-gmsa.html)). The credentials-fetcher README describes "a Linux daemon that retrieves gMSA credentials from Active Directory over LDAP" and "creates and refreshes kerberos tickets". It is licensed Apache-2.0 and lists Amazon Linux 2023 and Fedora 41+ ([aws/credentials-fetcher](https://github.com/aws/credentials-fetcher)), while the ECS page says Fedora 37 and later ([ECS page](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/linux-gmsa.html)). The latest release is v2.0.3, published 2026-06-12 ([releases](https://github.com/aws/credentials-fetcher/releases)). The README does not mention Kubernetes, EKS or OpenShift, so use on OpenShift is not confirmed.

## 6. Amazon RDS for SQL Server Windows authentication

### With AWS Managed Microsoft AD

- RDS supports only AWS Managed Microsoft AD for this mode; AD Connector is not supported. On-premises or self-hosted AD connects through a one-way or two-way forest trust. The master user always uses SQL authentication ([Working with AWS Managed Active Directory with RDS for SQL Server](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_SQLServerWinAuth.html)).
- Windows authentication is supported only for DB instances in a VPC. Directory DNS names longer than 47 characters are not supported. Enabling Kerberos on an existing instance requires a reboot ([Setting up Windows Authentication](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_SQLServerWinAuth.SettingUp.html)).
- For Kerberos, clients connect to the customer-specified host name followed by the domain FQDN, for example `ad-test.corp-ad.company.com`, and not to the `*.rds.amazonaws.com` endpoint ([Creating the endpoint for Kerberos authentication](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_SQLServerWinAuth.KerberosEndpoint.html)).

### With self-managed AD

- AWS says the instance joins your own AD directly, with "no need for complex intermediary domains or forest trusts". Both NTLM and Kerberos are supported, in "all commercial AWS Regions and AWS GovCloud (US) Regions" ([Working with self-managed Active Directory](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_SQLServer_SelfManagedActiveDirectory.html)).
- NTLM: connect to the RDS endpoint, or to the listener endpoint for Multi-AZ ([Joining your DB instance to self-managed AD](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_SQLServer_SelfManagedActiveDirectory.Joining.html)).
- Kerberos: connect to the Kerberos endpoint, a CNAME in the AD DNS zone. For `<name>.<hash>.<region>.rds.amazonaws.com`, the Kerberos endpoint is `<name>.<hash>.<region>.awsrds.<your AD FQDN>`. AWS says it "is automatically updated to point to the new SPN of the primary" after failover. If a remote client connecting to the CNAME still gets NTLM, AWS says to check that the required ports are open ([same page](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_SQLServer_SelfManagedActiveDirectory.Joining.html)).
- Requirements include a domain functional level of Windows Server 2008 R2 or higher, no single-label domain names, an AD FQDN of 47 characters or fewer, and a domain service account with join rights on the target OU. For Kerberos that account also needs Write SPN and read permissions in DNS. The instance must reach the DCs on 53, 88, 464 and 389 (TCP/UDP), and on 135, 445, 636 and 49152 to 65535 (TCP) ([Requirements](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_SQLServer_SelfManagedActiveDirectory.Requirements.html)).
- Other limits: instances sync time with AWS's NTP service, not the AD time source; linked servers between instances can use only SQL authentication; GPOs are not applied to the instance ([self-managed AD page](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_SQLServer_SelfManagedActiveDirectory.html)).

### Linux clients and Region availability

AWS's RDS for SQL Server connection page describes only a domain-joined Windows computer using SSMS ([Connecting to SQL Server with Windows authentication](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_SQLServerWinAuth.Connecting.html)). No AWS page found gives Linux client steps for RDS for SQL Server, so that part is not confirmed.

AWS's Kerberos Region table marks RDS for SQL Server as "Not available" in Asia Pacific (Malaysia), Asia Pacific (New Zealand), Asia Pacific (Taipei), Asia Pacific (Thailand), Canada West (Calgary), Israel (Tel Aviv) and Mexico (Central) ([Supported Regions and DB engines for Kerberos authentication](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.RDS_Fea_Regions_DB-eng.Feature.KerberosAuthentication.html)). That conflicts with the "all commercial Regions" statement on the self-managed AD page. Which statement covers self-managed AD in those Regions is not confirmed.

## 7. NTLM deprecation status

- In June 2024 Microsoft announced: "All versions of NTLM, including LANMAN, NTLMv1, and NTLMv2, are no longer under active feature development and are deprecated". Microsoft recommends replacing NTLM calls with Negotiate. A November 2024 update notes that "NTLMv1 is removed starting in Windows 11, version 24H2 and Windows Server 2025" ([Deprecated features in the Windows client](https://learn.microsoft.com/en-us/windows/whats-new/deprecated-features)).
- In Windows Server 2025, NTLMv1 is removed. LANMAN and NTLMv2 are deprecated, and "NTLMv2 will continue to work but will be removed from Windows Server in a future release." The Windows Server 2028 (Preview) deprecated and removed lists did not include NTLM when checked on 2026-11-23 ([Features Removed or No Longer Developed in Windows Server](https://learn.microsoft.com/en-us/windows-server/get-started/removed-deprecated-features-windows-server)).
- The October 13, 2026 security update switches `BlockNtlmv1SSO` to Enforce by default when the value is absent, which blocks NTLMv1-derived single sign-on such as MS-CHAPv2 ([Manage NTLMv1 changes](https://support.microsoft.com/en-us/topic/upcoming-changes-to-ntlmv1-in-windows-11-version-24h2-and-windows-server-2025-c0554217-cdbc-420f-b47c-e02b2db49b2e)).
- The roadmap to turning NTLM off by default (blog published Jan 29, 2026, updated Oct 22, 2026) has three phases. Phase 1, available now: enhanced NTLM auditing on Windows Server 2025 and Windows 11 24H2 and later. Phase 2, second half of 2026: fixes for the main blockers. Phase 3, next major Windows Server release: "Network NTLM will be disabled by default", can be re-enabled by policy, and is not removed yet ([Advancing Windows security: Disabling NTLM by default](https://techcommunity.microsoft.com/blog/windows-itpro-blog/advancing-windows-security-disabling-ntlm-by-default/4489526)). The FAQ says the target is "the next major release of Server and Client (subject to change)", that "In-market versions of Windows client and Windows Server will continue to support NTLM", and that non-Windows clients "will continue to work if they can negotiate Kerberos with the Windows server" ([Retiring NTLM: FAQ](https://techcommunity.microsoft.com/blog/windows-itpro-blog/retiring-ntlm-frequently-asked-questions/4550522)). No release name or date is given.
- From November 10, 2026, Windows clients offer NTLM signing by default. In May 2027 Windows servers begin requiring it by default, and "Clients that do not offer signing will be rejected by secure servers". The change covers Windows Server 2016 through 2025, and Microsoft lists compatibility problems with some third-party NTLM implementations ([NTLM signing security improvements](https://support.microsoft.com/en-us/servicing/os/windows/docs/2026/11/kb5135557-ntlm-signing-security-improvements)).

### What this means for Linux drivers

- The Microsoft ODBC driver on Linux has no NTLM path at all: "If Kerberos authentication fails, the driver on Linux and macOS doesn't fall back to NT LAN Manager (NTLM) authentication" ([Use Integrated Authentication, ODBC](https://learn.microsoft.com/en-us/sql/connect/odbc/linux-mac/using-integrated-authentication)).
- The Microsoft JDBC driver supports `authenticationScheme=NTLM` from version 7.4. It requires the domain (`domain`/`domainName`), the Windows user name and the password in the connection properties ([JDBC connection properties](https://learn.microsoft.com/en-us/sql/connect/jdbc/setting-the-connection-properties)).
- Whether the May 2027 signing requirement applies to NTLM inside SQL Server's TDS login, and whether a given Linux driver offers NTLM signing, is not confirmed.

## 8. Alternatives that are not Windows authentication

SQL Server 2022 and later supports Microsoft Entra ID authentication "on both Windows and Linux on-premises". The host and SQL Server register with Azure Arc, which needs the Azure Arc agent and the Azure extension for SQL Server. A manual setup without Arc exists for Windows only. Failover cluster instances are not supported. Methods include default, password, integrated, interactive with MFA, service principal, managed identity and access token. Entra logins use `CREATE LOGIN ... FROM EXTERNAL PROVIDER` ([Microsoft Entra authentication for SQL Server overview](https://learn.microsoft.com/en-us/sql/relational-databases/security/authentication-access/azure-ad-authentication-sql-server-overview), [CREATE LOGIN](https://learn.microsoft.com/en-us/sql/t-sql/statements/create-login-transact-sql)).

Linux ODBC driver support for Entra ([Using Microsoft Entra ID with the ODBC Driver](https://learn.microsoft.com/en-us/sql/connect/odbc/using-azure-active-directory)):

- Access tokens work on all platforms from Driver 17.
- `ActiveDirectoryServicePrincipal` is available from 17.7. Microsoft recommends it when managed identity is not available, for example outside Azure, and prefers a client certificate over a secret.
- `ActiveDirectoryMsi` (managed identity) is supported, and from 18.3 also on Azure Arc.
- `ActiveDirectoryIntegrated` works on Linux from 17.6 for federated accounts and needs a Kerberos ticket.
- `ActiveDirectoryInteractive` is Windows only.
- `ActiveDirectoryPassword` is deprecated.
- Conditional Access scenarios are not supported on Linux.

Contained database users authenticate at the database level and support both Windows and SQL Server authentication. Setup: enable `contained database authentication` with `sp_configure`, set `CONTAINMENT = PARTIAL` on the database, then `CREATE USER <name> WITH PASSWORD = '...'`. The connection string must name the database. Microsoft notes that database owners and users with `ALTER ANY USER` can then grant access to the database ([Contained User Access to Contained Databases](https://learn.microsoft.com/en-us/sql/relational-databases/security/contained-database-users-making-your-database-portable)). A server-level SQL login is `CREATE LOGIN <name> WITH PASSWORD = '...'`, and passwords must be 8 to 128 characters ([CREATE LOGIN](https://learn.microsoft.com/en-us/sql/t-sql/statements/create-login-transact-sql)).

## 9. Network ports and clock skew

| Port | Protocol | Service | Source |
| --- | --- | --- | --- |
| 88 | TCP and UDP | Kerberos KDC | [Windows port requirements](https://learn.microsoft.com/en-us/troubleshoot/windows-server/networking/service-overview-and-network-port-requirements), [RFC 4120 section 7.2](https://www.rfc-editor.org/rfc/rfc4120) |
| 464 | TCP and UDP | Kerberos password change and set (kpasswd) | [Windows port requirements](https://learn.microsoft.com/en-us/troubleshoot/windows-server/networking/service-overview-and-network-port-requirements), [RFC 3244](https://www.rfc-editor.org/rfc/rfc3244) |
| 53 | TCP and UDP | DNS | [Windows port requirements](https://learn.microsoft.com/en-us/troubleshoot/windows-server/networking/service-overview-and-network-port-requirements) |
| 389 | TCP (UDP for DC locator) | LDAP | [Windows port requirements](https://learn.microsoft.com/en-us/troubleshoot/windows-server/networking/service-overview-and-network-port-requirements) |
| 636 | TCP | LDAP over TLS | [Windows port requirements](https://learn.microsoft.com/en-us/troubleshoot/windows-server/networking/service-overview-and-network-port-requirements) |
| 1433 | TCP | SQL Server default instance | [Configure the Windows Firewall to Allow SQL Server Access](https://learn.microsoft.com/en-us/sql/sql-server/install/configure-the-windows-firewall-to-allow-sql-server-access) |
| 1434 | UDP | SQL Server Browser | [same page](https://learn.microsoft.com/en-us/sql/sql-server/install/configure-the-windows-firewall-to-allow-sql-server-access) |

- **88.** RFC 4120 says KDCs must accept both UDP and TCP and should listen on port 88 ([RFC 4120](https://www.rfc-editor.org/rfc/rfc4120)). MIT also uses 88 as the default KDC port ([MIT: Realm configuration decisions](https://web.mit.edu/kerberos/krb5-latest/doc/admin/realm_config.html)).
- **464.** Port 464 carries the change-password and set-password protocol ([RFC 3244](https://www.rfc-editor.org/rfc/rfc3244)). A client only uses it when something changes a password, such as the rotation tools in section 4.
- **389 and 636.** LDAP is needed only by tools that query the directory. adutil, msktutil and credentials-fetcher all talk LDAP to AD ([adutil intro](https://learn.microsoft.com/en-us/sql/linux/security/authentication/adutil-introduction), [msktutil man page](https://github.com/msktutil/msktutil/blob/master/msktutil.1.in), [credentials-fetcher](https://github.com/aws/credentials-fetcher)).
- **53.** In a gMSA troubleshooting note, Microsoft says large SRV responses for DC lookups can exceed 512 bytes over UDP and must be retried over TCP 53 ([AKS gMSA troubleshooting](https://learn.microsoft.com/en-us/azure/aks/use-group-managed-service-accounts)).
- **UDP to DCs.** Microsoft's SQL on Linux tutorial suggests `udp_preference_limit = 0` in `krb5.conf` to skip UDP calls to domain controllers ([Linux tutorial](https://learn.microsoft.com/en-us/sql/linux/security/authentication/active-directory-tutorial)).
- **1433 and 1434.** Named instances use dynamic ports by default. The Browser service on UDP 1434 tells clients the port, and it is not needed if the client connects to a specific port ([SQL Server firewall page](https://learn.microsoft.com/en-us/sql/sql-server/install/configure-the-windows-firewall-to-allow-sql-server-access)). Kerberos SPNs need a static port; with dynamic ports the registered SPN becomes invalid when the port changes ([Kerberos Configuration Manager article](https://learn.microsoft.com/en-us/troubleshoot/sql/database-engine/connect/using-kerberosmngr-sqlserver)).

For clock skew, the Default Domain Policy sets "Maximum tolerance for computer clock synchronization" to 5 minutes, and that is also the effective default on domain controllers ([Maximum tolerance for computer clock synchronization](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/security-policy-settings/maximum-tolerance-for-computer-clock-synchronization)). MIT's `clockskew` default is 300 seconds ([MIT: krb5.conf](https://web.mit.edu/kerberos/krb5-latest/doc/admin/conf_files/krb5_conf.html)). RFC 4120 returns `KRB_AP_ERR_SKEW` when the skew exceeds the allowed value, giving "e.g., 5 minutes" as the example ([RFC 4120](https://www.rfc-editor.org/rfc/rfc4120)). Microsoft's SQL troubleshooting guide states that clocks cannot differ by more than five minutes ([consistent authentication issues](https://learn.microsoft.com/en-us/troubleshoot/sql/database-engine/connect/consistent-authentication-connectivity-issues)).

## 10. krb5.conf essentials for AD and DNS pitfalls

Microsoft's sample client configuration for the Linux ODBC driver ([Use Integrated Authentication, ODBC](https://learn.microsoft.com/en-us/sql/connect/odbc/linux-mac/using-integrated-authentication)):

```ini
[libdefaults]
default_realm = YYYY.CORP.CONTOSO.COM
dns_lookup_realm = false
dns_lookup_kdc = true
ticket_lifetime = 24h
forwardable = yes

[domain_realm]
.yyyy.corp.contoso.com = YYYY.CORP.CONTOSO.COM
.zzzz.corp.contoso.com = ZZZZ.CORP.CONTOSO.COM
```

The same page says principal names are case-sensitive, and that `dns_lookup_kdc=true` needs DNS servers that can resolve the realm's KDC records ([same page](https://learn.microsoft.com/en-us/sql/connect/odbc/linux-mac/using-integrated-authentication)). Microsoft's SQL on Linux sample instead lists `kdc` and `admin_server` under `[realms]`. If `[domain_realm]` is missing, SQL Server on Linux assumes `contoso.com` maps to `CONTOSO.COM` ([Understanding AD authentication for SQL Server on Linux](https://learn.microsoft.com/en-us/sql/linux/security/authentication/understand-active-directory)).

What each setting means, from MIT ([MIT: krb5.conf](https://web.mit.edu/kerberos/krb5-latest/doc/admin/conf_files/krb5_conf.html) unless noted):

- **`default_realm`.** Names the client's default realm. Realm names are case-sensitive, and uppercase is "strongly encouraged" ([RFC 4120 section 7.2.3.1](https://www.rfc-editor.org/rfc/rfc4120)). MIT's convention is the DNS domain name in upper case, for example `EXAMPLE.COM` ([MIT: Realm configuration decisions](https://web.mit.edu/kerberos/krb5-latest/doc/admin/realm_config.html)). Microsoft's samples follow the same pattern.
- **`dns_lookup_kdc`.** Uses DNS SRV records to find KDCs that are not listed in `krb5.conf`. The current MIT page does not state a default for this setting.
- **`dns_lookup_realm`.** Uses DNS TXT records for host-to-realm mapping. Default false.
- **`rdns`.** "If this flag is true, reverse name lookup will be used in addition to forward name lookup" to canonicalize host names in service principal names. Default true, and no effect when `dns_canonicalize_hostname` is false.
- **`dns_canonicalize_hostname`.** Whether DNS lookups canonicalize host names for service principals. Values are true (the default), false, or `fallback` (from release 1.18, canonicalize only if the original name is not found). Setting false "can improve security by reducing reliance on DNS, but means that short hostnames will not be canonicalized".

### Why reverse DNS and CNAMEs break SQL Server SPN matching

1. The client builds the SPN from the server name it was given. SQL Server clients use `MSSQLSvc/<host>:<port>`, which Microsoft calls "a hardcoded format in most SQL Server clients" ([Understanding AD authentication for SQL Server on Linux](https://learn.microsoft.com/en-us/sql/linux/security/authentication/understand-active-directory)).
2. With the MIT defaults, the Kerberos library then rewrites the host part. Forward resolution follows CNAME records, and when `rdns` is true the library also does a reverse lookup of the first returned address ([MIT: Principal names and DNS](https://web.mit.edu/kerberos/krb5-latest/doc/admin/princ_dns.html)). A CNAME alias or a PTR record that returns a different name therefore changes the SPN the client asks for.
3. If that name does not match a registered SPN, the KDC has no service principal to issue a ticket for. If it matches an SPN on another account, authentication fails ([SPN article](https://learn.microsoft.com/en-us/sql/database-engine/configure-windows/register-a-service-principal-name-for-kerberos-connections)). The error text Microsoft shows for a missing SPN is "Server not found in Kerberos database" ([Linux tutorial](https://learn.microsoft.com/en-us/sql/linux/security/authentication/active-directory-tutorial), [JDBC connection properties](https://learn.microsoft.com/en-us/sql/connect/jdbc/setting-the-connection-properties)).
4. On Windows clients Microsoft lists HOSTS file entries, SQL aliases and disjoint DNS namespaces as causes of unexpected SPNs. Those cases fall back to NTLM or end in an SSPI failure ([consistent authentication issues](https://learn.microsoft.com/en-us/troubleshoot/sql/database-engine/connect/consistent-authentication-connectivity-issues)). The Linux ODBC driver has no NTLM fallback, so the connection fails instead ([ODBC integrated authentication](https://learn.microsoft.com/en-us/sql/connect/odbc/linux-mac/using-integrated-authentication)).
5. The IETF position is that implementations "MUST NOT use insecure DNS queries to canonicalize the hostname components of the service principal names". RFC 4120 also notes that many implementations do it anyway and that they are inconsistent about reverse resolution ([RFC 4120 section 1.3](https://www.rfc-editor.org/rfc/rfc4120)).

Documented ways to avoid the problem:

- If forward and reverse DNS do not match, MIT recommends `rdns = false` on client machines ([MIT: Principal names and DNS](https://web.mit.edu/kerberos/krb5-latest/doc/admin/princ_dns.html)).
- Microsoft recommends connecting with the server's FQDN or putting the SPN in the connection string ([consistent authentication issues](https://learn.microsoft.com/en-us/troubleshoot/sql/database-engine/connect/consistent-authentication-connectivity-issues)). The Linux ODBC driver accepts `ServerSPN` ([ODBC integrated authentication](https://learn.microsoft.com/en-us/sql/connect/odbc/linux-mac/using-integrated-authentication)). JDBC with `JavaKerberos` needs the FQDN in `serverName` or `serverSpn` ([JDBC connection properties](https://learn.microsoft.com/en-us/sql/connect/jdbc/setting-the-connection-properties)).
- For SQL Server in containers, clients must connect with the exact name and port in the keytab's SPN, for example `sqlcontainer.domain.com,8000` for `MSSQLSvc/sqlcontainer.domain.com:8000` ([Understanding AD authentication for SQL Server on Linux](https://learn.microsoft.com/en-us/sql/linux/security/authentication/understand-active-directory)).
- The RDS self-managed AD Kerberos endpoint is itself a CNAME that AWS says points to the primary's SPN ([Joining your DB instance to self-managed AD](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_SQLServer_SelfManagedActiveDirectory.Joining.html)). How it behaves with `dns_canonicalize_hostname = false` or `rdns = false` on a Linux client is not documented by AWS and is not confirmed.

## Items not confirmed

1. Whether Kerberos Configuration Manager can inspect SQL Server on Linux. The download page lists Windows operating systems only ([Download Center](https://www.microsoft.com/download/details.aspx?id=39046)).
2. Whether any Microsoft page mentions ktutil or msktutil. A code search of the public sql-docs repository found none, but that is not proof of absence ([MicrosoftDocs/sql-docs](https://github.com/MicrosoftDocs/sql-docs)).
3. Microsoft support for gMSA on Linux hosts or Linux containers. No Microsoft page found.
4. Whether credentials-fetcher works on Kubernetes or OpenShift. The README does not say ([aws/credentials-fetcher](https://github.com/aws/credentials-fetcher)).
5. Linux client steps for RDS for SQL Server Windows authentication. No AWS page found.
6. Which of AWS's two Region statements applies to Kerberos with self-managed AD in Regions such as Asia Pacific (Thailand).
7. How the RDS `awsrds` CNAME behaves with `rdns = false` or `dns_canonicalize_hostname = false` on a Linux client.
8. Whether the May 2027 NTLM signing requirement affects NTLM inside SQL Server's TDS login from Linux drivers such as JDBC.
9. The release name and date of the "next major" Windows Server release that turns network NTLM off by default.
10. The default value of `dns_lookup_kdc`, which the current MIT page does not state.
11. An explicit Microsoft rule that the AD realm equals the uppercase DNS domain name. Microsoft's samples follow this pattern, and the SQL on Linux page states the mapping it assumes.
12. Guidance from Microsoft or MIT on rotating client-side keytabs mounted into containers.

## Beginner reading list

Each URL was opened on 2026-11-23 and returned the page title shown.

1. [Kerberos V5 concepts](https://web.mit.edu/kerberos/krb5-latest/doc/basic/index.html) (MIT Kerberos Documentation). Short pages on credential caches, keytabs and other core terms.
2. [For users](https://web.mit.edu/kerberos/krb5-latest/doc/user/index.html) (MIT Kerberos Documentation). `kinit`, `klist`, `kvno` and ticket management from the client side.
3. [For administrators](https://web.mit.edu/kerberos/krb5-latest/doc/admin/index.html) (MIT Kerberos Documentation). Includes the `krb5.conf` reference and "Principal names and DNS".
4. [Kerberos authentication overview in Windows Server](https://learn.microsoft.com/en-us/windows-server/security/kerberos/kerberos-authentication-overview) (Microsoft Learn). How the AD KDC issues TGTs and service tickets, and how Windows interoperates with standard Kerberos.
5. [Register a Service Principal Name for Kerberos Connections - SQL Server](https://learn.microsoft.com/en-us/sql/database-engine/configure-windows/register-a-service-principal-name-for-kerberos-connections) (Microsoft Learn). SPN formats, setspn, and the `auth_scheme` check.
6. [Understanding Active Directory Authentication for SQL Server on Linux and Containers - SQL Server](https://learn.microsoft.com/en-us/sql/linux/security/authentication/understand-active-directory) (Microsoft Learn). Keytabs, `krb5.conf`, and the ticket flow drawn step by step.
7. [Use Integrated Authentication - ODBC Driver for SQL Server](https://learn.microsoft.com/en-us/sql/connect/odbc/linux-mac/using-integrated-authentication) (Microsoft Learn). The Linux client side: tickets, keytabs, `ServerSPN`, and no NTLM fallback.
8. [Group Managed Service Accounts overview](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/group-managed-service-accounts/group-managed-service-accounts/group-managed-service-accounts-overview) (Microsoft Learn). The older URL `/windows-server/security/group-managed-service-accounts/group-managed-service-accounts-overview` redirects here.
9. [Working with AWS Managed Active Directory with RDS for SQL Server](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_SQLServerWinAuth.html) (Amazon RDS User Guide).
10. [Working with self-managed Active Directory with an Amazon RDS for SQL Server DB instance](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_SQLServer_SelfManagedActiveDirectory.html) (Amazon RDS User Guide).
