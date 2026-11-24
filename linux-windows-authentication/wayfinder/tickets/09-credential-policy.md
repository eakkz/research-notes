# Credential storage and rotation rules

Label: wayfinder:grilling. Status: closed. Blocked by: none

## Question

Is a keytab in a Kubernetes Secret acceptable? Is gMSA or a vault mandated? What rotates passwords?

## Resolution

Not known. Compare Kubernetes Secret, AWS Secrets Manager via Secrets Store CSI, HashiCorp Vault; list questions for the security and AD teams. NTLM goes in as a documented, not-recommended fallback.
