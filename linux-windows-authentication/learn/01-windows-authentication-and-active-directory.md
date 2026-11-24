# Windows Authentication and Active Directory

Read this first. It explains what "Windows Authentication" means to SQL Server and what Active Directory does, before any Kerberos detail.

## Two ways to log in to SQL Server

SQL Server accepts two kinds of login.

A SQL Server login has a name and a password stored inside SQL Server itself, for example `app_user` with password `...`. The app sends both in the connection string. SQL Server checks them against its own list. Nothing else is involved.

A Windows login belongs to an account in Active Directory, for example `CORP\svc-app`. SQL Server does not store a password for it. The client proves "I am `CORP\svc-app`" by some other means, and SQL Server trusts that proof because SQL Server and the account belong to the same AD domain, or to domains that trust each other. This is what people mean by Windows Authentication, Integrated Security or a trusted connection.

Companies prefer Windows logins because the password lives in one place (AD), accounts can be disabled centrally, group membership controls access, and password rules apply to every system at once.

## What Active Directory is

Active Directory (AD) is Microsoft's directory service. For this project, three of its jobs matter:

- It stores accounts: users, computers, service accounts and groups.
- It runs the Kerberos Key Distribution Center (KDC) on every domain controller. This is the service that hands out the tickets used to prove identity.
- It depends on DNS for the domain, usually served by the domain controllers themselves, so names such as `sql01.corp.example.com`, and the records that tell clients where the KDCs are, resolve correctly.

A domain is a group of accounts and computers managed together, named like a DNS domain (`corp.example.com`). A domain controller (DC) is a Windows server that holds a copy of the domain's data and answers login requests. A forest is one or more domains that trust each other. A trust between two forests lets accounts from one be used in the other.

Joining a computer to the domain gives it a computer account and makes it trust the domain. SQL Server on EC2 in this project is joined. The Linux containers are not, and do not need to be.

## How Windows does it without asking

On a domain-joined Windows server, an app connecting with `Trusted_Connection=yes` does almost nothing:

1. When the service account logged on, Windows already got a Kerberos ticket for it from a domain controller.
2. The SQL driver asks Windows, through an interface called SSPI, for proof of identity aimed at SQL Server.
3. Windows uses the stored ticket to get a ticket for SQL Server and passes it on.
4. If Kerberos fails, Windows falls back to the older NTLM method without telling anyone.

All of this works because Windows logged the account on, holds its secret, and renews tickets in the background.

## What a Linux container lacks

A Linux container is not domain-joined, nobody logs on to it, and on OpenShift it even runs under an unpredictable user ID taken from the project's range. Nothing in it holds a ticket, and nothing renews one. Linux also has no SSPI; programs use a standard Kerberos interface called GSSAPI instead, provided by the MIT Kerberos library.

So on Linux you do by hand what Windows does at logon:

- Give the container the account's secret in a file called a keytab.
- Run `kinit` to turn the keytab into a ticket.
- Run it again before the ticket expires.
- Make sure the container can reach and resolve the domain controllers.

The rest of the `learn/` folder explains each of those pieces.

## NTLM, the older method

NTLM is the protocol Windows used before Kerberos. The client proves it knows the password by answering a challenge from the server, and the server checks the answer with a domain controller. It needs no tickets and no SPN, which makes it look easier from Linux.

Microsoft deprecated every NTLM version in June 2024 ([Microsoft: Deprecated features](https://learn.microsoft.com/en-us/windows/whats-new/deprecated-features)) and plans to switch network NTLM off by default in the next major Windows release, with timelines that may change ([Windows IT Pro blog](https://techcommunity.microsoft.com/blog/windows-itpro-blog/advancing-windows-security-disabling-ntlm-by-default/4489526)). The Microsoft ODBC driver on Linux has no documented NTLM mode and does not fall back to NTLM when Kerberos fails. That is why this project uses Kerberos and keeps NTLM only as a fallback through a different driver.

## Where to go next

- [02-kerberos.md](02-kerberos.md) for how tickets work.
- [Microsoft: Kerberos authentication overview](https://learn.microsoft.com/en-us/windows-server/security/kerberos/kerberos-authentication-overview) for Microsoft's own short introduction.
