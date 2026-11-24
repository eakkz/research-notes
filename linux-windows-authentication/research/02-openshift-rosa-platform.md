# OpenShift and ROSA platform facts for Kerberos from non-domain-joined pods

Research date: 2026-11-23. Scope: what Red Hat OpenShift Service on AWS (ROSA) and generic Linux containers offer or block when a pod that is not domain-joined needs a Kerberos ticket from Active Directory (AD) to reach Microsoft SQL Server with Windows Authentication.

Red Hat now publishes two ROSA doc sets. The set titled "Red Hat OpenShift Service on AWS" carries the banner "This is the documentation for the default architecture of Red Hat OpenShift Service on AWS. The classic architecture documentation is available here" ([ROSA DNS Operator page](https://docs.redhat.com/en/documentation/red_hat_openshift_service_on_aws/4/html/networking_operators/dns-operator)). This file calls that set "ROSA HCP" (hosted control planes) and the other set "ROSA classic".

Some facts below come from source code in official GitHub or GitLab repositories. Those reflect the default branch on the research date. Anything I could not confirm from a primary source is marked "not confirmed".

## 1. Kerberos sidecar pattern on OpenShift

### Red Hat's write-up

- The post is "Kerberos Sidecar Container" by Ed Seymour (Container Adoption Lead, EMEA), dated April 6, 2018. Its old URL was blog.openshift.com/kerberos-sidecar-container and it now lives at https://www.redhat.com/en/blog/kerberos-sidecar-container.
- The motivating case was existing applications that "used Kerberos to authenticate with external services, for example, using the Microsoft ODBC Driver for SQL Server" ([blog](https://www.redhat.com/en/blog/kerberos-sidecar-container)).
- The post compares three options: embedded, externalized and sidecar. It recommends the sidecar because the refresh logic stays separate from the application image and can be patched on its own release cadence ([blog](https://www.redhat.com/en/blog/kerberos-sidecar-container)).

### How the ticket cache is shared

- The post shares the token "using common namespaces, for example, shared memory". It writes the cache to the shared memory location by changing `default_ccache_name` in the image's built-in `/etc/krb5.conf` ([blog](https://www.redhat.com/en/blog/kerberos-sidecar-container)).
- The demo repository's `krb5.conf` sets `default_ccache_name=FILE:/dev/shm/ccache`, `default_client_keytab_name=/krb5/client.keytab`, `default_keytab_name=/krb5/krb5.keytab` and `includedir /etc/krb5.conf.d/` ([edseymour/kinit-sidecar krb5.conf](https://github.com/edseymour/kinit-sidecar/blob/master/krb5.conf)).
- The demo OpenShift template mounts one `emptyDir` volume with `medium: Memory`, named `ccache`, at `/dev/shm` in both the sidecar and the application container. It mounts a ConfigMap at `/etc/krb5.conf.d` in both containers ([example-client-deploy.yaml](https://github.com/edseymour/kinit-sidecar/blob/master/openshift/example-client-deploy.yaml)).
- Realm and KDC settings go in as ConfigMap files under the `/etc/krb5.conf.d` include directory ([blog](https://www.redhat.com/en/blog/kerberos-sidecar-container)).
- The blog provides the keytab from a Kubernetes Secret mounted at the default keytab path. In the demo template, the `keytabs` volume is declared as an `emptyDir` that the demo script fills ([blog](https://www.redhat.com/en/blog/kerberos-sidecar-container), [template](https://github.com/edseymour/kinit-sidecar/blob/master/openshift/example-client-deploy.yaml)).
- Kubernetes documents that "All containers in the Pod can read and write the same files in the `emptyDir` volume". With `medium: "Memory"`, Kubernetes mounts a tmpfs, and the files "count against the memory limit of the container that wrote them" ([Kubernetes volume types, emptyDir](https://kubernetes.io/docs/reference/storage/volume-types/#emptydir)).

### How tickets are renewed

- The entrypoint `rekinit.sh` loops forever. On each pass it runs `kinit -V $OPTIONS $APPEND_OPTIONS`, prints `klist -c /dev/shm/ccache`, then sleeps `PERIOD_SECONDS`, which defaults to 3600 ([blog](https://www.redhat.com/en/blog/kerberos-sidecar-container)).
- `OPTIONS` defaults to `-k` when `/krb5/krb5.keytab` exists and to `-k -i` when `/krb5/client.keytab` exists ([blog](https://www.redhat.com/en/blog/kerberos-sidecar-container)).
- Each pass gets a fresh ticket from the keytab rather than renewing the existing one. The post says Kubernetes handles any container failure, so nothing more sophisticated is needed ([blog](https://www.redhat.com/en/blog/kerberos-sidecar-container)).
- You must supply the principal explicitly. The post says it is "not possible to infer this information from the container's runtime environment" ([blog](https://www.redhat.com/en/blog/kerberos-sidecar-container)).
- The demo image is built `FROM centos:centos7`, sets `USER 1001` and declares `VOLUME ["/krb5","/dev/shm","/etc/krb5.conf.d"]` ([blog](https://www.redhat.com/en/blog/kerberos-sidecar-container)). The repository was last pushed on 2019-03-24 ([GitHub API metadata for edseymour/kinit-sidecar](https://github.com/edseymour/kinit-sidecar)).

### Newer Red Hat guidance

- I found no newer official Red Hat product documentation for a Kerberos client inside OpenShift pods (not confirmed that none exists).
- Red Hat Jira RFE-4794, "Provide supported implementation of Kerberos inside OCP pods", was created 2023-10-16 and last updated 2026-10-20. It has status "Approved", no resolution, and component "RHEL CoreOS" ([RFE-4794](https://redhat.atlassian.net/browse/RFE-4794)).
  - It asks for "Needed packages on the nodes and into container images", "Needed krb5.conf configuration", "Needed cache mechanism implemented to store credentials" and "Needed SELinux policies and/or seccompProfiles".
  - Its main use case is Kerberos NFS shares.
- Red Hat knowledge base solution 3464981, "how to implement the Kerberos client for processes running in containers", is marked "Solution in progress", was last updated May 31, 2018, and is subscriber-only. Its resolution text was not visible (not confirmed) ([KB 3464981](https://access.redhat.com/node/3464981)).
- Kubernetes now has native sidecar containers: init containers with `restartPolicy: Always`, "Stable since Kubernetes v1.33". They "share the same network and storage namespaces with the primary container" ([Kubernetes sidecar containers](https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/)). Which OpenShift version first ships this as GA: not confirmed.

## 2. Default ccache type in RHEL/UBI images and container sharing

### What the UBI images ship

On 2026-11-23 I pulled the public amd64 images `registry.access.redhat.com/ubi9/ubi:latest` and `ubi9/ubi-minimal:latest` (both "Red Hat Enterprise Linux release 9.9") and read the image layer.

- Both images contain `/etc/krb5.conf` with `includedir /etc/krb5.conf.d/` and `default_ccache_name = KEYRING:persistent:%{uid}`.
- Both include `libkrb5.so.3`.
- Neither includes `kinit`, `klist` or an `sssd-kcm` drop-in. I checked this by inspecting the image layers, so there is no document URL for the observation itself.

The build source explains the line. The RHEL-derived krb5 spec sets `configured_default_ccache_name KEYRING:persistent:%%{uid}` and injects it into `/etc/krb5.conf` at build time. This is the same in CentOS Stream 9 and 10 ([krb5.spec c9s](https://gitlab.com/redhat/centos-stream/rpms/krb5/-/blob/c9s/krb5.spec), [krb5.spec c10s](https://gitlab.com/redhat/centos-stream/rpms/krb5/-/blob/c10s/krb5.spec)).

### Why KCM and KEYRING do not suit cross-container sharing

- KCM works by "contacting a daemon process called kcm" ([MIT credential cache types](https://web.mit.edu/kerberos/krb5-latest/doc/basic/ccache_def.html)).
- When `sssd-kcm` is installed, `/etc/krb5.conf.d/kcm_default_ccache` sets `default_ccache_name = KCM:`. The KCM server listens on a UNIX socket that "can be bind-mounted to only some containers" ([Fedora change: Kerberos KCM credential cache by default](https://fedoraproject.org/wiki/Changes/KerberosKCMCache)). So a KCM cache needs a reachable daemon and socket, and UBI does not ship one (observation above).
- KEYRING "uses the kernel keyring support to store credential data in unswappable kernel memory". `KEYRING:persistent:uidnumber` is a "persistent per-UID collection" ([MIT](https://web.mit.edu/kerberos/krb5-latest/doc/basic/ccache_def.html)).
- Red Hat engineers changed FreeIPA so it does not configure KEYRING in containers, because "Kernel keyrings are not namespaced yet. Keyrings can leak into other containers. Therefore keyrings should not be used in containerized environment" ([freeipa commit 165a941](https://github.com/freeipa/freeipa/commit/165a941109a9a5f7ac8f85bdda93b4132875a7b1)).
- The restricted-v2 SCC forces `seccompProfiles: runtime/default` ([restricted-v2 manifest](https://github.com/openshift/cluster-kube-apiserver-operator/blob/main/bindata/bootkube/scc-manifests/0000_20_kube-apiserver-operator_00_scc-restricted-v2.yaml)).
- CRI-O's internal default seccomp profile is built from `go.podman.io/common`'s `seccomp.DefaultProfile()` ([CRI-O seccomp.go](https://github.com/cri-o/cri-o/blob/main/internal/config/seccomp/seccomp.go)). That profile allows `keyctl` but does not list `add_key` or `request_key`, and its default action is `SCMP_ACT_ERRNO` with errno 38, which is ENOSYS ([container-libs seccomp.json](https://github.com/containers/container-libs/blob/main/common/pkg/seccomp/seccomp.json)).
- Taken together, this points to KEYRING cache creation failing in a restricted-v2 pod. No Red Hat document states this, and I did not test it (not confirmed). I also did not confirm that a given OpenShift release ships this exact profile.

### Setting needed for a shared FILE cache

- MIT resolves the default cache in this order ([MIT](https://web.mit.edu/kerberos/krb5-latest/doc/basic/ccache_def.html)):
  1. the `KRB5CCNAME` environment variable
  2. `default_ccache_name` in `[libdefaults]`
  3. the compiled-in DEFCCNAME
- So you can share a cache by setting `KRB5CCNAME=FILE:/<shared-volume>/<name>` in both containers, or by mounting a `krb5.conf.d` snippet with `default_ccache_name = FILE:/<shared-volume>/<name>`. The Red Hat demo used `FILE:/dev/shm/ccache` ([kinit-sidecar krb5.conf](https://github.com/edseymour/kinit-sidecar/blob/master/krb5.conf)).
- OpenJDK's `FileCredentialsCache.getDefaultCacheName()` looks up the cache in this order:
  1. `KRB5CCNAME`, where only the `FILE:` type is supported and the prefix is stripped
  2. `/tmp/krb5cc_<uid>`
  3. paths under `user.home`

  It does not read `default_ccache_name` from krb5.conf, so Java needs `KRB5CCNAME` or the `/tmp/krb5cc_<uid>` path ([FileCredentialsCache.java](https://github.com/openjdk/jdk/blob/master/src/java.security.jgss/share/classes/sun/security/krb5/internal/ccache/FileCredentialsCache.java)).

## 3. Arbitrary UIDs and restricted-v2 SCC

### How OpenShift assigns the UID

- "By default, OpenShift Container Platform runs containers using an arbitrarily assigned user ID." Files that processes write must be owned by the root group and be group read/writable (`chgrp -R 0` and `chmod -R g=u`), because "the container user is always a member of the root group" ([OCP 4.19 Creating images](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/images/creating-images)).
- restricted-v2 is "the most restrictive SCC provided by a new installation and will be used by default for authenticated users" ([OCP 4.19 Managing SCCs](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/authentication_and_authorization/managing-pod-security-policies)). It differs from the older restricted SCC in four ways:
  - it drops ALL capabilities
  - it allows only `NET_BIND_SERVICE` to be added
  - it sets `seccompProfile` to `runtime/default` by default
  - it requires `allowPrivilegeEscalation` to be unset or false
- The manifest also sets these fields ([restricted-v2 manifest](https://github.com/openshift/cluster-kube-apiserver-operator/blob/main/bindata/bootkube/scc-manifests/0000_20_kube-apiserver-operator_00_scc-restricted-v2.yaml)):
  - `runAsUser: MustRunAsRange`
  - `fsGroup: MustRunAs`
  - `seLinuxContext: MustRunAs`
  - allowed volumes `configMap`, `csi`, `downwardAPI`, `emptyDir`, `ephemeral`, `image`, `persistentVolumeClaim`, `projected` and `secret`
- When a pod does not request a UID, restricted-v2 reads the project's `openshift.io/sa.scc.uid-range` annotation, and "a container will have runAsUser equal to the first value of the range" ([OCP 4.19 Managing SCCs](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/authentication_and_authorization/managing-pod-security-policies)).

### Effect on ccache file ownership

- MIT krb5 creates FILE caches with mode 0600. The source comment reads "Ignore user's umask, set mode = 0600" ([krb5 cc_file.c](https://github.com/krb5/krb5/blob/master/src/lib/krb5/ccache/cc_file.c)).
- So a sidecar and an application container that both take the default `runAsUser` get the same UID and can share the cache file. If they run as different UIDs, group permissions such as `fsGroup` do not help, because the file has no group bits. This follows from the two sources above and was not tested.

### Usernames, Java and nss_wrapper

- "CRI-O supports the insertion of arbitrary user IDs into a container's /etc/passwd file." The same page also says "the /etc/passwd file should not exist in any container image. If it does, the CRI-O container runtime will fail to inject a random UID" ([OCP 4.19 Creating images](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/images/creating-images)).
- CRI-O's `GeneratePasswd` adds an entry only when all three of these hold ([CRI-O utils.go](https://github.com/cri-o/cri-o/blob/main/utils/utils.go)):
  - the UID is missing from the image's `/etc/passwd`
  - that file exists and is non-empty
  - the file is not group- or world-writable and not owner-writable by that UID

  It uses the username `default` and home `/tmp` when none is given. The docs and the source describe the no-file case differently; actual behaviour when `/etc/passwd` is absent: not confirmed.
- Java reads its `user.name` property from `getpwuid(getuid())` and falls back to `"?"` when there is no passwd entry ([OpenJDK java_props_md.c](https://github.com/openjdk/jdk/blob/master/src/java.base/unix/native/libjava/java_props_md.c)). The JDK's default cache path uses the numeric UID first ([FileCredentialsCache.java](https://github.com/openjdk/jdk/blob/master/src/java.security.jgss/share/classes/sun/security/krb5/internal/ccache/FileCredentialsCache.java)). Supplying the principal and `KRB5CCNAME` explicitly avoids any dependency on a username. That is a reading of the code; no Red Hat document states it.
- nss_wrapper: Red Hat's sclorg images still install `nss_wrapper-libs`, for example the PostgreSQL image ([sclorg postgresql-container Dockerfile](https://github.com/sclorg/postgresql-container/blob/master/src/Dockerfile)). The current OpenShift image guidelines rely on CRI-O passwd injection and do not mention nss_wrapper. Whether any Kerberos client on OpenShift needs nss_wrapper: not confirmed.

## 4. DNS forwarding for the AD zone

### DNS Operator forwarding

- Both ROSA doc sets document "Use DNS forwarding" with `oc edit dns.operator/default` and `spec.servers[].zones` plus `forwardPlugin.upstreams` ([ROSA HCP DNS Operator](https://docs.redhat.com/en/documentation/red_hat_openshift_service_on_aws/4/html/networking_operators/dns-operator), [ROSA classic DNS Operator](https://docs.redhat.com/en/documentation/red_hat_openshift_service_on_aws_classic_architecture/4/html/networking_operators/dns-operator)).
- The Operator writes the result into the `dns-default` ConfigMap.
- You "must specify at least one zone", and forwarding should go "only to specific zones, such as your intranet".
- `forwardPlugin.policy` defaults to `Random`, and each `forwardPlugin` allows no more than 15 upstreams.
- Queries with no matching zone fall back to the upstream resolvers, which default to the node's `/etc/resolv.conf` ([ROSA HCP DNS Operator](https://docs.redhat.com/en/documentation/red_hat_openshift_service_on_aws/4/html/networking_operators/dns-operator)).

### Is it allowed on ROSA

- Both ROSA service definitions say: "For Red Hat OpenShift Service on AWS clusters that have a private cloud network configuration, a customer can specify internal DNS servers available on that private connection that should be queried for explicitly provided domains" ([ROSA HCP service definition](https://docs.redhat.com/en/documentation/red_hat_openshift_service_on_aws/4/html/introduction_to_rosa/policies-and-service-definition), [ROSA classic service definition](https://docs.redhat.com/en/documentation/red_hat_openshift_service_on_aws_classic_architecture/4/html/introduction_to_rosa/policies-and-service-definition)).
- AWS's ROSA responsibility matrix lists "Configure any necessary DNS forwarding rules" as a customer task ([AWS ROSA responsibilities](https://docs.aws.amazon.com/rosa/latest/userguide/rosa-responsibilities.html)).
- Red Hat KB 6971494, "DNS forwarding in OSD/ROSA clusters" (marked "Solution unverified", updated January 17, 2024), says "The DNS forwarding is possible in OSD and ROSA clusters" through the DNS operator ([KB 6971494](https://access.redhat.com/solutions/6971494)).
- Permissions:
  - The ROSA classic service definition lists `operator.openshift.io` among the API groups in which "Managed OpenShift customers may not manage any objects" ([ROSA classic service definition](https://docs.redhat.com/en/documentation/red_hat_openshift_service_on_aws_classic_architecture/4/html/introduction_to_rosa/policies-and-service-definition)).
  - Red Hat SRE's cluster config grants a specific exception. The `dedicated-admins-aggregate-cluster` ClusterRole, in a block labelled "DNS Forwarder", allows `get`, `list`, `patch`, `update` and `watch` on `dnses.operator.openshift.io` named `default` ([managed-cluster-config ClusterRole](https://github.com/openshift/managed-cluster-config/blob/master/deploy/rbac-permissions-operator-config/05-dedicated-admins-aggregate-cluster.ClusterRole.yaml)).
  - The same rule appears in the ACM policy placed on clusters labelled `hypershift.open-cluster-management.io/hosted-cluster: "true"`, which means it also applies to HCP ([ACM policy](https://github.com/openshift/managed-cluster-config/blob/master/deploy/acm-policies/50-GENERATED-rbac-permissions-operator-config.Policy.yaml)).
  - Whether a validating webhook restricts which DNS fields a dedicated-admin may change: not confirmed.

### VPC-level alternative and on-prem DNS docs

- Route 53 Resolver outbound endpoints and forwarding rules forward queries for a named domain from a VPC to resolvers on your network over Direct Connect, VPN or NAT ([Route 53 outbound forwarding](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resolver-forwarding-outbound-queries.html)).
- ROSA classic supports custom DHCP option sets. By default, clusters need "domain name servers" set to `AmazonProvidedDNS`, and custom DNS servers require extra configuration: forward the cluster's `p1.openshiftapps.com` private hosted zone to a Route 53 Inbound Resolver ([ROSA classic custom DNS resolver tutorial](https://docs.redhat.com/en/documentation/red_hat_openshift_service_on_aws_classic_architecture/4/html/tutorials/cloud-experts-custom-dns-resolver)).
- An HCP version, "Tutorial: Deploy Red Hat OpenShift Service on AWS with a custom DNS resolver", is listed in the ROSA HCP table of contents ([ROSA HCP Networking Operators, single page](https://docs.redhat.com/en/documentation/red_hat_openshift_service_on_aws/4/html-single/networking_operators/index)). I did not read its zone list (not confirmed).
- I found no ROSA document specific to forwarding an AD domain to on-prem AD DNS (not confirmed that none exists).
- For Kerberos specifically, MIT's `dns_lookup_kdc` "indicate[s] whether DNS SRV records should be used to locate the KDCs" ([MIT krb5.conf](https://web.mit.edu/kerberos/krb5-latest/doc/admin/conf_files/krb5_conf.html)). So the forwarded zone must answer SRV queries if you rely on KDC discovery instead of listing `kdc =` entries.

## 5. Egress source IP and EgressIP

### Default source IP

- ROSA classic service definition: NAT gateways present "a public, static IP for any public outbound traffic", with up to 3 per cluster (one per availability zone). In contrast, "Any traffic that remains inside the cluster, or that does not go out to the public Internet, will not pass through the NAT gateway and will have a source IP address belonging to the node that the traffic originated from. Node IP addresses are dynamic; therefore, a customer must not rely on allowlisting individual IP addresses when accessing private resources" ([ROSA classic service definition](https://docs.redhat.com/en/documentation/red_hat_openshift_service_on_aws_classic_architecture/4/html/introduction_to_rosa/policies-and-service-definition)).
- The ROSA HCP service definition does not repeat that paragraph (not confirmed for HCP in the service definition). The HCP egress tutorial says "By default, Red Hat OpenShift Service on AWS uses the OVN-Kubernetes container network interface (CNI) to assign random IP addresses from a pool" ([ROSA HCP egress IP tutorial](https://docs.redhat.com/en/documentation/red_hat_openshift_service_on_aws/4/html/tutorials/cloud-experts-consistent-egress-ip)).
- OpenShift OVN-Kubernetes: "Any pods communicate out to the wider network using source network address translation (SNAT). Their IP address is translated into the IP address of the node that the pod is running on" ([OCP 4.16 OVN-Kubernetes network plugin](https://docs.redhat.com/en/documentation/openshift_container_platform/4.16/html/networking/ovn-kubernetes-network-plugin)).

### EgressIP on ROSA

- ROSA HCP documents EgressIP in the tutorial "Assign a consistent egress IP for external traffic" ([ROSA HCP egress IP tutorial](https://docs.redhat.com/en/documentation/red_hat_openshift_service_on_aws/4/html/tutorials/cloud-experts-consistent-egress-ip)):
  - The prerequisite is a cluster "deployed with OVN-Kubernetes".
  - You label a machine pool with `rosa update machinepool ... --labels "k8s.ovn.org/egress-assignable="`, which replaces existing machine pool labels.
  - You check per-node capacity in the `cloud.network.openshift.io/egress-ipconfig` annotation (example value 14).
  - Egress IPs "should exist as a part of the subnets in which the worker nodes are provisioned", and you reserve them with VPC subnet CIDR reservations.
  - `EgressIP` objects can select a namespace, or pods by `podSelector`. `namespaceSelector` is mandatory.
- ROSA classic has a full chapter, "Configuring an egress IP address" ([ROSA classic OVN-Kubernetes egress IP](https://docs.redhat.com/en/documentation/red_hat_openshift_service_on_aws_classic_architecture/4/html/ovn-kubernetes_network_plugin/configuring-egress-ips-ovn)). It says:
  - egress IPs give "a consistent source IP address for services outside the cluster network"
  - assigning them to control plane nodes "is not supported"
  - AWS limits addresses by instance type ("IP addresses per network interface per instance type")
  - "In ROSA with HCP clusters, application pods and ingress router pods run on the same node. If you configure an egress IP address for an application project in this scenario, the IP address is not used when you send a request to a route from the application project."
- Inference (no document states this): with no EgressIP, an on-prem firewall sees worker node private IPs. Because node IPs are dynamic, the options the docs leave open are allowlisting the worker subnets or configuring EgressIP.

## 6. Windows nodes and gMSA

### Windows nodes on ROSA

- ROSA HCP: "Red Hat OpenShift support for Windows containers is not available on Red Hat OpenShift Service on AWS. Alternatively, it is supported to run Windows-based virtual machines on OpenShift Virtualization running on a Red Hat OpenShift Service on AWS cluster" ([ROSA HCP service definition](https://docs.redhat.com/en/documentation/red_hat_openshift_service_on_aws/4/html/introduction_to_rosa/policies-and-service-definition)).
- ROSA classic: "Red Hat OpenShift support for Windows containers is not available on Red Hat OpenShift Service on AWS classic architecture" ([ROSA classic service definition](https://docs.redhat.com/en/documentation/red_hat_openshift_service_on_aws_classic_architecture/4/html/introduction_to_rosa/policies-and-service-definition)).
- On self-managed OpenShift 4.21, the Windows Machine Config Operator (WMCO) supports AWS, Azure, vSphere, Google Cloud, Nutanix and bare metal ([OCP 4.21 Windows Container Support](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html-single/windows_container_support_for_openshift/)):
  - "Hybrid networking with OVN-Kubernetes is the only supported networking configuration."
  - "You must specify the network configuration when you install the cluster."
  - Section 2.4, "Windows Machine Config Operator known limitations", lists "Hosted Control Planes".
  - That page does not mention ROSA, OpenShift Dedicated or gMSA.

### Kubernetes gMSA is Windows-only

- The Kubernetes task page is titled "Configure GMSA for Windows Pods and containers". It covers "Pods and containers that will run on Windows nodes", states "The cluster is expected to have Windows worker nodes", and installs the GMSACredentialSpec CRD and webhooks from `kubernetes-sigs/windows-gmsa` ([Kubernetes gMSA task](https://kubernetes.io/docs/tasks/configure-pod-container/configure-gmsa/)).
- In the Pod API, `securityContext.windowsOptions` (which holds `gmsaCredentialSpecName` and `gmsaCredentialSpec`) carries the note "this field cannot be set when spec.os.name is linux" ([kubernetes/api core/v1 types.go](https://github.com/kubernetes/api/blob/master/core/v1/types.go)).
- "While you can only run the control plane on Linux, you can deploy worker nodes running either Windows or Linux" ([Windows containers in Kubernetes](https://kubernetes.io/docs/concepts/windows/intro/)).
- AWS's EKS gMSA guidance is also for Windows pods only ([EKS best practices: Configure gMSA for Windows Pods and containers](https://docs.aws.amazon.com/eks/latest/best-practices/windows-gmsa.html)).

## 7. gMSA for Linux containers (credentials-fetcher)

### What credentials-fetcher is

- "credentials-fetcher is a Linux daemon that retrieves gMSA credentials from Active Directory over LDAP. It creates and refreshes kerberos tickets from gMSA credentials" ([aws/credentials-fetcher README](https://github.com/aws/credentials-fetcher)).
- The README lists:
  - supported platforms: "Amazon Linux 2023 (recommended), Fedora 41+"
  - modes: managed ECS/Fargate and "standalone"
  - Kerberos cache directory `/var/credentials-fetcher/krbdir`
  - gRPC socket directory `/var/credentials-fetcher/socket`
- The latest release is v2.0.3, published 2026-06-12 ([GitHub releases](https://github.com/aws/credentials-fetcher/releases)).

### Where AWS supports it

- Amazon ECS on EC2: "A Linux container that runs with gMSA relies on the credentials-fetcher daemon that runs on the container's host Amazon EC2 instance" ([ECS: Using gMSA for EC2 Linux containers](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/linux-gmsa.html)).
  - Hosts can be domain-joined or use "domainless gMSA", with AD user credentials kept in Secrets Manager.
  - Tasks reference a CredSpec through `credentialSpecs` with a `credentialspec:` or `credentialspecdomainless:` prefix.
  - The ECS doc says the daemon "is only available for Amazon Linux 2023 and Fedora 37 and later". The README says Fedora 41+.
- Amazon ECS on Fargate: Linux gMSA requires platform version 1.4 or later and "You must use domainless gMSA" ([ECS: Using gMSA for Linux containers on Fargate](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/fargate-linux-gmsa.html)).

### EKS, Kubernetes and OpenShift

- I found no AWS or Red Hat document that supports credentials-fetcher or Linux gMSA for pods on EKS, Kubernetes or OpenShift.
- The AWS Open Source Blog says only that credentials-fetcher "is positioned to be extended into services such as Amazon Elastic Container Service (ECS) and Amazon Elastic Kubernetes Service (EKS)" ([AWS Open Source Blog](https://aws.amazon.com/blogs/opensource/aws-now-supports-credentials-fetcher-for-gmsa-on-amazon-linux-2023/)).
- Upstream pull request #53, "Kube support for gMSA", was closed without merging ([PR #53](https://github.com/aws/credentials-fetcher/pull/53)). Community discussion #38 has users asking about AKS and EKS pods, with no official procedure ([discussion #38](https://github.com/aws/credentials-fetcher/discussions/38)).
- On ROSA, all nodes run Red Hat CoreOS, and managed customers "may not manage any objects" in `machineconfiguration.openshift.io` ([ROSA classic service definition](https://docs.redhat.com/en/documentation/red_hat_openshift_service_on_aws_classic_architecture/4/html/introduction_to_rosa/policies-and-service-definition)). RHCOS is also not on the daemon's supported platform list. Installing the host daemon on ROSA nodes is therefore not a documented path.
- Conclusion from the sources: Linux gMSA with credentials-fetcher is an Amazon ECS feature (EC2 and Fargate launch types). Whether the README's standalone mode could run inside a Kubernetes pod is not covered by any official doc (not confirmed).

## 8. Secrets: CSI driver, External Secrets Operator and binary keytabs

### Secrets Store CSI driver with AWS Secrets Manager on ROSA

- Both ROSA doc sets have this tutorial: "Chapter 10. Tutorial: Use AWS Secrets Manager CSI on Red Hat OpenShift Service on AWS with STS" for HCP, and "Chapter 9. Tutorial: Use AWS Secrets Manager CSI on Red Hat OpenShift Service on AWS classic architecture with STS" for classic ([ROSA HCP tutorial](https://docs.redhat.com/en/documentation/red_hat_openshift_service_on_aws/4/html/tutorials/cloud-experts-aws-secret-manager), [ROSA classic tutorial](https://docs.redhat.com/en/documentation/red_hat_openshift_service_on_aws_classic_architecture/4/html/tutorials/cloud-experts-aws-secret-manager)). The tutorial:
  - installs the upstream driver with Helm from `kubernetes-sigs.github.io/secrets-store-csi-driver/charts`
  - installs the AWS provider from a `rh-mobb` manifest
  - grants the `privileged` SCC to the `secrets-store-csi-driver` and `csi-secrets-store-provider-aws` service accounts
  - labels the CSIDriver `security.openshift.io/csi-ephemeral-volume-profile=restricted`
  - uses an IAM role trusted through the cluster OIDC provider, with the service account annotated `eks.amazonaws.com/role-arn`
  - mounts a `SecretProviderClass` with `objectType: "secretsmanager"` at `/mnt/secrets-store`
- On self-managed OpenShift, the Secrets Store CSI Driver Operator became GA in 4.18 after Technology Preview in 4.14 ([OCP 4.18 release notes](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/release_notes/ocp-4-18-release-notes)). Its providers include AWS Secrets Manager, and "Red Hat does not test all factors associated with third-party secrets store provider functionality" ([OCP 4.21 Storage, CSI](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/storage/using-container-storage-interface-csi)).
- The OpenShift GitOps guide states: "Using the SSCSI Driver Operator with AWS Secrets Manager is not supported in a hosted control plane cluster" ([GitOps 1.17 Security](https://docs.redhat.com/en/documentation/red_hat_openshift_gitops/1.17/html/security/managing-secrets-securely-using-sscsid-with-gitops)).
- Whether the Operator (as opposed to the Helm method in the ROSA tutorial) is supported on ROSA: not confirmed.

### External Secrets Operator on OpenShift

- "External Secrets Operator for Red Hat OpenShift" release history ([OCP 4.21 Security and compliance](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/security_and_compliance/external-secrets-operator-for-red-hat-openshift)):
  - 0.1.0 (Technology Preview), issued 2025-06-26
  - 1.0.0 "(General Availability)", issued 2025-11-03 and based on upstream v0.19.0
  - 1.0.1, issued 16 July 2026
- Listed providers include AWS Secrets Manager ([OCP 4.19 Security and compliance](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/security_and_compliance/external-secrets-operator-for-red-hat-openshift)).
- ROSA product docs: no ESO page found (not confirmed). Red Hat's Cloud Experts site has a guide, "Using AWS Secrets Manager with External Secrets Operator on ROSA HCP", which assumes cluster-admin. The guide is field content and is not part of the product documentation ([Cloud Experts ESO guide](https://cloud.redhat.com/experts/rosa/eso/)).

### Can a binary keytab be stored and mounted as a file

- Kubernetes Secret: "The values for all keys in the `data` field have to be base64-encoded strings", so you can store binary bytes. "Individual Secrets are limited to 1MiB in size." For mounted Secrets "the kubelet stores a copy of the data into a `tmpfs`" ([Kubernetes Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)). Secrets "are, by default, stored unencrypted in the API server's underlying data store (etcd)" (same page).
- AWS Secrets Manager `SecretBinary` is a "Base64-encoded binary data object" of up to 65536 bytes, and "This parameter is not available in the Secrets Manager console" ([CreateSecret API](https://docs.aws.amazon.com/secretsmanager/latest/apireference/API_CreateSecret.html)).
- The AWS CSI provider writes `SecretBinary` bytes to the mounted file when `SecretString` is empty ([provider source](https://github.com/aws/secrets-store-csi-driver-provider-aws/blob/main/provider/secrets_manager_provider.go)). Mounted files default to permission `0644` unless `filePermission` is set ([provider README](https://github.com/aws/secrets-store-csi-driver-provider-aws)).
- ESO handling of binary keytab bytes (decoding options): not confirmed.

## 9. Time sync on ROSA nodes

- Kerberos tolerance: MIT `clockskew` "default value is 300 seconds, or five minutes" ([MIT krb5.conf](https://web.mit.edu/kerberos/krb5-latest/doc/admin/conf_files/krb5_conf.html)).
- Amazon Time Sync Service is reachable from any EC2 instance at link-local `169.254.169.123` (IPv4) or `fd00:ec2::123` (Nitro, IPv6), which "restricts this traffic to within your VPC". The recommended chrony line is `server 169.254.169.123 prefer iburst minpoll 4 maxpoll 4` ([EC2 local Amazon Time Sync Service](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configure-ec2-ntp.html)). The service smears leap seconds ([EC2 time sync overview](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/set-time.html)).
- RHCOS on AWS uses this by default, according to the source:
  - CoreOS's `coreos-platform-chrony-config` writes `server 169.254.169.123 prefer iburst minpoll 4 maxpoll 4` on the `aws | ec2` platform and sets `PEERNTP=no`, unless `/etc/chrony.conf` was modified ([fedora-coreos-config script](https://github.com/coreos/fedora-coreos-config/blob/testing-devel/overlay.d/20platform-chrony/usr/libexec/coreos-platform-chrony-config)).
  - The RHEL-based CoreOS manifest includes `overlay.d/20platform-chrony` ([shared-el.yaml](https://github.com/coreos/fedora-coreos-config/blob/testing-devel/manifests/shared-el.yaml)).
  - ROSA docs do not state the node NTP source (not confirmed in ROSA docs). The ROSA HCP service definition has no NTP statement ([ROSA HCP service definition](https://docs.redhat.com/en/documentation/red_hat_openshift_service_on_aws/4/html/introduction_to_rosa/policies-and-service-definition)).
- ROSA customers may not manage `machineconfiguration.openshift.io` objects ([ROSA classic service definition](https://docs.redhat.com/en/documentation/red_hat_openshift_service_on_aws_classic_architecture/4/html/introduction_to_rosa/policies-and-service-definition)), so a custom chrony MachineConfig is not a customer option on ROSA classic. The HCP equivalent: not confirmed.
- Pods use the node clock: Linux "time namespaces do not virtualize the CLOCK_REALTIME clock" ([time_namespaces(7)](https://man7.org/linux/man-pages/man7/time_namespaces.7.html)).
- The AD domain controllers' own time source is outside ROSA. Skew between the AD domain controllers and Amazon time is not covered by any doc found (not confirmed).

## 10. Connectivity to on-prem and AWS Managed Microsoft AD

### Private connectivity

- Both ROSA service definitions: ROSA "allows for the configuration of a private network connection through AWS-managed technologies", listing VPN, VPC peering, Transit Gateway and Direct Connect. "Red Hat site reliability engineers (SREs) do not monitor private network connections" ([ROSA HCP service definition](https://docs.redhat.com/en/documentation/red_hat_openshift_service_on_aws/4/html/introduction_to_rosa/policies-and-service-definition), [ROSA classic service definition](https://docs.redhat.com/en/documentation/red_hat_openshift_service_on_aws_classic_architecture/4/html/introduction_to_rosa/policies-and-service-definition)).
- ROSA procedures: "Configuring private connections" covers VPC peering, VPN and Direct Connect ([ROSA HCP](https://docs.redhat.com/en/documentation/red_hat_openshift_service_on_aws/4/html/cluster_administration/configuring-private-connections), [ROSA classic](https://docs.redhat.com/en/documentation/red_hat_openshift_service_on_aws_classic_architecture/4/html/cluster_administration/configuring-private-connections)).
- AWS responsibility matrix: the customer will "Set up and maintain optional Amazon VPC components, such as Amazon VPC-to-VPC connection, Site-to-Site VPN connection, or Direct Connect" ([AWS ROSA responsibilities](https://docs.aws.amazon.com/rosa/latest/userguide/rosa-responsibilities.html)).
- Transit Gateway is "a network transit hub used to interconnect virtual private clouds (VPCs) and on-premises networks". Attachments include VPCs, VPN connections and Direct Connect gateways ([What is AWS Transit Gateway](https://docs.aws.amazon.com/vpc/latest/tgw/what-is-transit-gateway.html)).
- Direct Connect "links your internal network to a Direct Connect location over a standard Ethernet fiber-optic cable". Private and transit virtual interfaces reach VPCs and transit gateways ([What is Direct Connect](https://docs.aws.amazon.com/directconnect/latest/UserGuide/Welcome.html)).
- Site-to-Site VPN supports IPsec, and "Each VPN connection includes two VPN tunnels" ([What is AWS Site-to-Site VPN](https://docs.aws.amazon.com/vpn/latest/s2svpn/VPC_VPN.html)).

### AWS Managed Microsoft AD with a trust to on-prem AD

- AWS Managed Microsoft AD "creates a highly available pair of domain controllers in your Amazon VPC across different Availability Zones", and supports trust relationships with self-managed AD ([AWS Managed Microsoft AD](https://docs.aws.amazon.com/directoryservice/latest/admin-guide/directory_microsoft_ad.html)).
- It supports "one and two-way external and forest trust relationships" with self-managed directories, in incoming, outgoing and two-way directions ([Creating a trust relationship](https://docs.aws.amazon.com/directoryservice/latest/admin-guide/ms_ad_setup_trust.html)). Prerequisites from the same page:
  - network connection from the on-prem network to the VPC, with the Microsoft-listed AD and trust ports open
  - Kerberos pre-authentication enabled on user accounts
  - DNS conditional forwarders on the self-managed domain
  - single-label domains not supported
- The ECS gMSA docs accept either AWS Managed Microsoft AD or on-prem AD, with on-prem reachability via Direct Connect ([ECS Linux gMSA](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/linux-gmsa.html)).
- Whether a SQL Server login on on-prem SQL Server would accept a principal from an AWS Managed AD domain across a trust is outside these docs (not confirmed here).

## Beginner reading list

I opened every URL below on 2026-11-23 and checked that it loads with the title shown.

1. Pods: https://kubernetes.io/docs/concepts/workloads/pods/. Page title "Pods". It opens with "Pods are the smallest deployable units of computing that you can create and manage in Kubernetes."
2. Sidecar containers: https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/. Page title "Sidecar Containers". It explains native sidecars and shared volumes.
3. Secrets: https://kubernetes.io/docs/concepts/configuration/secret/. Page title "Secrets". It covers base64 `data`, size limits, tmpfs mounts and etcd encryption caveats.
4. Security context constraints: https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/authentication_and_authorization/managing-pod-security-policies. Page title "Chapter 16. Managing security context constraints". It covers restricted-v2 and UID ranges.
5. Arbitrary UIDs in images: https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/images/creating-images. Page title "Chapter 4. Creating images". See section 4.1.2.2, "Support arbitrary user ids".
6. DNS Operator on ROSA: https://docs.redhat.com/en/documentation/red_hat_openshift_service_on_aws/4/html/networking_operators/dns-operator. Page title "Chapter 2. DNS Operator in Red Hat OpenShift Service on AWS". It covers zone forwarding.
7. ROSA networking basics: https://docs.redhat.com/en/documentation/red_hat_openshift_service_on_aws/4/html/networking_overview/about-managed-networking. Page title "Chapter 1. About networking for managed clusters". It is part of the ROSA Networking overview.
8. EgressIP on ROSA: https://docs.redhat.com/en/documentation/red_hat_openshift_service_on_aws/4/html/tutorials/cloud-experts-consistent-egress-ip. Page title "Chapter 12. Tutorial: Assign a consistent egress IP for external traffic".
9. Secrets Store CSI on ROSA: https://docs.redhat.com/en/documentation/red_hat_openshift_service_on_aws/4/html/tutorials/cloud-experts-aws-secret-manager. Page title "Chapter 10. Tutorial: Use AWS Secrets Manager CSI on Red Hat OpenShift Service on AWS with STS".
10. What ROSA customers can and cannot change: https://docs.redhat.com/en/documentation/red_hat_openshift_service_on_aws/4/html/introduction_to_rosa/policies-and-service-definition. Page title "Chapter 5. Policies and service definition". It covers Windows support, DNS forwarding, private connections and dedicated-admin.
