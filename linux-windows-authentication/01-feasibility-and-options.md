# Feasibility and options

Question: can a Python app in a Linux container on ROSA HCP log in to SQL Server with Windows Authentication?

Scope settled with the owner:

- SQL Server is self-managed on EC2 and joined to an AD domain.
- The app uses Python with pyodbc and Microsoft ODBC Driver 18.
- SQL Server sees one AD service account per app, never the end user.
- The cluster is ROSA with hosted control planes (HCP), already running.
- The AD domain is not decided yet, so this pack covers corporate AD and AWS Managed Microsoft AD. In both cases the service account lives in the same domain as SQL Server.
- The end goal is Apache Airflow on OpenShift. Airflow only gets a short section at the end.

## Verdict

Yes, it is feasible, using Kerberos. On the OpenShift side, a pod needs only a keytab, a `krb5.conf` file, a shared in-memory volume and a small helper container. Most of the work and the schedule risk sit with other teams. The AD team has to issue a service account and an AES keytab and agree on how its password rotates. The DBA has to confirm the SPN and create the login. The network team has to let the ROSA worker subnets reach the domain controllers on port 88 and resolve the AD DNS zone.

gMSA is not possible on ROSA. NTLM is possible only with a different Python driver, and Microsoft has deprecated it, so it stays a fallback.

## Why this is harder from Linux than from Windows

On a domain-joined Windows server, "Windows Authentication" happens without any setup in the app. Windows already holds a Kerberos ticket for the logged-on account and hands it to the SQL driver through SSPI. If Kerberos fails, Windows falls back to NTLM without telling the app.

A Linux container has none of that. It is not joined to the domain, nobody logs on to it, and OpenShift runs it under an unpredictable user ID taken from the project's range. Three things therefore have to be supplied from outside:

1. An identity: the AD service account's secret, stored as a keytab file.
2. A way to turn that secret into a ticket and keep it fresh: running `kinit`, again and again before the ticket expires.
3. A path to Active Directory: DNS that resolves the AD domain, and a network route to the domain controllers.

Microsoft's ODBC driver on Linux makes this stricter. It supports Kerberos only. It has no NTLM mode and does not fall back to NTLM when Kerberos fails, and it never renews tickets itself ([Microsoft: Use Integrated Authentication, ODBC Driver](https://learn.microsoft.com/en-us/sql/connect/odbc/linux-mac/using-integrated-authentication)). Something next to the app has to keep a valid ticket in a place the driver can read.

## Options

| | Option | Works with pyodbc + ODBC 18 | Works on ROSA HCP | Main cost or risk | Verdict |
| --- | --- | --- | --- | --- | --- |
| A | Kerberos sidecar with a keytab | Yes | Yes | One extra container per pod; startup ordering | Recommended target |
| B | Kerberos inside the app container | Yes | Yes | Keytab sits next to the app code | First PoC step |
| C | Kerberos with a password instead of a keytab | Yes | Yes | Password stored in the cluster; still breaks on password change | Possible variant, not preferred |
| D | NTLM with `DOMAIN\user` and password | No (needs pymssql/FreeTDS) | Yes | Deprecated by Microsoft; may be blocked by AD policy | Fallback only |
| E | gMSA | No | No | No Windows containers on ROSA; Linux gMSA is ECS-only | Not feasible |
| F | Windows intermediary service | Not applicable | Yes, as a Windows VM | Another service to build and run | Only if Kerberos from Linux is refused |
| G | Not Windows auth (SQL login, Entra ID) | Yes | Yes | Changes the requirement | Out of scope, listed for completeness |

### A. Kerberos sidecar with a keytab (recommended)

A sidecar is a second, small container inside the same pod as the app. [learn/07-sidecars.md](learn/07-sidecars.md) explains it from zero. The sidecar and the app container split the work:

- The sidecar mounts the keytab, runs `kinit` with it, and writes the ticket into a shared volume backed by memory. It repeats this on a timer, every hour in Red Hat's example, so the ticket never expires.
- The app container mounts the same volume and sets `KRB5CCNAME` to the ticket file. pyodbc and the ODBC driver find the ticket there and use it to log in.
- The app container never sees the keytab.

Red Hat published this pattern for Kerberos-dependent apps on OpenShift; apps using the Microsoft ODBC driver for SQL Server were the motivating case, although the demo itself uses a test KDC and a sample client ([Red Hat blog: Kerberos Sidecar Container](https://www.redhat.com/en/blog/kerberos-sidecar-container)). The Airflow Helm chart ships the same design, so a PoC built this way carries over to the end goal.

Things to get right:

- The app must not start before the first ticket exists. Use an init container that runs `kinit` once, or a native sidecar with a startup check (`klist -s`).
- Both containers must run as the same user ID, because MIT Kerberos creates the ticket file with mode 0600. OpenShift's default `restricted-v2` SCC already gives every container in a pod the same UID, so the only rule is not to override `runAsUser` on one of them.
- Red Hat's UBI images default to a `KEYRING:` ticket cache, which is the wrong type for sharing between containers. Set `KRB5CCNAME=FILE:/<shared volume>/<file>` in both containers.
- If the sidecar is an ordinary container, a pod whose app finishes (a Job, or an Airflow task) stays `Running` because the sidecar never exits. Native sidecars (init containers with `restartPolicy: Always`, stable since Kubernetes 1.33) fix this ([Kubernetes: Sidecar containers](https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/)).

### B. Kerberos inside the app container

The app image's entrypoint runs `kinit -kt <keytab>` before starting Python and keeps a renewal loop in the background. It needs one container instead of two, so it is the quickest way to prove that DNS, network, SPN and keytab all work. The cost is that the keytab is mounted into the same container as the app code, and ticket renewal now depends on the app's entrypoint script.

MIT Kerberos can also fetch tickets by itself from a "client keytab" (`KRB5_CLIENT_KTNAME`) when no ticket exists ([MIT: keytab](https://web.mit.edu/kerberos/krb5-latest/doc/basic/keytab_def.html)). Microsoft does not document this for the ODBC driver, so treat it as a PoC experiment.

### C. Kerberos with a password

`kinit` can read the account password from standard input instead of a keytab. This avoids building a keytab, but the cluster then stores the password itself, which is just as sensitive. It still breaks when AD changes the password. Use it only if the AD team cannot produce keytabs.

### D. NTLM

NTLM logs in with `DOMAIN\user` and the password, with no tickets and no KDC. The Microsoft ODBC driver on Linux cannot do NTLM, so this option means switching to pymssql, which uses FreeTDS. FreeTDS switches to a domain login (NTLM) when the user name contains a backslash ([FreeTDS: Domain logins](https://www.freetds.org/userguide/domains.html), [freetds.conf `use ntlmv2`](https://www.freetds.org/userguide/freetdsconf.html)). Airflow's `MsSqlHook` also uses pymssql.

Reasons it stays a fallback:

- Microsoft deprecated all NTLM versions in June 2024. NTLMv1 is gone in Windows Server 2025, and network NTLM will be off by default in the next major Windows release ([Microsoft: Deprecated features](https://learn.microsoft.com/en-us/windows/whats-new/deprecated-features), [Windows IT Pro blog: Disabling NTLM by default](https://techcommunity.microsoft.com/blog/windows-itpro-blog/advancing-windows-security-disabling-ntlm-by-default/4489526)).
- From May 2027, Windows servers start requiring NTLM signing by default ([KB5135557](https://support.microsoft.com/en-us/servicing/os/windows/docs/2026/11/kb5135557-ntlm-signing-security-improvements)). Whether this affects NTLM inside a SQL Server login from FreeTDS is not confirmed.
- Many security teams already block NTLM for service accounts.

### E. gMSA

A group Managed Service Account is an AD account whose password Windows rotates by itself. Only Windows can retrieve that password. Kubernetes supports gMSA for Windows pods only ([Kubernetes: Configure GMSA](https://kubernetes.io/docs/tasks/configure-pod-container/configure-gmsa/)), and Windows containers are not available on ROSA ([ROSA HCP service definition](https://docs.redhat.com/en/documentation/red_hat_openshift_service_on_aws/4/html/introduction_to_rosa/policies-and-service-definition)). AWS's `credentials-fetcher` daemon gives gMSA to Linux containers, but AWS documents it only for Amazon ECS. It would have to run on the node itself, and ROSA customers cannot change node configuration ([AWS: gMSA for Linux containers on ECS](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/linux-gmsa.html)).

### F. Windows intermediary

A small service on a domain-joined Windows machine (an EC2 instance, or a Windows VM on OpenShift Virtualization, which ROSA HCP supports) connects to SQL Server with normal Windows authentication and exposes an API to the Linux app. This works, but it adds a service to build, secure and operate. Consider it only if security refuses keytabs in the cluster.

### G. Not Windows authentication

Two ways to drop the requirement altogether:

- A SQL Server login with its own password removes AD from the picture.
- SQL Server 2022 and later can accept Microsoft Entra ID logins. The usual route is registering the host with Azure Arc; a manual setup without Arc exists for Windows only ([Microsoft: Entra authentication for SQL Server](https://learn.microsoft.com/en-us/sql/relational-databases/security/authentication-access/azure-ad-authentication-sql-server-overview)).

Both change the requirement, so they are outside this pack.

## Recommendation

1. Build the PoC with option B first. It proves the AD, DNS, network, SPN and keytab pieces with the fewest moving parts.
2. Move to option A for anything beyond the PoC, and for Airflow.
3. Keep D as a documented fallback only, pending a security decision.

Platform defaults agreed for ROSA HCP:

- DNS: check first whether the ROSA VPC already resolves the AD domain. If it does not, add a Route 53 Resolver forwarding rule for the AD domain. OpenShift DNS Operator zone forwarding is the fallback; both are documented for ROSA HCP.
- Egress: pod traffic leaves with the worker node's IP, and node IPs change. For the PoC, allow the worker subnet CIDRs on the domain controller and SQL Server firewalls. Use EgressIP when a firewall needs one fixed address.
- Keytab storage: a plain OpenShift Secret for the PoC. For production, AWS Secrets Manager through the Secrets Store CSI driver (the Helm method from the ROSA tutorial) or the External Secrets Operator. The security team picks.

## Risks to raise early

| Risk | Why it matters | Owner |
| --- | --- | --- |
| RC4-only keytab | Since April 2026 domain controllers assume AES-only for accounts with no encryption types set. A keytab with only RC4 keys fails ([KB5073381](https://support.microsoft.com/en-us/topic/how-to-manage-kerberos-kdc-usage-of-rc4-for-service-account-ticket-issuance-changes-related-to-cve-2026-20833-1ebcda33-720a-4da8-93c1-b0496e1910dc)) | AD team |
| Password expiry | Any password change makes the keytab useless. A 90-day expiry policy breaks the app on day 90 unless rotation is planned | AD team, security |
| Wrong or missing SPN | The Linux ODBC driver has no NTLM fallback, so a bad SPN makes the login fail outright | DBA, AD team |
| Server name | Connecting by IP address or by a DNS alias makes the client ask for the wrong SPN | App team |
| DNS | Pods must resolve the AD domain; Kerberos fails with "Cannot find KDC" otherwise | Network team |
| TLS | ODBC Driver 18 encrypts by default and checks the server certificate. A self-signed SQL Server certificate fails before Kerberos even starts | DBA |
| Approval time | Keytabs for service accounts often need a security exception | Security |

## Airflow later

Facts to keep for the Airflow phase ([research/04-airflow-pointers.md](research/04-airflow-pointers.md) has the sources):

- The official Helm chart already implements option A. With `kerberos.enabled`, it mounts `krb5.conf`, a keytab Secret and an in-memory ticket cache. It also adds a `worker-kerberos` sidecar that runs `airflow kerberos` and an init container that runs `airflow kerberos -o` once.
- The official image ships `krb5-user` and `msodbcsql18`.
- Use `OdbcHook` (pyodbc) with an empty login and password and `Trusted_Connection=yes` in the connection extras. Airflow's own example spells it `TrustedConnection`; the hook passes keys through unchanged, so use the driver's spelling, `Trusted_Connection`. `MsSqlHook` uses pymssql, and its docs do not cover Windows authentication.
- Chart 1.22.0 runs the sidecar as an ordinary container. Task pods under the KubernetesExecutor can stay `Running` after the task ends. The native sidecar fix (PR #72555) is merged on `main` but not released yet.
- The chart sets fixed user IDs. A `securityContexts.disableDefaults` option that lets OpenShift assign UIDs is also on `main` only. Check how the released chart fits the `restricted-v2` SCC before planning.
- Whether the chart's sidecar reaches pods started by `KubernetesPodOperator` is not documented.
