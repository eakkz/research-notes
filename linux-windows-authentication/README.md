# Windows Authentication from Linux containers on ROSA to SQL Server

Feasibility pack for one question: can a Python app (pyodbc with Microsoft ODBC Driver 18) running in a Linux container on ROSA HCP log in to self-managed SQL Server on EC2 with Windows Authentication, as one AD service account? The end goal is Apache Airflow on OpenShift; this pack treats the app as a plain Linux container.

## Short answer

Yes, with Kerberos. A small helper container in the pod (a sidecar) holds the service account's keytab, runs `kinit`, and keeps a fresh Kerberos ticket in a shared in-memory volume. The app reads that ticket and connects with `Trusted_Connection=yes`. Nothing needs a privileged pod or a change to ROSA nodes.

What does not work:

- gMSA. ROSA has no Windows containers, and AWS supports Linux gMSA only on ECS.
- NTLM with the Microsoft ODBC driver. The driver has no NTLM mode on Linux. NTLM through pymssql/FreeTDS is possible but deprecated, so it is kept as a fallback only.

Most of the work sits outside OpenShift:

- The AD team issues a service account and an AES keytab, and agrees how its password rotates.
- The DBA confirms the SPN and creates the login.
- The network team opens port 88 and AD DNS resolution to the ROSA worker subnets.

## Reading order

For a visual walkthrough first, open [architecture.html](architecture.html) in a browser: four diagrams (end to end, inside the pod, one login step by step, keytab rotation) with notes and the port table. Badges on figures 1 and 2 mark what already exists, what is new, and what is still to decide. PNG copies of all four figures are in [images/](images/), for pasting into OneNote, Word or slides.

Decision documents:

1. [01-feasibility-and-options.md](01-feasibility-and-options.md): verdict, the seven options compared, recommendation, risks, and a short Airflow section.
2. [02-architecture.md](02-architecture.md): diagrams for both AD variants, the inside of the pod, the login sequence, DNS, keytab rotation, and network flows.
3. [03-prerequisites-checklist.md](03-prerequisites-checklist.md): everything needed, grouped by owning team, with questions to ask each team.
4. [04-poc-test-plan.md](04-poc-test-plan.md): ordered checks to prove feasibility, and what each error message usually means.
5. [05-consultation-pre-read.md](05-consultation-pre-read.md): the short pre-read for the container platform team, asking whether a pattern already exists before the PoC. [05-consultation-pre-read.docx](05-consultation-pre-read.docx) is the same page as a Word file, for OneNote.

Learning material, if the terms are new. Keep [learn/00-abbreviations.md](learn/00-abbreviations.md) open while reading; it spells out every abbreviation in the pack.

1. [learn/01-windows-authentication-and-active-directory.md](learn/01-windows-authentication-and-active-directory.md)
2. [learn/02-kerberos.md](learn/02-kerberos.md): Kerberos from the start, then one login followed message by message.
3. [learn/03-spn-keytab-and-service-accounts.md](learn/03-spn-keytab-and-service-accounts.md)
4. [learn/04-linux-kerberos-client.md](learn/04-linux-kerberos-client.md)
5. [learn/05-sql-server-and-odbc-driver.md](learn/05-sql-server-and-odbc-driver.md)
6. [learn/06-kubernetes-openshift-and-pods.md](learn/06-kubernetes-openshift-and-pods.md): containers, Kubernetes architecture, pods, what OpenShift adds, and who runs what on ROSA HCP.
7. [learn/07-sidecars.md](learn/07-sidecars.md)
8. [learn/08-openshift-and-rosa-hcp.md](learn/08-openshift-and-rosa-hcp.md)
9. [learn/09-aws-networking-and-dns.md](learn/09-aws-networking-and-dns.md)
10. [learn/10-reading-list.md](learn/10-reading-list.md)

[CONTEXT.md](CONTEXT.md) is the glossary.

## Supporting material

- [research/](research/): the source notes behind every claim, one file per area, with a link on each fact and a list of what could not be confirmed.
- [wayfinder/MAP.md](wayfinder/MAP.md): the decisions made while scoping this pack and the tickets that hold them.

## Decisions taken while scoping

| Topic | Decision |
| --- | --- |
| SQL Server | Self-managed on EC2, joined to AD |
| AD domain | Not decided; corporate AD and AWS Managed AD both covered |
| Service account location | Always the same domain as SQL Server (no cross-realm Kerberos) |
| Client | Python, pyodbc, ODBC Driver 18 |
| Identity | One AD service account per app; no end-user delegation |
| Cluster | ROSA HCP, already running |
| Design | Kerberos sidecar as the target; first PoC step runs `kinit` in the app container |
| DNS | Route 53 Resolver rule first; DNS Operator forwarding as fallback |
| Firewall source | Worker subnet CIDRs for the PoC; EgressIP when a fixed IP is required |
| Keytab storage | OpenShift Secret for the PoC; AWS Secrets Manager (CSI) or External Secrets Operator for production, chosen with security |
| NTLM | Documented as a not-recommended fallback |

## Not confirmed yet (test in the PoC)

- Whether ODBC Driver 18 can use MIT's client keytab (`KRB5_CLIENT_KTNAME`) with no `kinit` at all.
- Whether the ticket cache can be mounted read-only into the app container.
- Whether NTLM signing, required by Windows servers from May 2027, affects NTLM logins from FreeTDS.
