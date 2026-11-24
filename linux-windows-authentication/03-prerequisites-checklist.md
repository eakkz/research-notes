# Prerequisites checklist

Everything needed before the PoC can succeed, grouped by the team that usually owns it. Names follow the placeholders in [02-architecture.md](02-architecture.md). Each group ends with the questions to ask that team.

## Active Directory team

- [ ] Decide which domain: the corporate domain (variant A) or AWS Managed Microsoft AD (variant B). The service account must be in the same domain SQL Server is joined to.
- [ ] Create a user account for the app, for example `svc-app`. Keep the pre-Windows 2000 name (sAMAccountName) at 20 characters or fewer, because SQL Server logins based on AD users must be shorter than 21 characters ([CREATE LOGIN](https://learn.microsoft.com/en-us/sql/t-sql/statements/create-login-transact-sql)).
- [ ] Enable AES on the account ("This account supports Kerberos AES 128 bit encryption" and "... AES 256 bit encryption", which sets `msDS-SupportedEncryptionTypes` to `0x18`).
- [ ] Produce a keytab for `svc-app@CORP.EXAMPLE.COM` with AES256 keys and, if wanted, AES128 keys, and no RC4. With `ktpass`, always pass `/crypto` and use `/ptype KRB5_NT_PRINCIPAL` ([ktpass](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/ktpass)). Note the kvno in the keytab.
- [ ] Agree that building the keytab may reset the account password, and that any older keytab for the account stops working when that happens.
- [ ] Decide the password policy for the account: a never-expires exception, or a rotation procedure that follows the order in [02-architecture.md, section 6](02-architecture.md#6-keytab-rotation).
- [ ] Confirm the SQL Server SPN `MSSQLSvc/sql01.corp.example.com:1433` exists once, on the account that runs SQL Server (`setspn -Q MSSQLSvc/sql01.corp.example.com:1433`).
- [ ] Hand over the facts the cluster needs: realm name, domain DNS name, domain controller host names or IPs, AD DNS server IPs.
- [ ] Confirm the domain controllers keep accurate time. Kerberos rejects clocks more than 5 minutes apart. Red Hat CoreOS nodes on AWS default to Amazon Time Sync (per the CoreOS chrony configuration; ROSA docs do not state the node time source).

Questions to ask:

1. Which domain is SQL Server on EC2 joined to, and can we create a service account there?
2. Do you issue keytabs for service accounts? Which tool do you use, and which encryption types does it write?
3. What is the password expiry policy for service accounts, and can it be exempted?
4. Are the domain controllers on Windows Server 2025, or enforcing AES-only since the April 2026 update?
5. Is NTLM allowed for service accounts today, and is it scheduled to be blocked?
6. Who can see the SPN list for the SQL Server account, and is it clean (no duplicates)?

## SQL Server DBA

- [ ] Confirm SQL Server listens on a static TCP port, normally 1433, with TCP/IP enabled. Kerberos SPNs carry the port, so a dynamic port breaks them ([Register an SPN](https://learn.microsoft.com/en-us/sql/database-engine/configure-windows/register-a-service-principal-name-for-kerberos-connections), [Kerberos Configuration Manager article](https://learn.microsoft.com/en-us/troubleshoot/sql/database-engine/connect/using-kerberosmngr-sqlserver)).
- [ ] Confirm which account runs the SQL Server service and that the SPN is registered on it. SQL Server registers its own SPN at startup only for some account types or if it has permission.
- [ ] Create the login and database user:
  - `CREATE LOGIN [CORP\svc-app] FROM WINDOWS;` (use `DOMAIN\name`, not `name@domain`)
  - `CREATE USER [CORP\svc-app] FOR LOGIN [CORP\svc-app];` in each database the app uses, with only the rights it needs.
- [ ] Make sure the server certificate works with ODBC Driver 18, which encrypts by default and validates the certificate. The certificate name should match `sql01.corp.example.com` and chain to a CA the container trusts. `TrustServerCertificate=yes` is acceptable only in the PoC.
- [ ] Be ready to run the check query from the PoC plan, or grant `VIEW SERVER STATE` (`VIEW SERVER PERFORMANCE STATE` on SQL Server 2022 and later) to the tester.

Questions to ask:

1. Which account runs the SQL Server service, and is it a domain account, a gMSA or a virtual account?
2. Is the instance a default instance on 1433 or a named instance? If named, which static port?
3. Which certificate does SQL Server present, and which CA issued it?
4. Do existing Windows clients connect with Kerberos today? (`auth_scheme` from a remote Windows client tells you whether the SPN is already correct.)

## AWS network team

- [ ] Route from the ROSA worker subnets to the domain controllers. Variant A: Transit Gateway plus Direct Connect or VPN. Variant B: peering or Transit Gateway to the managed AD VPC.
- [ ] Route from the ROSA worker subnets to the SQL Server EC2 instance.
- [ ] Security groups, NACLs and on-prem firewalls open for the flows in [02-architecture.md, section 7](02-architecture.md#7-network-flows): 88 TCP/UDP and 53 TCP/UDP to the domain controllers, 1433 TCP to SQL Server.
- [ ] Allow the worker subnet CIDRs as the source, because node IPs change. If a firewall needs a single address, plan EgressIP with the platform team.
- [ ] DNS: check whether the ROSA VPC already resolves `corp.example.com`. If not, create a Route 53 Resolver outbound endpoint and a forwarding rule for the AD domain, and associate it with the ROSA VPC ([Route 53 outbound forwarding](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resolver-forwarding-outbound-queries.html)).

Questions to ask:

1. Is there already a Route 53 Resolver rule for the AD domain shared to our account?
2. Which on-prem firewalls sit between AWS and the domain controllers, and how long does a rule change take?
3. Are the domain controllers reachable from AWS at all today, and from which VPCs?

## OpenShift platform team (ROSA HCP)

- [ ] A project (namespace) for the PoC. The default `restricted-v2` SCC is enough; nothing in this design needs a privileged pod.
- [ ] If DNS goes through the cluster instead of Route 53: add a zone to `dns.operator/default` that forwards `corp.example.com` to the AD DNS servers ([ROSA DNS Operator](https://docs.redhat.com/en/documentation/red_hat_openshift_service_on_aws/4/html/networking_operators/dns-operator)).
- [ ] If a firewall needs fixed IPs: label a machine pool `k8s.ovn.org/egress-assignable=`, reserve addresses in the worker subnets, and create an `EgressIP` for the project ([ROSA HCP egress IP tutorial](https://docs.redhat.com/en/documentation/red_hat_openshift_service_on_aws/4/html/tutorials/cloud-experts-consistent-egress-ip)).
- [ ] Check which Kubernetes version the cluster runs, to know whether native sidecars are available (stable from Kubernetes 1.33).
- [ ] An image for the sidecar with the MIT Kerberos tools (`kinit`, `klist`). UBI images contain the Kerberos library but not the tools, so the image needs `krb5-workstation` installed.
- [ ] For production secrets, decide with security between the Secrets Store CSI driver with AWS Secrets Manager (installed with Helm as in the ROSA HCP tutorial, whose provider needs the `privileged` SCC) and the External Secrets Operator.

Questions to ask:

1. Which OpenShift and Kubernetes version does the ROSA HCP cluster run?
2. Is any secret integration (CSI driver, External Secrets Operator, Vault) already installed?
3. Are EgressIP or DNS Operator changes already in use on this cluster?

## Security team

- [ ] Approve storing a keytab in the cluster, and decide where: OpenShift Secret, AWS Secrets Manager, or Vault.
- [ ] Decide who may read the Secret (RBAC on the project).
- [ ] Approve the password policy for the service account (exception or rotation).
- [ ] Decide whether NTLM may be used at all, even as a fallback.

Questions to ask:

1. Is a keytab treated like a password under our policy? (It should be: anyone holding it can act as the account.)
2. Is etcd encryption, or an external secret store, required for this class of secret?
3. Is an exception from password expiry allowed if rotation is manual and logged?

## App team

- [ ] App image with Python, pyodbc, unixODBC, Microsoft ODBC Driver 18 and the MIT Kerberos libraries.
- [ ] Connection string using the SQL Server's fully qualified name, never an IP address or a DNS alias:
  `Driver={ODBC Driver 18 for SQL Server};Server=sql01.corp.example.com,1433;Database=<db>;Trusted_Connection=yes;Encrypt=yes;`
  No `UID` or `PWD`. Add `ServerSPN=MSSQLSvc/sql01.corp.example.com:1433` if the name the app uses differs from the SPN.
- [ ] Environment variables in the app container: `KRB5CCNAME=FILE:/var/kerberos-ccache/cache` and `KRB5_CONFIG` pointing at the mounted `krb5.conf`.
- [ ] Do not set `runAsUser` differently on the app and the sidecar.
- [ ] Know that a connection that is already open stays logged in after its ticket expires. Only new connections need a valid ticket, so a pool hides renewal problems until it opens a new connection.

## Files the pod needs

| Item | Where it comes from | Mounted into |
| --- | --- | --- |
| `svc-app.keytab` | AD team, stored as a Secret | Sidecar only (plus the init container in the fallback) |
| `krb5.conf` | Platform or app team, stored as a ConfigMap | All containers |
| Ticket cache volume | `emptyDir` with `medium: Memory` | Sidecar and app |
| CA certificate for SQL Server TLS | DBA or PKI team | App container (in the system trust store) |

A minimal `krb5.conf` for this design, built from Microsoft's sample for the Linux ODBC driver and MIT's settings:

```ini
[libdefaults]
  default_realm = CORP.EXAMPLE.COM
  dns_lookup_realm = false
  dns_lookup_kdc = true
  rdns = false
  dns_canonicalize_hostname = false
  udp_preference_limit = 0
  forwardable = true

[domain_realm]
  .corp.example.com = CORP.EXAMPLE.COM
  corp.example.com = CORP.EXAMPLE.COM
```

If DNS SRV records for the KDCs cannot be resolved from the cluster, set `dns_lookup_kdc = false` and list the domain controllers under `[realms]` with `kdc = dc01.corp.example.com`. [learn/04-linux-kerberos-client.md](learn/04-linux-kerberos-client.md) explains each line.
