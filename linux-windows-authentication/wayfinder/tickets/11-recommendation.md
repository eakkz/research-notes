# Recommended solution and PoC plan

Label: wayfinder:grilling. Status: closed. Blocked by: 04, 05, 06, 07, 09, 01, 02, 03

## Question

Given the answers and research, which option goes first, and what does a PoC have to prove?

## Resolution

Kerberos sidecar (option A) is the target; first PoC stage runs kinit in the app container (option B). NTLM via pymssql/FreeTDS is a not-recommended fallback; gMSA is not feasible on ROSA. Defaults: Route 53 Resolver rule for AD DNS (DNS Operator fallback), worker subnet CIDRs for the PoC firewall source (EgressIP for fixed IPs), OpenShift Secret for the PoC keytab. Details in 01-feasibility-and-options.md and 04-poc-test-plan.md.
