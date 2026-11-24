# Reading list

Primary sources only: Microsoft, MIT, Red Hat, AWS, Kubernetes and project docs. Every link was opened on 2026-11-23 or 2026-11-24. Within each topic, read top to bottom; the first link is the one to read if you only read one.

## 1. Kerberos and Active Directory

1. [Kerberos authentication overview in Windows Server](https://learn.microsoft.com/en-us/windows-server/security/kerberos/kerberos-authentication-overview) (Microsoft). How AD hands out TGTs and service tickets.
2. [Kerberos V5 concepts](https://web.mit.edu/kerberos/krb5-latest/doc/basic/index.html) (MIT). Short pages on credential caches, keytabs, realms and principals.
3. [For users](https://web.mit.edu/kerberos/krb5-latest/doc/user/index.html) (MIT). `kinit`, `klist`, `kvno` and environment variables.
4. [Principal names and DNS](https://web.mit.edu/kerberos/krb5-latest/doc/admin/princ_dns.html) (MIT). Why reverse DNS and CNAMEs change the SPN a client asks for.
5. [krb5.conf reference](https://web.mit.edu/kerberos/krb5-latest/doc/admin/conf_files/krb5_conf.html) (MIT). Look up settings here when needed.

## 2. SQL Server, SPNs and keytabs

1. [Understanding Active Directory authentication for SQL Server on Linux and containers](https://learn.microsoft.com/en-us/sql/linux/security/authentication/understand-active-directory) (Microsoft). The clearest diagram-led explanation of keytabs and the ticket flow. Written for SQL Server running on Linux; the concepts carry over.
2. [Register a Service Principal Name for Kerberos connections](https://learn.microsoft.com/en-us/sql/database-engine/configure-windows/register-a-service-principal-name-for-kerberos-connections) (Microsoft). SPN formats, `setspn`, and the `auth_scheme` check.
3. [Determine the authentication type](https://learn.microsoft.com/en-us/troubleshoot/sql/database-engine/connect/determine-the-authentication-type) (Microsoft). How to tell Kerberos from NTLM on a live connection.
4. [Overview of consistent authentication issues](https://learn.microsoft.com/en-us/troubleshoot/sql/database-engine/connect/consistent-authentication-connectivity-issues) (Microsoft). The usual SPN and DNS mistakes and their error messages.
5. [ktpass](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/ktpass) (Microsoft). The tool the AD team will probably use to build the keytab.
6. [Rotate SQL Server on Linux keytabs](https://learn.microsoft.com/en-us/sql/linux/security/authentication/rotate-keytabs) (Microsoft). The kvno-first rotation order.
7. [Detect and remediate RC4 usage in Kerberos](https://learn.microsoft.com/en-us/windows-server/security/kerberos/detect-remediate-rc4-kerberos) (Microsoft). Why the keytab must be AES.
8. [Group Managed Service Accounts overview](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/group-managed-service-accounts/group-managed-service-accounts/group-managed-service-accounts-overview) (Microsoft). Background for the "why not gMSA" question.

## 3. The ODBC driver and pyodbc

1. [Use Integrated Authentication, ODBC Driver for SQL Server](https://learn.microsoft.com/en-us/sql/connect/odbc/linux-mac/using-integrated-authentication) (Microsoft). The single most relevant page: tickets, keytabs, `ServerSPN`, no NTLM fallback, no renewal.
2. [DSN and connection string keywords](https://learn.microsoft.com/en-us/sql/connect/odbc/dsn-connection-string-attribute) (Microsoft). `Trusted_Connection`, `Encrypt`, `ServerSPN`.
3. [The pyodbc module](https://github.com/mkleehammer/pyodbc/wiki/The-pyodbc-Module) (pyodbc wiki). How the connection string is passed through.
4. [Using Kerberos integrated authentication, JDBC driver](https://learn.microsoft.com/en-us/sql/connect/jdbc/using-kerberos-integrated-authentication-to-connect-to-sql-server) (Microsoft). Only if a Java app joins later.

## 4. Kubernetes, pods and sidecars

1. [Pods](https://kubernetes.io/docs/concepts/workloads/pods/) (Kubernetes).
2. [Kubernetes Components](https://kubernetes.io/docs/concepts/overview/components/) (Kubernetes). One line on each part of the control plane and the nodes.
3. [Cluster Architecture](https://kubernetes.io/docs/concepts/architecture/) (Kubernetes). The longer version, with a diagram.
4. [Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/) (Kubernetes). Phases, restarts and probes.
5. [Sidecar containers](https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/) (Kubernetes). Native sidecars and their start and stop order.
6. [Kerberos Sidecar Container](https://www.redhat.com/en/blog/kerberos-sidecar-container) (Red Hat, 2018). The original write-up of this exact pattern for the ODBC driver on OpenShift. The image it uses is old; the design still holds.
7. [Volumes: emptyDir](https://kubernetes.io/docs/reference/storage/volume-types/#emptydir) (Kubernetes).
8. [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/) (Kubernetes). Base64 data, size limit, tmpfs mounts, etcd caveats.

## 5. OpenShift and ROSA HCP

1. [Policies and service definition](https://docs.redhat.com/en/documentation/red_hat_openshift_service_on_aws/4/html/introduction_to_rosa/policies-and-service-definition) (Red Hat). What ROSA customers can and cannot change, including Windows support and DNS forwarding.
2. [Introduction to ROSA](https://docs.redhat.com/en/documentation/red_hat_openshift_service_on_aws/4/html-single/introduction_to_rosa/index) (Red Hat). The hosted control plane and classic models compared: which account runs what, PrivateLink, availability zones.
3. [Control plane architecture](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/architecture/control-plane) (Red Hat). CRI-O, the kubelet, and how Operators run the rest of OpenShift.
4. [Managing security context constraints](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/authentication_and_authorization/managing-pod-security-policies) (Red Hat). `restricted-v2` and UID ranges.
5. [Creating images](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/images/creating-images) (Red Hat). Section "Support arbitrary user ids".
6. [About networking for managed clusters](https://docs.redhat.com/en/documentation/red_hat_openshift_service_on_aws/4/html/networking_overview/about-managed-networking) (Red Hat).
7. [DNS Operator in ROSA](https://docs.redhat.com/en/documentation/red_hat_openshift_service_on_aws/4/html/networking_operators/dns-operator) (Red Hat). Zone forwarding.
8. [Assign a consistent egress IP](https://docs.redhat.com/en/documentation/red_hat_openshift_service_on_aws/4/html/tutorials/cloud-experts-consistent-egress-ip) (Red Hat). EgressIP on ROSA HCP.
9. [Use AWS Secrets Manager CSI on ROSA with STS](https://docs.redhat.com/en/documentation/red_hat_openshift_service_on_aws/4/html/tutorials/cloud-experts-aws-secret-manager) (Red Hat).

## 6. AWS networking and directory

1. [Forwarding outbound DNS queries to your network](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resolver-forwarding-outbound-queries.html) (AWS). Route 53 Resolver rules.
2. [AWS Managed Microsoft AD](https://docs.aws.amazon.com/directoryservice/latest/admin-guide/directory_microsoft_ad.html) (AWS).
3. [Creating a trust relationship](https://docs.aws.amazon.com/directoryservice/latest/admin-guide/ms_ad_setup_trust.html) (AWS).
4. [What is AWS Transit Gateway](https://docs.aws.amazon.com/vpc/latest/tgw/what-is-transit-gateway.html) (AWS).

## 7. For later: Airflow

1. [Production deployment: Kerberos](https://airflow.apache.org/docs/apache-airflow/stable/administration-and-deployment/production-deployment.html#production-deployment-kerberos) (Apache Airflow). The renewer and sidecar model.
2. [Kerberos](https://airflow.apache.org/docs/apache-airflow/stable/security/kerberos.html) (Apache Airflow). The `airflow kerberos` command and the `[kerberos]` settings.
3. [Helm chart pod template (1.22.0)](https://github.com/apache/airflow/blob/helm-chart/1.22.0/chart/files/pod-template-file.kubernetes-helm-yaml) (Apache Airflow). Where the chart puts the keytab, cache and sidecar.
4. [ODBC connection](https://airflow.apache.org/docs/apache-airflow-providers-odbc/stable/connections/odbc.html) (Apache Airflow). How `OdbcHook` builds the connection string from extras.

## Background on NTLM's retirement

1. [Deprecated features in the Windows client](https://learn.microsoft.com/en-us/windows/whats-new/deprecated-features) (Microsoft).
2. [Advancing Windows security: disabling NTLM by default](https://techcommunity.microsoft.com/blog/windows-itpro-blog/advancing-windows-security-disabling-ntlm-by-default/4489526) (Microsoft Windows IT Pro blog).

The research notes in [../research/](../research/) cite many more sources, one per claim.
