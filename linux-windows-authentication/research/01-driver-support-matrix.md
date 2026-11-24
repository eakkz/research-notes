# Driver support matrix: Windows Authentication to SQL Server from Linux

Sources checked on 2026-11-23. Every claim links to the source that owns it. Reference labels such as [odbc-ia] resolve to the URLs listed under "Sources" at the end. "Not confirmed" means no primary source read for this note states the fact.

Driver versions in the repositories at the time of reading: mssql-jdbc v13.6.2 ([jdbc-releases]), Microsoft.Data.SqlClient v7.1.2 ([sqlclient-releases]), go-mssqldb v1.12.0 ([go-releases]), tedious v20.4.3 ([tedious-releases]), FreeTDS v1.5.22 ([freetds-releases]).

## Matrix

| Driver | Kerberos on Linux | Needs ticket cache or reads keytab | NTLM | Notes |
| --- | --- | --- | --- | --- |
| Microsoft ODBC Driver 17 and 18 (used by pyodbc) | Yes. `Trusted_Connection=yes`, through GSSAPI and Kerberos 5 libraries ([odbc-ia]) | Needs a credential cache. Get a TGT with `kinit` (password or keytab) or a Kerberos PAM login; the driver does not renew tickets ([odbc-ia]). No keytab keyword ([odbc-kw]) | No documented NTLM mode on Linux. If Kerberos fails, the driver on Linux and macOS does not fall back to NTLM ([odbc-ia]) | `ServerSPN` works on Linux in current 17 and 18 releases ([odbc-ia]). pyodbc passes the connection string to the driver manager unchanged ([pyodbc-mod]) |
| Microsoft JDBC Driver (mssql-jdbc) | Yes. `integratedSecurity=true;authenticationScheme=JavaKerberos`, pure Java Kerberos, since 4.0 ([jdbc-kerb]) | Default JAAS config reads the ticket cache (`useTicketCache=true`) ([jdbc-kerb], [jdbc-jaassrc]). Can read a keytab directly through a JAAS `Krb5LoginModule` entry with `useKeyTab=true`, without running `kinit` ([jdbc-kerbsrc], [krb5lm], [jdbc-sample]). Also accepts `userName` and `password` as the Kerberos principal (6.2+) ([jdbc-kerb]) | Yes. `authenticationScheme=NTLM`, listed as version 7.4+ ([jdbc-props]); first shipped in 7.3.1 preview ([jdbc-changelog]). NTLMv2 only ([jdbc-ntlm]) | FQDN required in `serverName` or `serverSpn` for JavaKerberos ([jdbc-props]). From 13.6, `java.security.auth.login.config` must be a local path or `file:` URI ([jdbc-kerb]) |
| Microsoft.Data.SqlClient (.NET 6/8) | Yes. `Integrated Security=true`; on Linux it uses managed networking and .NET `NegotiateAuthentication` ([sqlclient-auth]) | Needs valid credentials or a ticket cache; obtain a ticket with `kinit` before starting the app ([sqlclient-auth]). No keytab keyword (not confirmed that one exists; none found in sources read) | Not native. The 7.0.0 release notes say the driver "does not provide natively" NTLM username and password authentication; the public `SspiContextProvider` (7.0.0) lets an app supply it ([sqlclient-700]) | `User ID` and `Password` are ignored when integrated security is on ([sqlclient-auth]). SqlClient 6.0+ targets .NET 8+; 5.2 targets .NET 6+ ([sqlclient-600], [sqlclient-520]) |
| go-mssqldb | Yes. `authenticator=krb5` plus a blank import of `github.com/microsoft/go-mssqldb/integratedauth/krb5` ([go-readme]). Pure Go Kerberos library ([go-krb5lib]) | Either. Keytab (`krb5-keytabfile`), credential cache (`krb5-credcachefile`), or user and password ([go-readme], [go-krb5src]) | Yes. Default provider on non-Windows is `ntlm`; triggered by `user id` in `DOMAIN\User` form ([go-readme], [go-authunix], [go-ntlmsrc]) | README names the cache variable `KRBCCNAME`; the source reads `KRB5CCNAME` ([go-readme], [go-krb5src]) |
| Node.js tedious | No. Valid authentication types are `default`, `ntlm`, `token-credential` and the `azure-active-directory-*` types ([tedious-conn]). A contributor stated "we do not have a plan for supporting Kerberos within tedious" ([tedious-1526]) | Not applicable | Yes. `authentication.type: 'ntlm'` with `userName`, `password`, `domain` ([tedious-conn]); payload builds LMv2 and NTLMv2 responses ([tedious-ntlmpayload]) | `domain` is required and is upper-cased by the driver ([tedious-conn]) |
| FreeTDS, and pymssql on top of it | Yes when FreeTDS is built with `--enable-krb5` ([freetds-config]). Kerberos is used when the user name is empty (TDS 7.0+) ([freetds-login]) | Uses the default GSSAPI credential (`GSS_C_NO_CREDENTIAL`) ([freetds-gss]); docs say to run `kinit` or configure PAM ([freetds-kerb]) | Yes. `DOMAIN\username` plus the domain password ([freetds-domains]); `use ntlmv2` defaults to `yes` ([freetds-conf]) | pymssql Linux wheels are built with `--enable-krb5` ([pymssql-pyproject]). pymssql docs do not document Windows authentication (not confirmed beyond build config and source) |

## Per-driver notes

### Microsoft ODBC Driver 17 and 18 (pyodbc)

- The driver on Linux and macOS supports Kerberos integrated authentication using GSSAPI and Kerberos 5 libraries, and supports the MIT Kerberos KDC ([odbc-ia]).
- Connection string from the docs: `Driver={ODBC Driver 18 for SQL Server};Server=<server>;Encrypt=yes;TrustServerCertificate=no;Trusted_Connection=yes;`. For Driver 17, change the driver name ([odbc-ia]).
- For Windows integrated authentication, omit `UID` and `PWD`; on Linux and macOS, configure Kerberos and obtain a ticket for the application's identity ([odbc-conf]). Do not combine `Trusted_Connection=yes` with an `Authentication` value ([odbc-conf]).
- Default SPN for TCP is `MSSQLSvc/<fqdn>:<port>`. Use `ServerSPN` to override; current Driver 17 and 18 releases support it on Linux and macOS. `FailoverPartnerSPN` is Windows-only ([odbc-ia], [odbc-kw]).
- Ticket sources listed by Microsoft: `kinit` with a password at the prompt, `kinit` with a keytab for the service principal, or a Kerberos PAM login. "Kerberos credentials expire, and the ODBC driver doesn't renew them." The credential cache must be available to the process ([odbc-ia]).
- Realm and KDC come from `krb5.conf` (usually `/etc`); set `KRB5_CONFIG` to use another file. Client and KDC clocks must be in sync ([odbc-ia]).
- The page does not name `KRB5CCNAME`. MIT defines `KRB5CCNAME` as the "Default name for the credentials cache file, in the form TYPE:residual" ([mit-env]).
- NTLM: "If Kerberos authentication fails, the driver on Linux and macOS doesn't fall back to NT LAN Manager (NTLM) authentication" ([odbc-ia]). No NTLM keyword appears in the keyword reference ([odbc-kw]).
- `sqlcmd -E` and `bcp -T` use the same Kerberos credential cache ([odbc-ia]).
- Changing the process credential cache does not change the login of an already authenticated pooled connection ([odbc-ia]).
- pyodbc: `connect()` "uses that string exactly as given" and passes it to the driver manager; keyword arguments are appended ([pyodbc-mod]).

### Microsoft JDBC Driver (mssql-jdbc)

- `authenticationScheme` values: `JavaKerberos`, `NTLM` (version 7.4+), and the default `NativeAuthentication`. `NativeAuthentication` loads `mssql-jdbc_auth-<version>-<arch>.dll` on Windows ([jdbc-props]).
- Kerberos: set `integratedSecurity=true` and `authenticationScheme=JavaKerberos`. If `integratedSecurity=true` is missing, the driver ignores `authenticationScheme` and expects a user name and password ([jdbc-kerb]).
- Example URL pattern: `jdbc:sqlserver://<fqdn>:1433;databaseName=<db>;encrypt=true;integratedSecurity=true;authenticationScheme=JavaKerberos` ([jdbc-kerb]). With JavaKerberos, the FQDN must be in `serverName` or `serverSpn`, or the connection fails with "Server not found in Kerberos database" ([jdbc-props]).
- JVM flags: `-Djava.security.krb5.conf=<krb5.conf>` and `-Djava.security.auth.login.config=<jaas.conf>` ([jdbc-kerb]).
- Default JAAS behavior: for non-IBM JVMs the driver sets `useTicketCache=true` and `doNotPrompt=true`; the default entry name is `SQLJDBCDriver` ([jdbc-kerb]). The driver's built-in default config contains `useTicketCache=true` ([jdbc-jaassrc]).
- Linux steps in the docs: optionally set `KRB5CCNAME`, run `kinit USER@DOMAIN`, check with `klist`, then connect ([jdbc-kerb]).
- Keytab without `kinit`: the driver calls `new LoginContext(configName, callback)` and `lc.login()` against the JVM JAAS configuration ([jdbc-kerbsrc]). `Krb5LoginModule` options `useKeyTab`, `keyTab`, `principal`, `storeKey`, `doNotPrompt` read the key from a keytab ([krb5lm]). The repository's sample sets `useKeyTab=true`, `storeKey=true`, `doNotPrompt=true`, `keyTab`, `principal`, `isInitiator=true` ([jdbc-sample]). A JAAS entry built from those documented options (Microsoft does not publish this exact example):

  ```text
  SQLJDBCDriver {
    com.sun.security.auth.module.Krb5LoginModule required
      useKeyTab=true
      keyTab="/path/to/client.keytab"
      principal="svc_account@EXAMPLE.COM"
      storeKey=true
      doNotPrompt=true;
  };
  ```

- `jaasConfigurationName` (6.2+) selects a different entry name per connection ([jdbc-props]). `useDefaultJaasConfig` (12.6+) uses a JAAS configuration already set at the system level ([jdbc-props]). If `useDefaultJaasConfig=true` and a non-default name is set, the source logs a warning and uses the default name ([jdbc-kerbsrc]).
- From 13.6, `java.security.auth.login.config` must be a filesystem path or `file:` URI; `http:`, `https:`, `ldap:`, `jar:`, `rmi:` URLs are rejected ([jdbc-kerb], [jdbc-props]).
- Principal and password (6.2+): `integratedSecurity=true;authenticationScheme=JavaKerberos;userName=user@REALM;password=<password>`. `realm` property (9.4+) sets the realm separately ([jdbc-kerb], [jdbc-props]).
- `serverSpn` (4.2+) form: `MSSQLSvc/fqdn:port@REALM`; `@REALM` optional when the client's default realm matches ([jdbc-props]). From 6.2 the driver builds the SPN by default, including cross-realm ([jdbc-kerb]).
- `gsscredential` (6.2+) passes a `GSSCredential` for constrained delegation ([jdbc-props]).
- If the JDK `Krb5LoginModule` finds no ticket cache path, it looks in `/tmp/krb5cc_<uid>` on Linux, then `{user.home}/krb5cc_{user.name}` ([krb5lm]). The Oracle page read does not mention `KRB5CCNAME`; the Microsoft page instructs setting it ([jdbc-kerb]).
- Kerberos GSS context uses mechanism OID `1.2.840.113554.1.2.2` (Kerberos v5) in `KerbAuthentication` ([jdbc-kerbsrc]).
- NTLM: added in 7.3.1 preview ("Added support for NTLM Authentication", PR #998) ([jdbc-changelog]); the properties page lists 7.4+ ([jdbc-props]). Only NTLM v2 is supported ([jdbc-ntlm]). Properties: `integratedSecurity=true;authenticationScheme=NTLM;domain=<domain>;user=<user>;password=<password>` ([jdbc-ntlm]).
- NTLM docs disagree on `domain`: the properties page says domain must be specified ([jdbc-props]); the NTLM page marks `domain` optional ([jdbc-ntlm]). The source fills a default domain when absent and throws only when user or password is empty ([jdbc-connsrc]).
- NTLM constraint: "For the 7.4 release, enabling both Extended Protection and Encryption is not supported" ([jdbc-ntlm]). For Kerberos with Extended Protection, Force Encryption must not be enabled ([jdbc-kerb]).
- Trace loggers: `com.microsoft.sqlserver.jdbc.internals.KerbAuthentication` and `com.microsoft.sqlserver.jdbc.internals.NTLMAuthentication` ([jdbc-kerb], [jdbc-ntlm]).

### Microsoft.Data.SqlClient (.NET 6 and 8)

- Connection string from the docs: `Server=tcp:<server>,1433;Database=<database>;Integrated Security=true;Encrypt=true;TrustServerCertificate=false;`. `Trusted_Connection=true` is a synonym ([sqlclient-auth]).
- Modern .NET on Linux "uses managed networking and .NET `NegotiateAuthentication`. Configure the Kerberos client, realm, service principal name (SPN), and valid credentials or ticket cache." On Linux, obtain a ticket with `kinit` before starting the application ([sqlclient-auth]).
- With integrated security on, `User ID` and `Password` "are ignored. They don't select a different Windows user" ([sqlclient-auth]).
- The built-in provider creates `NegotiateAuthentication` with `Package = "Negotiate"` and `TargetName` set to the SPN, with no explicit credential ([sqlclient-neg]).
- SqlClient switched to `NegotiateAuthentication` on .NET 7.0 and above for managed SNI in 5.2.0; 5.2.0 also fixed the SPN sent for a named instance with Kerberos on Linux and macOS ([sqlclient-520]). The mechanism used on .NET 6 is not confirmed.
- `ServerSPN` / `Server SPN` and `FailoverServerSPN` / `Failover Server SPN` override the generated SPNs (5.0.0+) ([sqlclient-500]).
- Platform targets: 5.2 targets .NET 6.0+; 6.0, 6.1 and 7.0 target .NET 8.0+ ([sqlclient-520], [sqlclient-600], [sqlclient-610], [sqlclient-700]). The .NET install page for Debian lists .NET 8 and .NET 6 as no longer supported ([dotnet-debian]).
- .NET on Debian depends on `libgssapi-krb5-2` when installed without a package manager ([dotnet-debian]).
- Keytab: no keytab connection keyword was found in the sources read. The 7.0.0 notes describe `SspiContextProvider` for "Users running in containers who need manual Kerberos negotiation without deploying sidecars or external ticket-refresh mechanisms" ([sqlclient-700]).
- NTLM: 7.0.0 added a public `SspiContextProvider` on `SqlConnection` "enabling custom Kerberos ticket negotiation and NTLM username/password authentication scenarios", for "Users who need NTLM username/password authentication to SQL Server, which the driver does not provide natively" ([sqlclient-700]). The Learn page describes it as an extensibility point for "custom Kerberos or explicit NTLM credentials", not a connection-string mode ([sqlclient-auth]). The provider is part of the pool key ([sqlclient-700]).
- NTLM on Unix through .NET "may require installing the NTLM plugin for the GSSAPI layer" (`gss-ntlmssp` on Ubuntu), or opting in to managed NTLM with the `System.Net.Security.UseManagedNtlm` AppContext switch (the `_UseManagedNtlm` project property is .NET 9+ only) ([netcred]).
- "Don't assume a Windows NTLM fallback is available on a non-Windows client" ([sqlclient-auth]).

### go-mssqldb

- Import `_ "github.com/microsoft/go-mssqldb/integratedauth/krb5"`; it activates with `authenticator=krb5` ([go-readme]). The Kerberos code uses `github.com/go-krb5/krb5` ([go-krb5src]), described as a "Pure Go Kerberos 5 library" ([go-krb5lib]).
- Three Kerberos modes from the README ([go-readme]):
  - Keytab: `authenticator=krb5;server=<host>;database=<db>;user id=<user>;krb5-realm=<REALM>;krb5-configfile=/etc/krb5.conf;krb5-keytabfile=<path>`
  - Credential cache: `authenticator=krb5;server=<host>;database=<db>;krb5-configfile=/etc/krb5.conf;krb5-credcachefile=<path>`
  - User and password: `authenticator=krb5;server=<host>;database=<db>;user id=<user>;password=<pwd>;krb5-realm=<REALM>;krb5-configfile=/etc/krb5.conf`
- Other parameters: `krb5-dnslookupkdc` (default true), `krb5-udppreferencelimit` (default 1), `ServerSPN` (default `MSSQLSvc/host:port`) ([go-readme]).
- Environment fallbacks in source: `KRB5_CONFIG`, then `/etc/krb5.conf`; with a user name and no password, `KRB5_KTNAME`, then the krb5.conf `default_client_keytab_name`, then `default_keytab_name`; cache from `KRB5CCNAME` ([go-krb5src]). Keytab takes precedence over the cache when a user name is given without a password ([go-krb5src]).
- Validation in source: keytab mode needs `user id`, a realm and a krb5 config file; user and password mode needs a realm and a config file ([go-krb5src]). Realm can come from `user@REALM` or the config's default realm ([go-krb5src]).
- NTLM: on non-Windows builds the `ntlm` provider is registered and set as default ([go-authunix]). `user id` must contain a backslash, `DOMAIN\User`; URL form uses `%5C` (`sqlserver://DOMAIN%5Cusername:password@host`) ([go-readme], [go-ntlmsrc]). NTLMv2 responses are built when the server's challenge includes target info ([go-ntlmsrc]).
- When `authenticator` is absent and the default provider rejects the config (for example no backslash in `user id`), the error is ignored "to force a fallback to sqlserver authentication" ([go-authsrc]).

### Node.js tedious

- Authentication types accepted: `default`, `ntlm`, `token-credential`, `azure-active-directory-password`, `azure-active-directory-access-token`, `azure-active-directory-default`, `azure-active-directory-msi-vm`, `azure-active-directory-msi-app-service`, `azure-active-directory-service-principal-secret` ([tedious-conn]). None is Kerberos.
- NTLM config shape from the type definitions ([tedious-conn]):

  ```js
  { server: '<host>',
    authentication: { type: 'ntlm', options: { userName: '<user>', password: '<password>', domain: '<DOMAIN>' } },
    options: { encrypt: true } }
  ```

- `options.domain` must be a string, and the driver stores it upper-cased ([tedious-conn]).
- The NTLM response payload contains LMv2 and NTLMv2 responses ([tedious-ntlmpayload]).
- Kerberos: issue #1526 "Kerberos support" is open; a contributor wrote "we do not have a plan for supporting Kerberos within tedious" and mentioned possible pluggable authentication ([tedious-1526]).

### FreeTDS and pymssql

- FreeTDS on non-Windows hosts provides single sign-on through Kerberos; without SSPI or Kerberos, use `DOMAIN\username` with the domain password. When FreeTDS sees `\` it chooses a domain login ([freetds-domains]).
- Domain logins need TDS 7.0 or later and work over TCP/IP only ([freetds-domains]).
- `--enable-krb5`: "With Kerberos you can connect to server using your stored Kerberos ticket" ([freetds-config]). The Kerberos page says to set up a keytab from Active Directory, configure `/etc/krb5.conf`, and use `kinit` or PAM ([freetds-kerb]).
- Source: for TDS 7+, a backslash in the user name selects NTLM; an empty user name selects GSSAPI if compiled with `ENABLE_KRB5`, else the login fails with "requested GSS authentication but not compiled in" ([freetds-login]). GSSAPI is called with `GSS_C_NO_CREDENTIAL` and the default mechanism ([freetds-gss]).
- `freetds.conf` settings: `enable gssapi delegation` (default off), `realm`, `SPN` (default `MSSQLSvc/fqdn:port`), `mutual authentication` (always on for TDS 7.0+), `use ntlmv2` (default yes), `use lanman` (default no) ([freetds-conf]).
- NTLM example from the docs: `tsql -S <server> -U 'DOMAIN\user' -P <password>` ([freetds-domains]).
- pymssql: `connect()` calls `DBSETLUSER` only when `user` is not `None` ([pymssql-mssql]). With `user` omitted, FreeTDS sees an empty user name and takes the GSSAPI path if built with Kerberos ([freetds-login]). pymssql docs do not describe this; the behavior here is derived from source.
- pymssql Linux wheels (manylinux and musllinux) are built with `--enable-krb5 --static-freetds` ([pymssql-pyproject]). macOS and Windows wheel builds do not pass `--enable-krb5` ([pymssql-pyproject]). Whether the Kerberos runtime libraries are present in a given installed wheel is not confirmed.
- The pymssql FAQ shows `tsql -C` output with a "Kerberos: yes/no" line for checking a FreeTDS build ([pymssql-faq]).

## Shared MIT Kerberos client settings

These apply to drivers that use the system GSSAPI library (ODBC, SqlClient on Linux, FreeTDS). go-mssqldb and the JDK use their own Kerberos implementations ([go-krb5lib], [krb5lm]).

- `KRB5CCNAME`: default credential cache name, `TYPE:residual` ([mit-env]).
- `KRB5_CONFIG`: location of the Kerberos configuration file ([mit-env]).
- `KRB5_KTNAME`: default keytab file; `KRB5_CLIENT_KTNAME`: default client keytab file ([mit-env]).
- `KRB5_TRACE`: file to write trace log output to ([mit-env]).
- MIT GSSAPI can acquire initial tickets from the default client keytab when no tickets exist, and refreshes them as they approach expiry ([mit-gssapi], [mit-keytab]). Microsoft docs for the ODBC driver and SqlClient do not describe this path; whether those drivers work with only `KRB5_CLIENT_KTNAME` and no `kinit` is not confirmed.

## Kerberos to NTLM fallback and how to check

- On the Windows client path described in the SPN article: "If a security token can't be obtained, authentication uses NTLM", and "In an Active Directory environment, Kerberos authentication is always attempted first" ([spn]). If the SPN lookup fails or maps to the wrong kind of account, local and remote connections use NTLM; if the SPN maps to an incorrect account, authentication fails ([spn]).
- The fallback depends on the client. Linux clients:
  - ODBC: no fallback to NTLM ([odbc-ia]).
  - SqlClient: "Don't assume a Windows NTLM fallback is available on a non-Windows client" ([sqlclient-auth]). Whether Negotiate on Linux selects NTLM when `gss-ntlmssp` is installed is not confirmed.
  - JDBC JavaKerberos: the GSS context uses the Kerberos v5 OID; NTLM runs only with `authenticationScheme=NTLM` ([jdbc-kerbsrc], [jdbc-connsrc]).
  - go-mssqldb: the provider is picked by `authenticator`; with the default `ntlm` provider and no `DOMAIN\` prefix the driver uses SQL authentication ([go-authsrc]).
  - FreeTDS and pymssql: the user name form picks the method, `DOMAIN\user` for NTLM and empty for GSSAPI ([freetds-login]).
  - tedious: NTLM only ([tedious-conn]).
- Check the scheme on the server: `SELECT auth_scheme FROM sys.dm_exec_connections WHERE session_id = @@SPID;` ([spn], [authtype]). `auth_scheme` is `nvarchar(40)`, "SQL Server/Windows Authentication scheme used with this connection" ([dmv]). The SPN article states the value is `KERBEROS` when Kerberos is used ([spn]); the troubleshooting article shows `NTLM` output ([authtype]). The value returned for SQL logins is not confirmed in the sources read.
- Run the check from a client computer, not on the SQL Server host; local connections always report `NTLM` because of the per-service SID hardening added in Windows 2008 ([authtype]).
- Permission: `VIEW SERVER STATE` on SQL Server and SQL Managed Instance; `VIEW SERVER PERFORMANCE STATE` on SQL Server 2022 and later ([dmv]).
- "A successful connection alone doesn't prove that Kerberos, rather than NTLM, was used" ([sqlclient-auth]).
- SPN formats: `MSSQLSvc/<FQDN>:<port>` (TCP default), `MSSQLSvc/<FQDN>` (default instance, non-TCP), `MSSQLSvc/<FQDN>:<instancename>` (named instance, non-TCP); register with `setspn -S` ([spn]).

## Not confirmed

- An NTLM mode for the Microsoft ODBC driver on Linux. No keyword is documented ([odbc-kw]) and the driver does not fall back to NTLM ([odbc-ia]).
- Keytab-only (no `kinit`) operation of the ODBC driver, SqlClient, or FreeTDS through MIT `KRB5_CLIENT_KTNAME`.
- Whether SqlClient's Negotiate path on Linux can negotiate NTLM when `gss-ntlmssp` is installed.
- The negotiation mechanism SqlClient uses on .NET 6.
- Kerberos runtime availability inside a specific published pymssql wheel.
- The `auth_scheme` string returned for SQL authentication logins.

## Beginner reading list

Each URL was opened on 2026-11-23; the title is the page title returned.

1. "Use Integrated Authentication - ODBC Driver for SQL Server": https://learn.microsoft.com/en-us/sql/connect/odbc/linux-mac/using-integrated-authentication
2. "Using Kerberos integrated authentication to connect to SQL Server - JDBC Driver for SQL Server": https://learn.microsoft.com/en-us/sql/connect/jdbc/using-kerberos-integrated-authentication-to-connect-to-sql-server
3. "Using NTLM authentication to connect to SQL Server - JDBC Driver for SQL Server": https://learn.microsoft.com/en-us/sql/connect/jdbc/using-ntlm-authentication-to-connect-to-sql-server
4. "SQL Server and Windows Authentication with Microsoft.Data.SqlClient - ADO.NET Provider for SQL Server": https://learn.microsoft.com/en-us/sql/connect/ado-net/sql/authentication-sql-server
5. "Register a Service Principal Name for Kerberos Connections - SQL Server": https://learn.microsoft.com/en-us/sql/database-engine/configure-windows/register-a-service-principal-name-for-kerberos-connections
6. "Determine the authentication type - SQL Server": https://learn.microsoft.com/en-us/troubleshoot/sql/database-engine/connect/determine-the-authentication-type
7. "go-mssqldb/README.md at main · microsoft/go-mssqldb", section "Kerberos Active Directory authentication outside Windows": https://github.com/microsoft/go-mssqldb/blob/main/README.md
8. "Kerberos Support" (FreeTDS User Guide), with the preceding chapter "Domain Logins": https://www.freetds.org/userguide/kerberos.html and https://www.freetds.org/userguide/domains.html

## Sources

- odbc-ia: https://learn.microsoft.com/en-us/sql/connect/odbc/linux-mac/using-integrated-authentication
- odbc-conf: https://learn.microsoft.com/en-us/sql/connect/odbc/configure-connections
- odbc-kw: https://learn.microsoft.com/en-us/sql/connect/odbc/dsn-connection-string-attribute
- pyodbc-mod: https://github.com/mkleehammer/pyodbc/wiki/The-pyodbc-Module
- jdbc-kerb: https://learn.microsoft.com/en-us/sql/connect/jdbc/using-kerberos-integrated-authentication-to-connect-to-sql-server
- jdbc-props: https://learn.microsoft.com/en-us/sql/connect/jdbc/setting-the-connection-properties
- jdbc-ntlm: https://learn.microsoft.com/en-us/sql/connect/jdbc/using-ntlm-authentication-to-connect-to-sql-server
- jdbc-kerbsrc: https://github.com/microsoft/mssql-jdbc/blob/main/src/main/java/com/microsoft/sqlserver/jdbc/KerbAuthentication.java
- jdbc-jaassrc: https://github.com/microsoft/mssql-jdbc/blob/main/src/main/java/com/microsoft/sqlserver/jdbc/JaasConfiguration.java
- jdbc-connsrc: https://github.com/microsoft/mssql-jdbc/blob/main/src/main/java/com/microsoft/sqlserver/jdbc/SQLServerConnection.java
- jdbc-sample: https://github.com/microsoft/mssql-jdbc/blob/main/src/samples/constrained/src/main/java/ConstrainedDelegation.java
- jdbc-changelog: https://github.com/microsoft/mssql-jdbc/blob/main/CHANGELOG.md
- jdbc-releases: https://github.com/microsoft/mssql-jdbc/releases
- krb5lm: https://docs.oracle.com/en/java/javase/21/docs/api/jdk.security.auth/com/sun/security/auth/module/Krb5LoginModule.html
- sqlclient-auth: https://learn.microsoft.com/en-us/sql/connect/ado-net/sql/authentication-sql-server
- sqlclient-neg: https://github.com/dotnet/SqlClient/blob/main/src/Microsoft.Data.SqlClient/src/Microsoft/Data/SqlClient/SSPI/NegotiateSspiContextProvider.cs
- sqlclient-500: https://github.com/dotnet/SqlClient/blob/main/release-notes/5.0/5.0.0.md
- sqlclient-520: https://github.com/dotnet/SqlClient/blob/main/release-notes/5.2/5.2.0.md
- sqlclient-600: https://github.com/dotnet/SqlClient/blob/main/release-notes/6.0/6.0.0.md
- sqlclient-610: https://github.com/dotnet/SqlClient/blob/main/release-notes/6.1/6.1.0.md
- sqlclient-700: https://github.com/dotnet/SqlClient/blob/main/release-notes/7.0/7.0.0.md
- sqlclient-releases: https://github.com/dotnet/SqlClient/releases
- netcred: https://learn.microsoft.com/en-us/dotnet/api/system.net.networkcredential
- dotnet-debian: https://learn.microsoft.com/en-us/dotnet/core/install/linux-debian
- go-readme: https://github.com/microsoft/go-mssqldb/blob/main/README.md
- go-krb5src: https://github.com/microsoft/go-mssqldb/blob/main/integratedauth/krb5/krb5.go
- go-ntlmsrc: https://github.com/microsoft/go-mssqldb/blob/main/integratedauth/ntlm/ntlm.go
- go-authsrc: https://github.com/microsoft/go-mssqldb/blob/main/integratedauth/auth.go
- go-authunix: https://github.com/microsoft/go-mssqldb/blob/main/auth_unix.go
- go-krb5lib: https://github.com/go-krb5/krb5
- go-releases: https://github.com/microsoft/go-mssqldb/releases
- tedious-conn: https://github.com/tediousjs/tedious/blob/master/src/connection.ts
- tedious-ntlmpayload: https://github.com/tediousjs/tedious/blob/master/src/ntlm-payload.ts
- tedious-1526: https://github.com/tediousjs/tedious/issues/1526
- tedious-releases: https://github.com/tediousjs/tedious/releases
- freetds-domains: https://www.freetds.org/userguide/domains.html
- freetds-kerb: https://www.freetds.org/userguide/kerberos.html
- freetds-conf: https://www.freetds.org/userguide/freetdsconf.html
- freetds-config: https://www.freetds.org/userguide/config.html
- freetds-login: https://github.com/FreeTDS/freetds/blob/master/src/tds/login.c
- freetds-gss: https://github.com/FreeTDS/freetds/blob/master/src/tds/gssapi.c
- freetds-releases: https://github.com/FreeTDS/freetds/releases
- pymssql-pyproject: https://github.com/pymssql/pymssql/blob/master/pyproject.toml
- pymssql-mssql: https://github.com/pymssql/pymssql/blob/master/src/pymssql/_mssql.pyx
- pymssql-faq: https://github.com/pymssql/pymssql/blob/master/docs/faq.rst
- spn: https://learn.microsoft.com/en-us/sql/database-engine/configure-windows/register-a-service-principal-name-for-kerberos-connections
- dmv: https://learn.microsoft.com/en-us/sql/relational-databases/system-dynamic-management-objects/sys-dm-exec-connections-transact-sql
- authtype: https://learn.microsoft.com/en-us/troubleshoot/sql/database-engine/connect/determine-the-authentication-type
- mit-env: https://web.mit.edu/kerberos/krb5-latest/doc/user/user_config/kerberos.html
- mit-keytab: https://web.mit.edu/kerberos/krb5-latest/doc/basic/keytab_def.html
- mit-gssapi: https://web.mit.edu/kerberos/krb5-latest/doc/appdev/gssapi.html

[odbc-ia]: https://learn.microsoft.com/en-us/sql/connect/odbc/linux-mac/using-integrated-authentication
[odbc-conf]: https://learn.microsoft.com/en-us/sql/connect/odbc/configure-connections
[odbc-kw]: https://learn.microsoft.com/en-us/sql/connect/odbc/dsn-connection-string-attribute
[pyodbc-mod]: https://github.com/mkleehammer/pyodbc/wiki/The-pyodbc-Module
[jdbc-kerb]: https://learn.microsoft.com/en-us/sql/connect/jdbc/using-kerberos-integrated-authentication-to-connect-to-sql-server
[jdbc-props]: https://learn.microsoft.com/en-us/sql/connect/jdbc/setting-the-connection-properties
[jdbc-ntlm]: https://learn.microsoft.com/en-us/sql/connect/jdbc/using-ntlm-authentication-to-connect-to-sql-server
[jdbc-kerbsrc]: https://github.com/microsoft/mssql-jdbc/blob/main/src/main/java/com/microsoft/sqlserver/jdbc/KerbAuthentication.java
[jdbc-jaassrc]: https://github.com/microsoft/mssql-jdbc/blob/main/src/main/java/com/microsoft/sqlserver/jdbc/JaasConfiguration.java
[jdbc-connsrc]: https://github.com/microsoft/mssql-jdbc/blob/main/src/main/java/com/microsoft/sqlserver/jdbc/SQLServerConnection.java
[jdbc-sample]: https://github.com/microsoft/mssql-jdbc/blob/main/src/samples/constrained/src/main/java/ConstrainedDelegation.java
[jdbc-changelog]: https://github.com/microsoft/mssql-jdbc/blob/main/CHANGELOG.md
[jdbc-releases]: https://github.com/microsoft/mssql-jdbc/releases
[krb5lm]: https://docs.oracle.com/en/java/javase/21/docs/api/jdk.security.auth/com/sun/security/auth/module/Krb5LoginModule.html
[sqlclient-auth]: https://learn.microsoft.com/en-us/sql/connect/ado-net/sql/authentication-sql-server
[sqlclient-neg]: https://github.com/dotnet/SqlClient/blob/main/src/Microsoft.Data.SqlClient/src/Microsoft/Data/SqlClient/SSPI/NegotiateSspiContextProvider.cs
[sqlclient-500]: https://github.com/dotnet/SqlClient/blob/main/release-notes/5.0/5.0.0.md
[sqlclient-520]: https://github.com/dotnet/SqlClient/blob/main/release-notes/5.2/5.2.0.md
[sqlclient-600]: https://github.com/dotnet/SqlClient/blob/main/release-notes/6.0/6.0.0.md
[sqlclient-610]: https://github.com/dotnet/SqlClient/blob/main/release-notes/6.1/6.1.0.md
[sqlclient-700]: https://github.com/dotnet/SqlClient/blob/main/release-notes/7.0/7.0.0.md
[sqlclient-releases]: https://github.com/dotnet/SqlClient/releases
[netcred]: https://learn.microsoft.com/en-us/dotnet/api/system.net.networkcredential
[dotnet-debian]: https://learn.microsoft.com/en-us/dotnet/core/install/linux-debian
[go-readme]: https://github.com/microsoft/go-mssqldb/blob/main/README.md
[go-krb5src]: https://github.com/microsoft/go-mssqldb/blob/main/integratedauth/krb5/krb5.go
[go-ntlmsrc]: https://github.com/microsoft/go-mssqldb/blob/main/integratedauth/ntlm/ntlm.go
[go-authsrc]: https://github.com/microsoft/go-mssqldb/blob/main/integratedauth/auth.go
[go-authunix]: https://github.com/microsoft/go-mssqldb/blob/main/auth_unix.go
[go-krb5lib]: https://github.com/go-krb5/krb5
[go-releases]: https://github.com/microsoft/go-mssqldb/releases
[tedious-conn]: https://github.com/tediousjs/tedious/blob/master/src/connection.ts
[tedious-ntlmpayload]: https://github.com/tediousjs/tedious/blob/master/src/ntlm-payload.ts
[tedious-1526]: https://github.com/tediousjs/tedious/issues/1526
[tedious-releases]: https://github.com/tediousjs/tedious/releases
[freetds-domains]: https://www.freetds.org/userguide/domains.html
[freetds-kerb]: https://www.freetds.org/userguide/kerberos.html
[freetds-conf]: https://www.freetds.org/userguide/freetdsconf.html
[freetds-config]: https://www.freetds.org/userguide/config.html
[freetds-login]: https://github.com/FreeTDS/freetds/blob/master/src/tds/login.c
[freetds-gss]: https://github.com/FreeTDS/freetds/blob/master/src/tds/gssapi.c
[freetds-releases]: https://github.com/FreeTDS/freetds/releases
[pymssql-pyproject]: https://github.com/pymssql/pymssql/blob/master/pyproject.toml
[pymssql-mssql]: https://github.com/pymssql/pymssql/blob/master/src/pymssql/_mssql.pyx
[pymssql-faq]: https://github.com/pymssql/pymssql/blob/master/docs/faq.rst
[spn]: https://learn.microsoft.com/en-us/sql/database-engine/configure-windows/register-a-service-principal-name-for-kerberos-connections
[dmv]: https://learn.microsoft.com/en-us/sql/relational-databases/system-dynamic-management-objects/sys-dm-exec-connections-transact-sql
[authtype]: https://learn.microsoft.com/en-us/troubleshoot/sql/database-engine/connect/determine-the-authentication-type
[mit-env]: https://web.mit.edu/kerberos/krb5-latest/doc/user/user_config/kerberos.html
[mit-keytab]: https://web.mit.edu/kerberos/krb5-latest/doc/basic/keytab_def.html
[mit-gssapi]: https://web.mit.edu/kerberos/krb5-latest/doc/appdev/gssapi.html
