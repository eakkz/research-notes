# Which AD domain and how AWS reaches it

Label: wayfinder:grilling. Status: closed. Blocked by: none

## Question

Corporate on-prem AD over VPN/Direct Connect, AWS Managed Microsoft AD with a trust, or unknown?

## Resolution

Not known. Cover both: corporate AD reached over Direct Connect/VPN, and AWS Managed Microsoft AD with a trust.

Service account must live in the same domain as SQL Server (no cross-realm).
