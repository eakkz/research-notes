# OpenShift and ROSA HCP

OpenShift is Red Hat's Kubernetes distribution. Everything in [06-kubernetes-openshift-and-pods.md](06-kubernetes-openshift-and-pods.md) and [07-sidecars.md](07-sidecars.md) applies, plus some stricter security defaults. ROSA is OpenShift run as a managed service on AWS, with rules about what customers may change. This page covers the parts that affect Kerberos.

## ROSA HCP in one paragraph

Red Hat OpenShift Service on AWS (ROSA) runs OpenShift clusters in your AWS account, operated by Red Hat SREs. With hosted control planes (HCP), the control plane (API server, etcd) runs in a Red Hat-managed account, and only the worker nodes run in your VPC. For this project that means: pods run on worker nodes in your VPC subnets, so their network traffic leaves from those subnets. The cluster's own management components are out of your hands, and some cluster settings are restricted. Red Hat's service definition lists what you may and may not change ([ROSA HCP: Policies and service definition](https://docs.redhat.com/en/documentation/red_hat_openshift_service_on_aws/4/html/introduction_to_rosa/policies-and-service-definition)).

## Random user IDs and the restricted-v2 SCC

A Security Context Constraint (SCC) is OpenShift's policy for what a pod may do: which user it runs as, which volumes it may mount, which Linux capabilities it gets. New pods get `restricted-v2` by default ([OpenShift: Managing SCCs](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/authentication_and_authorization/managing-pod-security-policies)). It:

- runs containers as a high user ID taken from the project's range (the first value of the range unless the pod asks for another), not as the user in the image,
- drops all Linux capabilities and blocks privilege escalation,
- applies the default seccomp profile, which filters system calls,
- allows `emptyDir`, Secret, ConfigMap and CSI volumes, everything this design needs.

What that means for Kerberos:

- Every container in the pod that does not set `runAsUser` gets the same UID, the first in the project's range. The sidecar and the app can therefore share the 0600 ticket file. Do not give one of them a different UID.
- The assigned UID has no name in the image's `/etc/passwd`. CRI-O, the container runtime, adds an entry for it when it can. Kerberos tools work by UID and do not need a name. Java's `user.name` falls back to `?` without an entry; Java apps should set the principal and `KRB5CCNAME` explicitly.
- Files your image writes to at runtime must be group-writable by group 0, because the assigned user is always in the root group ([OpenShift: Creating images, arbitrary user IDs](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/images/creating-images)). Mounted volumes are not affected.
- The default seccomp profile does not allow the system calls that kernel keyrings need. Together with keyrings not being container-aware, that is another reason to use a `FILE:` ticket cache.

Nothing in this design needs a privileged SCC.

## Windows containers are not available

Self-managed OpenShift can run Windows worker nodes. ROSA cannot: "Red Hat OpenShift support for Windows containers is not available on Red Hat OpenShift Service on AWS." ROSA HCP does support Windows virtual machines through OpenShift Virtualization ([ROSA HCP service definition](https://docs.redhat.com/en/documentation/red_hat_openshift_service_on_aws/4/html/introduction_to_rosa/policies-and-service-definition)). Kubernetes gMSA support needs Windows nodes, so gMSA is not an option on ROSA.

## Cluster DNS and DNS forwarding

Every pod resolves names through the cluster DNS service, managed by the DNS Operator. Names it does not own (anything outside the cluster) are passed to the resolver the nodes use, which on AWS is the VPC resolver.

You can point the cluster at the AD DNS servers in two ways:

- At the VPC level: a Route 53 Resolver rule sends `corp.example.com` queries to AD DNS. The cluster is not touched. See [09-aws-networking-and-dns.md](09-aws-networking-and-dns.md).
- At the cluster level: the DNS Operator can forward a named zone to specific servers by editing `dns.operator/default` ([ROSA HCP: DNS Operator](https://docs.redhat.com/en/documentation/red_hat_openshift_service_on_aws/4/html/networking_operators/dns-operator)). ROSA's service definition allows internal DNS servers for named domains, and Red Hat's managed-cluster configuration grants the `dedicated-admin` role patch rights on this object for HCP clusters too. Confirm it on your cluster before relying on it.

This pack defaults to the VPC level, with the DNS Operator as the fallback.

## Where pod traffic comes from: egress and EgressIP

When a pod connects to something outside the cluster, OpenShift's network plugin (OVN-Kubernetes) rewrites the source address to the IP of the node the pod runs on (SNAT). The domain controllers and SQL Server see the worker node's IP, not the pod's. Nodes come and go, so their IPs change; Red Hat warns not to allowlist individual node IPs ([ROSA classic service definition](https://docs.redhat.com/en/documentation/red_hat_openshift_service_on_aws_classic_architecture/4/html/introduction_to_rosa/policies-and-service-definition)).

Two ways to handle firewalls:

- Allow the whole worker subnet CIDRs. Simple, and enough for the PoC.
- Use EgressIP: reserve addresses in the worker subnets, label a machine pool as egress-capable, and create an `EgressIP` object for your project. Traffic from that project then leaves from those fixed addresses ([ROSA HCP: Consistent egress IP tutorial](https://docs.redhat.com/en/documentation/red_hat_openshift_service_on_aws/4/html/tutorials/cloud-experts-consistent-egress-ip)).

## Storing the keytab

| Option | How it works | Notes |
| --- | --- | --- |
| OpenShift Secret | The keytab is stored (base64) in the cluster and mounted as a file | Simplest. Binary is fine; size limit 1 MiB. Stored in etcd, so access control on the project matters ([Kubernetes: Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)) |
| AWS Secrets Manager + Secrets Store CSI driver | The pod mounts a CSI volume; the AWS provider fetches the secret at mount time using an IAM role | ROSA HCP has a tutorial using Helm. The provider's service accounts need the `privileged` SCC. Store the keytab as `SecretBinary` (64 KiB max), which is not available in the console ([ROSA HCP: AWS Secrets Manager CSI tutorial](https://docs.redhat.com/en/documentation/red_hat_openshift_service_on_aws/4/html/tutorials/cloud-experts-aws-secret-manager)) |
| External Secrets Operator | An operator copies the secret from AWS Secrets Manager (or Vault) into an OpenShift Secret and keeps it in sync | GA on OpenShift since late 2025; ROSA product docs do not cover it yet, Red Hat's Cloud Experts site does |

Red Hat's GitOps documentation says the Secrets Store CSI Driver Operator with AWS Secrets Manager is not supported on hosted control planes. The Helm method in the ROSA tutorial is a different install path. Check support status with Red Hat before choosing it for production.

## Time

Pods use the node's clock. ROSA nodes run Red Hat CoreOS, whose default chrony configuration on AWS uses Amazon Time Sync Service; ROSA docs do not state the node time source. ROSA customers cannot change node time settings, so if the domain controllers drift, the fix is on the AD side.

## What you cannot do on ROSA

- Install software on the nodes (no node-level daemons such as `credentials-fetcher`, no `MachineConfig` changes).
- Run Windows containers.
- Join nodes or pods to AD.

Because of these limits, Kerberos has to live inside the pod.

## Where to go next

- [09-aws-networking-and-dns.md](09-aws-networking-and-dns.md).
