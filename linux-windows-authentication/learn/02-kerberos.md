# Kerberos

Kerberos is the protocol that lets the container prove to SQL Server that it is `CORP\svc-app` without ever sending a password over the network. In this project Active Directory plays the Kerberos server.

## The problem it solves

A company has many servers: databases, file shares, websites. If each of them checked passwords itself, your password would cross the network many times, and every server would need a way to verify it. With Kerberos you prove who you are once, to one trusted service, and from then on you show tickets instead of your password. The servers trust the tickets because of how they are sealed, explained below.

Three parties take part:

| Party | Role | In this design |
| --- | --- | --- |
| Client | Wants to use a service | The sidecar and the app, both acting as `svc-app` |
| Service | What the client wants to use | SQL Server on EC2 |
| KDC (Key Distribution Center) | The trusted middle party that issues tickets | Runs on every AD domain controller |

Active Directory is a Kerberos system. Every time someone on a Windows PC opens a file share or connects to SQL Server with their Windows login, Kerberos runs in the background. In a Linux container nothing does this for you, so this design runs the same steps with the MIT Kerberos tools. The name comes from Kerberos, the three-headed dog of Greek myth.

## A picture to hold on to

Think of a funfair with a main gate and many rides.

- At the main gate you show ID once and get a wristband for the day. That wristband is the ticket-granting ticket (TGT).
- At a ride's ticket booth you show the wristband and get a ticket for that one ride. That is the service ticket.
- The ride operator only looks at the ride ticket. The operator does not need to phone the main gate; the ticket itself is enough, because only the booth could have made it.
- Wristbands and ride tickets expire. After that you go back to the gate.

In Kerberos terms the main gate and the booth are both the Key Distribution Center (KDC), which runs on every AD domain controller. The ride is SQL Server.

## The words

| Term | Meaning here |
| --- | --- |
| Realm | The Kerberos name of the AD domain, written in upper case: `CORP.EXAMPLE.COM` |
| Principal | A named identity in the realm. The app is `svc-app@CORP.EXAMPLE.COM`. SQL Server is identified by its SPN, `MSSQLSvc/sql01.corp.example.com:1433` |
| KDC | Key Distribution Center. Issues TGTs and service tickets. On AD, every domain controller is a KDC, listening on port 88 |
| TGT | Ticket-granting ticket. Proves you logged in; used to ask for service tickets |
| Service ticket | A ticket for one specific service, encrypted with that service's own key |
| Long-term key | A key derived from an account's password. The KDC and the account both know it. A keytab stores it on disk |
| Credential cache (ccache) | Where a Linux client keeps its tickets, usually a file. See [04-linux-kerberos-client.md](04-linux-kerberos-client.md) |
| SPN | Service principal name, the name a client asks the KDC for when it wants a service ticket. See [03-spn-keytab-and-service-accounts.md](03-spn-keytab-and-service-accounts.md) |
| `krbtgt` | The KDC's own AD account. Its key seals every TGT, so only domain controllers can read a TGT |
| Session key | A random, short-lived key the KDC creates for each ticket. The ticket holds one copy and the client gets another |
| Authenticator | A short note with the client's name and the current time, encrypted with the session key. Sent with every ticket to prove the sender is the ticket's owner |
| Pre-authentication | The timestamp, encrypted with the account's key, that a client sends when asking for a TGT. Proves it knows the key |
| PAC | Privilege Attribute Certificate. The part of an AD ticket that lists the account's SID and the SIDs of its groups, signed by the KDC |
| SID | Security identifier. The fixed ID that Windows and AD give every account and group, such as `S-1-5-21-...-1105`. SQL Server matches logins by SID, not by name |

Every other abbreviation is spelled out in [00-abbreviations.md](00-abbreviations.md).

## The exchange, step by step

1. AS (Authentication Service) exchange, getting the TGT. The client (here, `kinit` in the sidecar) asks the KDC for a TGT for `svc-app@CORP.EXAMPLE.COM`. AD requires pre-authentication by default, so the request includes a timestamp encrypted with the account's long-term key. The KDC checks it, which proves the client knows the key, and returns a TGT.
2. TGS (Ticket-Granting Service) exchange, getting the service ticket. When pyodbc connects, the Kerberos library shows the TGT to the KDC and asks for a ticket for `MSSQLSvc/sql01.corp.example.com:1433`. The KDC finds the AD account that owns that SPN and encrypts the service ticket with that account's key.
3. AP (application) exchange, using the ticket. The ODBC driver sends the service ticket to SQL Server inside the TDS (Tabular Data Stream) login. SQL Server decrypts it with its own key. If that works, the ticket can only have come from the KDC, so SQL Server accepts that the caller is `CORP\svc-app` and looks up the matching login.

The password never crosses the network. The client contacts the KDC only in steps 1 and 2. SQL Server does not need the KDC to accept the ticket, although it may contact a domain controller to validate the ticket's group data (the PAC) or look up group membership ([Microsoft: Kerberos authentication overview](https://learn.microsoft.com/en-us/windows-server/security/kerberos/kerberos-authentication-overview)).

[02-architecture.md, section 4](../02-architecture.md#4-what-happens-during-a-login) shows the same flow as a sequence diagram.

## Keys: who knows what

In Kerberos, every account shares a secret key with the KDC and with no one else. For a user or service account, the key is derived from its password. The KDC can use every key because it holds the AD database.

| Key | Known by |
| --- | --- |
| `svc-app`'s key | The KDC, and the keytab in the sidecar |
| SQL Server's key, which belongs to the account that runs SQL Server and holds its SPN | The KDC, and SQL Server |
| The `krbtgt` key | Only the KDCs (every domain controller has it) |

`svc-app` and SQL Server never share a key with each other. The KDC is the go-between.

## Tickets are sealed envelopes

A ticket is a message from the KDC, encrypted with the key of whoever has to read it. Think of it as a sealed envelope. The KDC writes "this is svc-app, member of these groups, valid until 18:00" and seals it with SQL Server's key. The client carries the envelope but cannot open it or change it. When SQL Server opens it with its own key, it knows only the KDC could have sealed it, so it trusts what is inside.

Someone who copies a ticket off the network still cannot use it, because of session keys. Each ticket contains a fresh random session key, and the KDC sends the client its own copy of that key, sealed separately for the client. Whenever the client shows a ticket, it adds an authenticator: its name and the current time, encrypted with the session key. Only the real client has the session key, so only it can write a valid authenticator, and the timestamp stops anyone replaying an old one.

## The login in detail, steps 1 to 5

This is the same login as above, slowed down to show what is inside each message. [02-architecture.md, section 4](../02-architecture.md#4-what-happens-during-a-login) and figure 3 of [architecture.html](../architecture.html) draw it with nine arrows, because they show the cache writes and reads separately. Step 1 here is arrow 1 there, step 2 is arrows 2 and 3, step 3 is arrows 4 and 5, step 4 is arrow 6, and step 5 is arrows 7 to 9.

### Step 1: the sidecar asks for a TGT (AS-REQ)

1. The sidecar runs `kinit -kt svc-app.keytab svc-app@CORP.EXAMPLE.COM`.
2. `kinit` reads `svc-app`'s key from the keytab.
3. It sends a domain controller an AS-REQ (Authentication Service request) on port 88: "I am svc-app, please give me a TGT". Attached is the pre-authentication: the current time, encrypted with `svc-app`'s key.
4. The domain controller looks up `svc-app` in AD, takes its own copy of the key, and decrypts the timestamp. If that works and the time is within 5 minutes of its clock, the sidecar has proved it holds the key without the password ever travelling.

### Step 2: the domain controller returns the TGT (AS-REP)

The KDC creates a random key, session key 1, and sends back an AS-REP (Authentication Service reply) in two sealed parts:

```text
Part A: the TGT, sealed with the krbtgt key. Only a KDC can open it.
  +-------------------------------------------------------------+
  | client: svc-app    PAC: SID and group SIDs    valid: 10 h   |
  | session key 1                                               |
  +-------------------------------------------------------------+
Part B: sealed with svc-app's key. The sidecar can open it.
  +-------------------------------------------------------------+
  | session key 1      valid: 10 h                              |
  +-------------------------------------------------------------+
```

The sidecar opens part B with the key from the keytab and gets session key 1. It cannot open the TGT, and does not need to. It writes the TGT and session key 1 into the shared ticket cache. With those two, anyone can ask for service tickets as `svc-app` until the TGT expires, which is why the cache lives in memory and only its owner can read it.

### Step 3: the app asks for a service ticket (TGS-REQ)

1. Python calls `pyodbc.connect(...)` with `Trusted_Connection=yes`.
2. The ODBC driver asks the Kerberos library, through GSSAPI, for a ticket to `MSSQLSvc/sql01.corp.example.com:1433`. It builds that SPN from the server name in the connection string.
3. The library reads the TGT and session key 1 from the cache file named by `KRB5CCNAME`.
4. It sends a domain controller a TGS-REQ (ticket-granting service request) on port 88, holding three things: the TGT, still sealed; an authenticator ("svc-app, 10:00:05") encrypted with session key 1; and the SPN it wants a ticket for.
5. The KDC opens the TGT with the `krbtgt` key, finds session key 1 inside, and uses it to open the authenticator. It checks that the names match and the time is fresh. That proves the request comes from whoever received the TGT in step 2, without the keytab being used again.
6. The KDC looks up which AD account holds that SPN (SQL Server's service account) and takes that account's key.

This step is why the app goes back to the domain controller even though the TGT is already in the cache. The TGT is proof of identity for the KDC only. SQL Server cannot read it, because it is sealed with the `krbtgt` key. The app needs a second ticket sealed with SQL Server's key, and only the KDC can make one.

### Step 4: the domain controller returns the service ticket (TGS-REP)

The KDC creates another random key, session key 2, and sends back a TGS-REP (ticket-granting service reply):

```text
Part C: the service ticket, sealed with SQL Server's key. Only SQL Server can open it.
  +-------------------------------------------------------------+
  | client: svc-app    PAC: SID and group SIDs  valid: <= TGT   |
  | session key 2                                               |
  +-------------------------------------------------------------+
Part D: sealed with session key 1. The app can open it.
  +-------------------------------------------------------------+
  | session key 2                                               |
  +-------------------------------------------------------------+
```

The library opens part D with session key 1 and gets session key 2. It normally stores the service ticket and session key 2 in the cache too, so later connections reuse them without asking the KDC again. A service ticket never outlives the TGT it came from.

### Step 5: the app logs in to SQL Server (AP-REQ)

1. The ODBC driver connects to SQL Server on port 1433. Inside the TDS login it sends an AP-REQ (application request): the service ticket from part C, and a new authenticator ("svc-app, 10:00:06") encrypted with session key 2.
2. SQL Server opens the service ticket with its own key. Only the KDC could have sealed it, so the contents can be trusted. Inside are the client name, the PAC with `svc-app`'s SID, and session key 2.
3. It opens the authenticator with session key 2 and checks the name and time. That proves the sender is the ticket's owner and not someone replaying a copy.
4. It takes `svc-app`'s SID from the PAC and finds the SQL Server login with the same SID, `[CORP\svc-app]`, which the DBA created. A login for one of the account's AD groups works too. That login's permissions apply.
5. The login succeeds, and `sys.dm_exec_connections` shows `auth_scheme = KERBEROS` for the session.

### The whole flow, condensed

```text
Sidecar --1. AS-REQ: "I am svc-app" + time sealed with keytab key ------> DC
Sidecar <-2. AS-REP: TGT (sealed for the KDC) + session key 1 ----------- DC
             the sidecar writes both into the shared cache
App ------3. TGS-REQ: TGT + authenticator (session key 1) + SPN -------> DC
App <-----4. TGS-REP: service ticket (sealed for SQL) + session key 2 --- DC
App ------5. AP-REQ in the TDS login: ticket + authenticator ----------> SQL Server
             SQL Server opens it with its own key, matches the SID to a login
```

## What fails at each step

| Step | Typical error | Usual cause |
| --- | --- | --- |
| 1 | `Cannot find KDC for realm`, `Cannot contact any KDC for realm` | DNS cannot find the domain controllers, or port 88 is blocked |
| 1 | `Clock skew too great` | The pod's and the domain controller's clocks are more than 5 minutes apart |
| 1 | `Preauthentication failed`, `KDC has no support for encryption type` | The keytab is out of date (old kvno), or it and the account disagree on AES or RC4 |
| 3 and 4 | `Server not found in Kerberos database` | The SPN is missing or misspelled, or the client asked for another name (IP address, DNS alias, short name) instead of the FQDN |
| 5 | `Login failed for user 'CORP\svc-app'` | Kerberos worked, but SQL Server has no login for the account or its groups |

[04-poc-test-plan.md](../04-poc-test-plan.md) has the full error table and a check for each step.

## Lifetimes and renewal

AD's default domain policy issues user tickets that last 10 hours and can be renewed for up to 7 days. When the TGT expires, the client can no longer get new service tickets, so new database connections fail.

On Windows, the operating system refreshes tickets in the background. On Linux, nothing does unless you arrange it. Microsoft's ODBC driver documentation says "Kerberos credentials expire, and the ODBC driver doesn't renew them" ([Microsoft: Use Integrated Authentication](https://learn.microsoft.com/en-us/sql/connect/odbc/linux-mac/using-integrated-authentication)). The sidecar in this design runs `kinit` from the keytab every hour and gets a fresh TGT each time instead of renewing the old one.

A connection that is already open keeps working after the ticket expires. Kerberos is only used while logging in.

## Clocks

Tickets carry timestamps. Both the default AD domain policy and the MIT library default allow 5 minutes of clock difference; beyond that you get `Clock skew too great` ([Microsoft: Maximum tolerance for computer clock synchronization](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/security-policy-settings/maximum-tolerance-for-computer-clock-synchronization), [MIT krb5.conf, clockskew](https://web.mit.edu/kerberos/krb5-latest/doc/admin/conf_files/krb5_conf.html)). Pods use the node's clock. Red Hat CoreOS defaults to Amazon Time Sync on AWS, but ROSA docs do not state the node time source, so confirm it with Red Hat if skew errors appear. Domain controllers sync within the domain.

## DNS matters more than people expect

Kerberos names services by host name. The client builds the SPN from the server name it was given, so `sql01.corp.example.com` and `10.0.5.20` lead to different SPN requests, and only one of them exists in AD. The client also finds the KDCs through DNS SRV records such as `_kerberos._tcp.corp.example.com`. If the cluster cannot resolve the AD domain, Kerberos fails before it starts.

## Where to go next

- [MIT Kerberos: Kerberos V5 concepts](https://web.mit.edu/kerberos/krb5-latest/doc/basic/index.html), short pages on caches, keytabs and principals.
- [Microsoft: Kerberos authentication overview](https://learn.microsoft.com/en-us/windows-server/security/kerberos/kerberos-authentication-overview).
- [03-spn-keytab-and-service-accounts.md](03-spn-keytab-and-service-accounts.md).
