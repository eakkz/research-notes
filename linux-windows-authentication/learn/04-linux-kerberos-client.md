# The Linux Kerberos client

Inside the container, Kerberos is handled by the MIT Kerberos library and a few command-line tools. This page covers the config file, the tools, where tickets are stored, and the settings that trip people up in containers.

## Packages

| Piece | RHEL / UBI package | Debian / Ubuntu package | Used by |
| --- | --- | --- | --- |
| Library (`libkrb5`, `libgssapi_krb5`) | `krb5-libs` | `libkrb5-3`, `libgssapi-krb5-2` | ODBC driver, every Kerberos-aware program |
| Tools (`kinit`, `klist`, `kdestroy`, `kvno`) | `krb5-workstation` | `krb5-user` | The sidecar, and you while debugging |

UBI 9 images include the library but not the tools, so the sidecar image needs `krb5-workstation`. Check your own base image with `rpm -q krb5-libs krb5-workstation`.

## krb5.conf

`krb5.conf` tells the library which realm to use and how to find the KDCs. It is read from `/etc/krb5.conf`, or from the path in `KRB5_CONFIG`. A version for this project:

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

Line by line, with definitions from [MIT's krb5.conf reference](https://web.mit.edu/kerberos/krb5-latest/doc/admin/conf_files/krb5_conf.html):

- `default_realm`: the realm used when a principal has no `@REALM`. It is the AD domain name in upper case. Realm names are case sensitive.
- `dns_lookup_realm = false`: do not use DNS TXT records to guess which realm a host belongs to; use `[domain_realm]` instead.
- `dns_lookup_kdc = true`: find KDCs through DNS SRV records (`_kerberos._tcp.corp.example.com`). AD publishes these automatically. If the cluster cannot resolve them, set this to `false` and add a `[realms]` section listing `kdc = dc01.corp.example.com` lines.
- `rdns = false`: do not use reverse DNS to rewrite the server name before building the SPN. With reverse lookups, a PTR record that returns a different name makes the client ask for an SPN that does not exist. MIT recommends turning it off when forward and reverse DNS disagree ([MIT: Principal names and DNS](https://web.mit.edu/kerberos/krb5-latest/doc/admin/princ_dns.html)).
- `dns_canonicalize_hostname = false`: do not follow CNAME records to rewrite the server name either. The client then asks for exactly the name in the connection string, which must be the server's FQDN as registered in the SPN. Short names are not expanded either. With this set to `false`, `rdns` has no effect; keeping both lines is harmless and records the intent.
- `udp_preference_limit = 0`: try TCP first for every KDC request, with UDP only as a fallback. AD tickets carry group membership (the PAC) and are often too large for UDP. Microsoft's SQL Server on Linux tutorial suggests the same.
- `forwardable = true`: allows the ticket to be forwarded. Harmless here; Microsoft's sample `krb5.conf` for the Linux ODBC driver sets it too.
- `[domain_realm]`: maps DNS names to realms, so the library knows `sql01.corp.example.com` belongs to `CORP.EXAMPLE.COM`.

## The tools

| Command | What it does |
| --- | --- |
| `kinit -kt /path/svc-app.keytab svc-app@CORP.EXAMPLE.COM` | Gets a TGT using the keytab and writes it to the credential cache |
| `klist` | Shows the tickets in the cache, with their expiry times |
| `klist -s` | Silent check: exit code 0 if a valid ticket exists. Good for a startup probe |
| `klist -kte /path/svc-app.keytab` | Lists the entries in a keytab: principal, kvno, encryption type |
| `kvno MSSQLSvc/sql01.corp.example.com:1433` | Asks for a service ticket for that SPN. Proves the SPN exists without involving SQL Server |
| `kdestroy` | Deletes the tickets in the cache |

User-side documentation: [MIT Kerberos: For users](https://web.mit.edu/kerberos/krb5-latest/doc/user/index.html).

## Where tickets live: the credential cache

The credential cache (ccache) is where `kinit` writes tickets and where the library reads them. MIT supports several types ([MIT: credential cache](https://web.mit.edu/kerberos/krb5-latest/doc/basic/ccache_def.html)):

| Type | Stored in | Suits containers? |
| --- | --- | --- |
| `FILE:/path` | A regular file | Yes. Two containers can share it through a volume |
| `KEYRING:...` | The Linux kernel keyring | No. Not namespaced per container, and likely blocked by OpenShift's default seccomp profile |
| `KCM:` | A daemon (`sssd-kcm`) over a socket | No. Needs a daemon that UBI does not ship |

UBI 9 and RHEL ship `default_ccache_name = KEYRING:persistent:%{uid}` in `/etc/krb5.conf`. Override it. The library picks the cache in this order: the `KRB5CCNAME` environment variable first, then `default_ccache_name` in `krb5.conf`, then a built-in default. Setting the variable in every container is the simplest fix:

```text
KRB5CCNAME=FILE:/var/kerberos-ccache/cache
```

Java programs read only `KRB5CCNAME` (and only the `FILE:` type), never `default_ccache_name`, so the variable is the setting that works for every client.

MIT creates the cache file with mode 0600, readable only by its owner. Every container that shares it must run with the same user ID. OpenShift does this by default; see [08-openshift-and-rosa-hcp.md](08-openshift-and-rosa-hcp.md).

## Environment variables

From [MIT: Kerberos environment variables](https://web.mit.edu/kerberos/krb5-latest/doc/user/user_config/kerberos.html):

| Variable | Meaning |
| --- | --- |
| `KRB5CCNAME` | Which credential cache to use, as `TYPE:residual` |
| `KRB5_CONFIG` | Path to `krb5.conf` |
| `KRB5_KTNAME` | Default keytab (for services) |
| `KRB5_CLIENT_KTNAME` | Default client keytab. MIT can get tickets from it by itself when none exist. Untested with the ODBC driver |
| `KRB5_TRACE` | File to write a detailed trace to, for example `/dev/stderr`. The fastest way to debug |

## Where to go next

- [05-sql-server-and-odbc-driver.md](05-sql-server-and-odbc-driver.md) for how pyodbc uses the ticket.
- [MIT Kerberos: For administrators](https://web.mit.edu/kerberos/krb5-latest/doc/admin/index.html) for the full `krb5.conf` reference.
