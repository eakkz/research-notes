# Map: Windows Authentication from Linux containers and ROSA to SQL Server

Label: wayfinder:map. Tracker: local markdown (tickets live in `wayfinder/tickets/`).

## Destination

A feasibility pack the owner can read and act on: candidate solutions, architecture diagrams,
a checklist of everything needed (accounts, SPNs, keytabs, DNS, firewall, secrets), and
beginner-level explainers for each component. No code and no network build.

## Notes

- Domain: identity (Active Directory, Kerberos, NTLM), SQL Server client drivers, OpenShift on AWS.
- Skills each session uses: grilling + domain-modeling for decisions, research for facts,
  humanizer on every document the owner will read.
- Override of "plan, don't do": this effort carries execution into the map. Writing the final
  documents is part of the route.
- Never put the company name in a web search or an external request.

## Decisions so far

- [Where SQL Server runs](tickets/04-sql-server-location.md): self-managed SQL Server on EC2, domain-joined.
- [Which AD domain and how AWS reaches it](tickets/05-ad-domain-and-reachability.md): unknown; document corporate AD over DX/VPN and AWS Managed AD with a trust.
- [Application language and SQL driver](tickets/06-app-runtime.md): Python with pyodbc and ODBC Driver 18. End goal is Airflow on OpenShift, but the research treats it as a generic Linux container.
- [Which identity the app uses at SQL Server](tickets/07-identity-model.md): one AD service account per app.
- [Credential storage and rotation rules](tickets/09-credential-policy.md): unknown; compare Kubernetes Secret, AWS Secrets Manager via CSI, and Vault, and list questions for security and AD teams.
- NTLM: include as a not-recommended fallback (recorded on [Credential storage and rotation rules](tickets/09-credential-policy.md)).
- ROSA flavor: a ROSA HCP cluster already exists (recorded on [What 'Linux container' covers](tickets/08-container-scope.md)).
- [Deliverable format and language](tickets/10-deliverable-format.md): Markdown with Mermaid diagrams, English.
- [Recommended solution and PoC plan](tickets/11-recommendation.md): option A (sidecar) as target, option B for the first PoC stage, NTLM fallback only; DNS via Route 53 Resolver rule, worker-subnet allowlist or EgressIP, OpenShift Secret for PoC. Written up in 01 to 04.
- [Airflow pointers for later](tickets/12-airflow-pointers.md): chart already ships the sidecar model; use OdbcHook; chart 1.22.0 sidecar keeps task pods Running.
- Main design: Kerberos sidecar is the target; the first PoC step runs kinit inside the app container. The docs must explain what a sidecar is from zero (recorded on [Recommended solution and PoC plan](tickets/11-recommendation.md)).
- The app's service account lives in the same AD domain as SQL Server, so there is no cross-realm Kerberos (recorded on [Which AD domain and how AWS reaches it](tickets/05-ad-domain-and-reachability.md)).
- Document layout: README, 01-feasibility-and-options, 02-architecture, 03-prerequisites-checklist, 04-poc-test-plan, learn/ (recorded on [Deliverable format and language](tickets/10-deliverable-format.md)).
- [What 'Linux container' covers](tickets/08-container-scope.md): ROSA is the target platform; explain everything as a plain Linux container running on it.

## Not yet specified

(none: destination reached 2026-11-23, see [README](../README.md))

## Out of scope

- Designing the Airflow deployment itself (chart values, executor choice, SCC fit). Facts are parked in [Airflow pointers for later](tickets/12-airflow-pointers.md); it needs its own map.

- Writing application code, manifests, or Terraform.
- Building network connectivity between AWS and on-prem.
