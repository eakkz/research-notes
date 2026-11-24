# AWS networking and DNS for this design

Kerberos needs two things from the network: the pod must resolve AD names, and it must reach the domain controllers on port 88. This page explains the AWS pieces involved, enough to talk to the network team.

## VPC, subnets and security groups

A VPC is a private network in AWS. It is split into subnets, each in one availability zone. ROSA HCP worker nodes live in subnets of the ROSA VPC, and SQL Server on EC2 lives in a subnet of the same VPC or another one.

Two layers of firewall apply inside AWS:

- Security groups are attached to instances (or network interfaces) and say which traffic may come in or go out. The SQL Server instance's security group must allow 1433 from the worker subnets. If the domain controllers are in AWS, theirs must allow 88 and 53.
- Network ACLs are attached to subnets. They are often left open, but check them when a security group looks correct and traffic still fails.

Traffic between AWS and the data center may also cross corporate firewalls, which the AWS console does not show.

## Getting from one network to another

| Service | What it connects | When it is used here |
| --- | --- | --- |
| VPC peering | Two VPCs, directly | ROSA VPC to a SQL Server VPC or a managed AD VPC |
| Transit Gateway | Many VPCs and on-prem links through one hub ([AWS: Transit Gateway](https://docs.aws.amazon.com/vpc/latest/tgw/what-is-transit-gateway.html)) | The usual enterprise hub; routes ROSA traffic to on-prem |
| Site-to-Site VPN | An encrypted tunnel over the internet to the data center ([AWS: Site-to-Site VPN](https://docs.aws.amazon.com/vpn/latest/s2svpn/VPC_VPN.html)) | Variant A, if there is no Direct Connect |
| Direct Connect | A dedicated private line to the data center ([AWS: Direct Connect](https://docs.aws.amazon.com/directconnect/latest/UserGuide/Welcome.html)) | Variant A, typical in large companies |

ROSA supports all of these, and the customer owns them. Red Hat SREs do not monitor private connections ([ROSA HCP service definition](https://docs.redhat.com/en/documentation/red_hat_openshift_service_on_aws/4/html/introduction_to_rosa/policies-and-service-definition)).

## DNS inside a VPC

Each VPC has a built-in resolver (AmazonProvidedDNS) at the VPC's base address plus two. EC2 instances and ROSA nodes use it by default. It answers public names, AWS names and any Route 53 private hosted zones attached to the VPC. It does not know about `corp.example.com` unless you tell it.

Route 53 Resolver is how you tell it ([AWS: Forwarding outbound DNS queries](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resolver-forwarding-outbound-queries.html)):

- An outbound endpoint is a set of network interfaces in your VPC that can send DNS queries to servers outside AWS DNS.
- A forwarding rule says "for `corp.example.com`, ask these IP addresses" (the AD DNS servers), through that endpoint.
- The rule is associated with one or more VPCs. Rules can be shared across AWS accounts with AWS RAM, so a central network account often owns them already.

Once the rule is associated with the ROSA VPC, pods resolve AD names with no change in OpenShift. That is why this pack prefers it.

The reverse direction, on-prem servers resolving names in AWS, uses an inbound endpoint. This design does not need it.

## AWS Managed Microsoft AD

AWS Managed Microsoft AD (part of AWS Directory Service) runs a real Microsoft AD domain for you: two domain controllers in your VPC, in different availability zones. AWS runs the servers; you get an OU where your admins can create users, groups and SPNs ([AWS: Managed Microsoft AD](https://docs.aws.amazon.com/directoryservice/latest/admin-guide/directory_microsoft_ad.html)).

It can trust a corporate AD forest, one-way or two-way. That needs network connectivity, the AD ports open both ways, and DNS conditional forwarders on each side ([AWS: Creating a trust relationship](https://docs.aws.amazon.com/directoryservice/latest/admin-guide/ms_ad_setup_trust.html)).

In variant B, SQL Server on EC2 joins the managed domain and the app's service account is created in it. The trust is only for people who log in with corporate accounts. The app's Kerberos traffic stays inside AWS.

## The two variants side by side

| | Variant A: corporate AD | Variant B: AWS Managed AD |
| --- | --- | --- |
| Domain controllers | Data center, or corporate DCs in AWS | Two DCs in an AWS VPC |
| Path for port 88 | ROSA VPC, Transit Gateway, Direct Connect or VPN, corporate firewall | ROSA VPC, peering or Transit Gateway |
| DNS rule target | Corporate DNS servers | The managed AD's DNS addresses |
| Who creates the service account and keytab | Corporate AD team | Your team, in the delegated OU, or whoever owns the managed directory |
| Effect of a Direct Connect outage | New logins fail | No effect on the app |
| Extra work | Firewall change requests across teams | Running the directory, and a trust if people need corporate logins |

## Ports, again

| Port | Protocol | To | Why |
| --- | --- | --- | --- |
| 88 | TCP and UDP | Domain controllers | Kerberos tickets |
| 53 | TCP and UDP | AD DNS, or the Resolver endpoint | Name lookups, including SRV records |
| 1433 | TCP | SQL Server | The database connection |

Large DNS answers (domains with many domain controllers) fall back from UDP to TCP on port 53, so allow both.

## Where to go next

- [10-reading-list.md](10-reading-list.md).
