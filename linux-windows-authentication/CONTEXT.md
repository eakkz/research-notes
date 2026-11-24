# Glossary

Terms as used across this folder. The `learn/` pages explain each one in context. Abbreviations are spelled out in [learn/00-abbreviations.md](learn/00-abbreviations.md).

**Windows Authentication**: SQL Server accepts a login that belongs to an Active Directory account (`DOMAIN\name`) instead of a SQL login with its own password. From a Linux client this means Kerberos, or NTLM where a driver supports it.
_Avoid_: treating "integrated security" or "trusted connection" as separate ideas; they are client-side names for the same thing.

**Service account**: The single AD user account an app runs as, for example `CORP\svc-app`. SQL Server sees this account, never the person who triggered the work.
_Avoid_: "user" for this account.

**Linux container**: Any OCI container on a Linux kernel. A pod on ROSA is one or more Linux containers sharing a network and volumes.

**SQL Server host**: The domain-joined EC2 instance that runs SQL Server.

**Active Directory (AD)**: Microsoft's directory of accounts, groups and computers. Its domain controllers also run the Kerberos KDC, and usually DNS for the domain.

**Domain controller (DC)**: A Windows server that holds the AD database and answers logins. Every DC is a KDC.

**Realm**: The Kerberos name of an AD domain, in upper case (`CORP.EXAMPLE.COM`).

**Principal**: A named identity in a realm, such as `svc-app@CORP.EXAMPLE.COM`.

**KDC (Key Distribution Center)**: The Kerberos service that issues tickets, on port 88.

**TGT (ticket-granting ticket)**: The ticket you get after proving your identity once; used to ask for service tickets.

**Service ticket**: A ticket for one service, encrypted with that service's key. SQL Server accepts it with its own key, without asking the KDC; it may still contact a domain controller to check group data.

**Session key**: A random, short-lived key the KDC puts inside each ticket and also gives the client. The client uses it to prove it owns the ticket.

**Authenticator**: The client's name and the current time, encrypted with a session key and sent with a ticket. Stops a copied ticket from being reused.

**PAC (Privilege Attribute Certificate)**: The part of an AD ticket that lists the account's SID and its group SIDs, signed by the KDC.

**SID (security identifier)**: The fixed ID AD gives every account and group. A Windows login in SQL Server is tied to the account's SID, not its name.

**SPN (service principal name)**: The name a client asks the KDC for when it wants a service ticket. For SQL Server: `MSSQLSvc/<FQDN>:<port>`. It sits on the account that runs SQL Server.

**Keytab**: A file holding an account's long-term Kerberos keys. Works like a password.

**kvno (key version number)**: A counter that goes up each time the account's password changes. A keytab with an old kvno stops working.

**Encryption type**: The algorithm behind a key: AES256, AES128 or the retired RC4.

**Credential cache (ccache)**: Where a Linux Kerberos client keeps its tickets. In this design, a file on a shared in-memory volume, named by `KRB5CCNAME`.

**krb5.conf**: The MIT Kerberos client configuration file: default realm, KDC discovery, DNS behaviour.

**kinit**: The command that gets a TGT, here from a keytab.

**gMSA (group Managed Service Account)**: An AD account whose password AD rotates and only Windows can retrieve. Not usable on ROSA.

**NTLM**: The older challenge-response login method. Deprecated by Microsoft. The ODBC driver on Linux has no documented NTLM mode and never falls back to it.

**Kubernetes**: The open-source system that runs containers across a cluster of machines and keeps the cluster matching the objects you give it.

**OpenShift**: Red Hat's Kubernetes distribution: Kubernetes plus RHCOS nodes, Operators for every platform part, projects, SCCs and its own tools.

**Control plane**: The part of a cluster that decides: API server, etcd, scheduler and controllers. On ROSA HCP it runs in a Red Hat-owned AWS account.

**Worker node**: A machine that runs pods. On ROSA, an EC2 instance in your VPC.

**Project**: OpenShift's name for a Kubernetes namespace with extra annotations: a named area that groups objects and controls who may use them. Each project has its own UID range.

**Deployment**: The object that keeps a set number of copies of a pod running and replaces them on upgrade.

**Operator**: A controller that installs, configures and repairs one piece of software. OpenShift runs its own parts, such as DNS, through Operators.

**Kubernetes ServiceAccount**: The identity a pod uses when it calls the Kubernetes API.
_Avoid_: confusing it with the AD service account `CORP\svc-app`; the two are unrelated.

**Pod**: The unit OpenShift runs: one or more containers that share a network identity and volumes and live and die together.

**Sidecar**: A helper container in the same pod as the app, running for the pod's whole life. Here it keeps the Kerberos ticket fresh.

**Native sidecar**: An init container with `restartPolicy: Always`; Kubernetes starts it before the app and stops it after.

**Init container**: A container that runs to completion before the app containers start.

**emptyDir (memory)**: A scratch directory created for a pod in RAM and shared by its containers.

**SCC (Security Context Constraint)**: OpenShift's policy for what a pod may do. `restricted-v2` is the default.

**ROSA HCP**: Red Hat OpenShift Service on AWS with hosted control planes. Worker nodes run in your VPC; Red Hat runs the control plane.

**EgressIP**: An OpenShift object that gives a project's outbound traffic fixed source IP addresses.

**Route 53 Resolver rule**: An AWS rule that forwards DNS queries for a named domain from a VPC to other DNS servers.

**AWS Managed Microsoft AD**: An AD domain whose domain controllers AWS runs in your VPC.
