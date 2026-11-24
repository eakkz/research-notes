# PoC test plan

The goal is to prove feasibility one layer at a time, so that when something fails you know which layer broke. Run the checks in order from a pod in the PoC project on ROSA HCP. Each check names what it proves and what to look at when it fails. Commands are the standard MIT Kerberos and Linux tools.

Start once everything in [03-prerequisites-checklist.md](03-prerequisites-checklist.md) is in place.

## Stage 1: one container, kinit by hand (option B)

Use a single debug pod built from the app image plus `krb5-workstation`, with the keytab and `krb5.conf` mounted.

| # | Check | How | Passes when | Proves |
| --- | --- | --- | --- | --- |
| 1 | SQL Server name resolves | `getent hosts sql01.corp.example.com` | Returns the SQL Server IP | Cluster DNS reaches AD DNS |
| 2 | KDCs can be found | `dig +short SRV _kerberos._tcp.corp.example.com` or `nslookup -type=SRV ...` (both need `bind-utils`, which UBI images lack; skip if `krb5.conf` lists KDCs) | Lists the domain controllers | SRV records resolve |
| 3 | Ports are open | `timeout 3 bash -c '</dev/tcp/dc01.corp.example.com/88'` and the same for `sql01...:1433` | Exits 0 | Firewalls and routes |
| 4 | Clocks agree | Compare `date -u` in the pod with the domain controller's time | Less than 5 minutes apart | No clock skew |
| 5 | Keytab is readable and AES | `klist -kte /path/svc-app.keytab` | Shows `svc-app@CORP.EXAMPLE.COM` with `aes256-cts-hmac-sha1-96` | Right principal, right encryption |
| 6 | Ticket from keytab | `kinit -kt /path/svc-app.keytab svc-app@CORP.EXAMPLE.COM` then `klist` | `klist` shows `krbtgt/CORP.EXAMPLE.COM@CORP.EXAMPLE.COM` | Keytab matches AD, port 88 works |
| 7 | SPN exists | `kvno MSSQLSvc/sql01.corp.example.com:1433@CORP.EXAMPLE.COM` | Prints a kvno | SPN registered once, on a real account |
| 8 | SQL login with Kerberos | pyodbc with `Trusted_Connection=yes`, then run the query below | `auth_scheme` is `KERBEROS` and the login is `CORP\svc-app` | End-to-end Windows authentication |

The query for check 8:

```sql
SELECT SUSER_SNAME() AS login_name, auth_scheme, encrypt_option
FROM sys.dm_exec_connections
WHERE session_id = @@SPID;
```

`SUSER_SNAME()` shows which AD account SQL Server mapped the ticket to. `auth_scheme` must say `KERBEROS`; a successful login alone does not prove which method was used ([Microsoft: Windows authentication with SqlClient](https://learn.microsoft.com/en-us/sql/connect/ado-net/sql/authentication-sql-server)). If the account cannot read the view, the DBA can run the same filter on the session from their side.

Stage 1 is the feasibility proof. Everything after it is about running the result safely.

## Stage 2: sidecar (option A)

| # | Check | How | Passes when |
| --- | --- | --- | --- |
| 9 | Keytab is only in the sidecar | Inspect the app container's mounts | No keytab path in the app container |
| 10 | Shared cache works | App container: `klist` with `KRB5CCNAME` set | Shows the ticket the sidecar wrote |
| 11 | Login from the app container | Repeat check 8 | `KERBEROS`, `CORP\svc-app` |
| 12 | Cache mount read-only or read-write | Try the app with the cache mounted read-only (the Airflow chart does this) | Decide which mount mode to keep |
| 13 | App waits for the ticket | Delete the pod and watch startup | The app never logs a "No Kerberos credentials available" error at start (exact wording varies by driver version) |
| 14 | Renewal | Run `kdestroy` in the sidecar, wait one refresh period, open a new connection | New connection succeeds |
| 15 | Sidecar crash | Kill the sidecar process | OpenShift restarts it and new connections keep working |
| 16 | UID check | `id` in both containers | Same UID |

## Stage 3: operations drills

| # | Drill | Passes when |
| --- | --- | --- |
| 17 | Keytab rotation in the documented order (new kvno added, Secret updated, then password changed) | No failed `kinit` in the sidecar log |
| 18 | Rotation in the wrong order (password first), on purpose, in the PoC only | You see the failure message and how long recovery takes |
| 19 | KDC unreachable for longer than the refresh period | Open connections survive; new ones fail once the ticket expires; recovery is automatic |
| 20 | `KRB5_CLIENT_KTNAME` only, no `kinit` (experiment) | Tells you whether MIT's client keytab path works with ODBC 18, which Microsoft does not document |

## When a check fails

| Message | Usual cause | Where to look |
| --- | --- | --- |
| `Cannot find KDC for realm "CORP.EXAMPLE.COM"` | DNS SRV lookup fails, or realm name typo | Check 2; DNS forwarding; `krb5.conf` |
| `Cannot contact any KDC for realm` | Port 88 blocked or no route | Check 3; firewall and security groups |
| `Clock skew too great` | Pod or domain controller clock is off | Check 4 |
| `Client 'svc-app@CORP.EXAMPLE.COM' not found in Kerberos database` | Wrong principal name or realm case | Principal is case sensitive; realm in upper case |
| `Preauthentication failed` or `Key table entry not found` | Keytab is out of date (password changed, kvno moved) | Ask the AD team for the current kvno; rebuild the keytab |
| `KDC has no support for encryption type` | Keytab and account disagree on AES or RC4 | Account AES flags; keytab encryption types (check 5) |
| `Server not found in Kerberos database` | SPN missing, or the client asked for a different name (IP address, DNS alias, reverse DNS) | Check 7; use the FQDN; `rdns = false`; `ServerSPN` |
| `SSPI Provider: No Kerberos credentials available (default cache: ...)` | No ticket in the cache the driver looked at | `KRB5CCNAME` value; the sidecar ran; same UID |
| `Login failed for user 'CORP\svc-app'` | Kerberos worked; SQL Server has no login or database user for the account | DBA: `CREATE LOGIN ... FROM WINDOWS` and database user |
| `SSL Provider: ... certificate verify failed` | ODBC 18 rejects the SQL Server certificate | CA trust in the app image, certificate name |

To see what the Kerberos library does, set `KRB5_TRACE=/dev/stderr` in the container before running `kinit` or the app ([MIT: environment variables](https://web.mit.edu/kerberos/krb5-latest/doc/user/user_config/kerberos.html)). The trace shows the realm, the KDCs contacted, the SPN requested and the encryption types offered.

## Exit criteria

The PoC proves feasibility when checks 1 to 8 pass, and checks 9 to 16 pass with the sidecar. Record for the decision:

- Which AD variant was used, and the turnaround time for the account, keytab and SPN requests.
- Which DNS path and firewall source (subnets or EgressIP) were used.
- Whether the cache can be mounted read-only into the app container.
- The rotation procedure agreed with the AD and security teams.
