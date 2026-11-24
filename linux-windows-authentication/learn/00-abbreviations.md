# Abbreviations

Every abbreviation used in this pack, with what it means here. Longer definitions of the main terms are in the glossary, [../CONTEXT.md](../CONTEXT.md).

| Abbreviation | Stands for | What it means in this pack |
| --- | --- | --- |
| ACL | Access control list | A list of who or what may reach something |
| AD | Active Directory | Microsoft's directory of accounts, groups and computers. Its domain controllers are also the Kerberos KDCs |
| AES, AES128, AES256 | Advanced Encryption Standard (128- or 256-bit keys) | The encryption type the keytab must use. `AES256-SHA1` is the name `ktpass` uses for `aes256-cts-hmac-sha1-96` |
| AP-REQ | Application request | The Kerberos message in which a client hands a service ticket and an authenticator to a service, here SQL Server |
| API | Application programming interface | Mostly the Kubernetes API, which `oc` and the web console talk to |
| AS-REQ, AS-REP | Authentication Service request, reply | The Kerberos messages that get a TGT |
| AWS | Amazon Web Services | |
| AZ | Availability zone | One of the separate data-centre groups inside an AWS region |
| CA | Certificate authority | Issues TLS certificates. The app must trust the CA that signed SQL Server's certificate |
| ccache | Credential cache | Where a Linux Kerberos client keeps its tickets; a file in this design |
| CIDR | Classless Inter-Domain Routing | The notation for an IP address range, such as `10.0.16.0/20` |
| CNAME | Canonical name record | A DNS alias that points one name at another |
| CPU | Central processing unit | Processor time, one of the resources a pod asks for |
| CRI | Container Runtime Interface | The Kubernetes standard for talking to a container runtime |
| CRI-O | CRI plus OCI | OpenShift's container runtime, which runs on every node |
| CSI | Container Storage Interface | The Kubernetes standard for volume plugins. The Secrets Store CSI driver mounts secrets from AWS Secrets Manager |
| CVO | Cluster Version Operator | The OpenShift Operator that keeps all the cluster Operators at the right version |
| DBA | Database administrator | |
| DC | Domain controller | A Windows server that holds the AD database and runs the KDC |
| DNS | Domain Name System | Turns names such as `sql01.corp.example.com` into IP addresses |
| DSN | Data source name | A saved, named set of ODBC connection settings |
| EC2 | Elastic Compute Cloud | AWS virtual machines. SQL Server and the ROSA worker nodes run on EC2 |
| ECS | Elastic Container Service | AWS's own container service, the only place AWS supports gMSA for Linux containers |
| etcd | From `/etc` (the Unix configuration folder) and "d" for distributed | The Kubernetes database that stores every object, Secrets included |
| FQDN | Fully qualified domain name | A complete host name, such as `sql01.corp.example.com`. Kerberos needs it to build the SPN |
| GA | Generally available | A release with full vendor support |
| gMSA | Group Managed Service Account | An AD account whose password AD rotates and only Windows can fetch. Not usable on ROSA |
| GSSAPI, GSS | Generic Security Service Application Programming Interface | The standard interface through which the ODBC driver uses Kerberos on Linux |
| HCP | Hosted control planes | The ROSA model where Red Hat runs the control plane in its own AWS account |
| IAM | Identity and Access Management | AWS's system of users, roles and permissions |
| IP | Internet Protocol | As in IP address |
| JAAS | Java Authentication and Authorization Service | How Java programs configure Kerberos |
| JDBC | Java Database Connectivity | The Java equivalent of ODBC |
| KB | Knowledge Base | A Microsoft support article number, such as KB5073381 |
| KCM | Kerberos Cache Manager | A credential cache type held by a daemon (`sssd-kcm`) instead of a file |
| KDC | Key Distribution Center | The Kerberos service that issues tickets, on port 88. Every AD domain controller is one |
| KiB, MiB | Kibibyte (1,024 bytes), mebibyte (1,048,576 bytes) | Used for size limits on Secrets and memory |
| krb5 | Kerberos version 5 | The prefix of MIT Kerberos names: `krb5.conf`, `KRB5CCNAME`, `KRB5_TRACE` |
| `KRB5CCNAME` | Kerberos 5 credential cache name | The environment variable that tells the library where the ticket cache is |
| `krbtgt` | Kerberos ticket-granting ticket (account) | The KDC's own AD account. Its key seals every TGT |
| kvno | Key version number | Goes up each time an account's password changes. A keytab with an old kvno stops working |
| LDAP | Lightweight Directory Access Protocol | How programs query AD, on ports 389 and 636 |
| MIT | Massachusetts Institute of Technology | Where Kerberos was created. "MIT Kerberos" is the Kerberos library and tools used on Linux |
| `msodbcsql18` | Microsoft ODBC Driver 18 for SQL Server | The driver's package name on Linux |
| `MSSQLSvc` | Microsoft SQL Server service | The service class at the start of a SQL Server SPN: `MSSQLSvc/<FQDN>:<port>` |
| NACL | Network access control list | AWS allow and deny rules applied to a whole subnet |
| NTLM, NTLMv1 | NT LAN Manager (version 1) | The older Windows challenge-response login method. Deprecated |
| OCI | Open Container Initiative | The standard for container image and runtime formats |
| `oc` | OpenShift command-line interface | The OpenShift command-line tool. It does everything `kubectl` does, plus OpenShift commands |
| ODBC | Open Database Connectivity | A standard database client interface. pyodbc uses Microsoft's ODBC driver |
| OLM | Operator Lifecycle Manager | Installs and upgrades add-on Operators in OpenShift |
| OU | Organizational unit | A folder inside AD that holds accounts and sets policies for them |
| OVN-Kubernetes | Open Virtual Network for Kubernetes | OpenShift's network plugin. Gives pods their IP addresses and handles their traffic |
| PAC | Privilege Attribute Certificate | The part of an AD ticket that lists the account's SID and group SIDs |
| PKI | Public key infrastructure | A company's certificate system, including its CAs |
| PoC | Proof of concept | The test described in [../04-poc-test-plan.md](../04-poc-test-plan.md) |
| PR | Pull request | A proposed code change on GitHub |
| PTR | Pointer record | A reverse DNS record, from IP address to name |
| PWD | Password | Connection string keyword for SQL logins. Not used with Windows Authentication |
| pyodbc | Python ODBC | The Python module that calls the ODBC driver |
| RAM | Random-access memory | Memory. The ticket cache lives in RAM |
| RBAC | Role-based access control | How Kubernetes and OpenShift decide who may do what |
| RC4 | Rivest Cipher 4 | An old encryption type that Microsoft is retiring. The keytab must not depend on it |
| rdns | Reverse DNS | A `krb5.conf` setting; turned off in this design |
| RHCOS | Red Hat Enterprise Linux CoreOS | The operating system on OpenShift nodes |
| RHEL | Red Hat Enterprise Linux | |
| ROSA | Red Hat OpenShift Service on AWS | OpenShift run as a managed service on AWS |
| rpm | RPM Package Manager | The package tool on RHEL and UBI |
| SCC | Security Context Constraint | OpenShift's policy for what a pod may do. `restricted-v2` is the default |
| SHA1 | Secure Hash Algorithm 1 | Part of the name of the AES Kerberos encryption types |
| SID | Security identifier | The fixed ID Windows gives every account and group. SQL Server matches Windows logins by SID |
| SNAT | Source network address translation | Rewriting a packet's source address. Pod traffic leaves with the node's IP address because of it |
| SPID | Server process ID | SQL Server's ID for a session. `@@SPID` is the current one |
| SPN | Service principal name | The name a client asks the KDC for to get a service ticket |
| SPNEGO | Simple and Protected GSSAPI Negotiation Mechanism | The wrapper that carries the Kerberos ticket inside the TDS login |
| SQL | Structured Query Language | |
| SRE | Site reliability engineer | Red Hat's engineers who run ROSA clusters |
| SRV | Service record | A DNS record that lists the servers for a service, such as `_kerberos._tcp.corp.example.com` |
| SSL | Secure Sockets Layer | The older name for TLS |
| SSPI | Security Support Provider Interface | Windows' equivalent of GSSAPI. ODBC driver errors say "SSPI Provider" even on Linux |
| sssd | System Security Services Daemon | A Linux service for AD and Kerberos logins on whole machines. Not used in containers here |
| STS | Security Token Service | The AWS service that hands out temporary credentials. ROSA uses it |
| TCP | Transmission Control Protocol | |
| TDS | Tabular Data Stream | The network protocol SQL Server clients speak, on port 1433 |
| TGS-REQ, TGS-REP | Ticket-granting service request, reply | The Kerberos messages that get a service ticket |
| TGT | Ticket-granting ticket | The first ticket, used to ask the KDC for service tickets |
| TGW | Transit Gateway | AWS's hub for connecting VPCs and on-premises networks |
| TLS | Transport Layer Security | Encryption for network connections. `Encrypt=yes` turns it on |
| tmpfs | Temporary file system | A filesystem kept in RAM |
| TXT | Text record | A DNS record that holds free text |
| UBI | Universal Base Image | Red Hat's freely redistributable container base images |
| UDP | User Datagram Protocol | |
| UID | User ID | The number a Linux process runs as. Also a connection string keyword for a SQL login name |
| VM | Virtual machine | |
| VPC | Virtual private cloud | Your private network in AWS |
| VPN | Virtual private network | |
| YAML | YAML Ain't Markup Language | The plain-text format Kubernetes objects are usually written in |
