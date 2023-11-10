---
layout: post
title: "Connecting to AWS Redshift from RHEL9 with FIPS (TLS EMS)"
date: 2023-11-10 10:30
categories: Linux
tags: [rhel, fips, aws, rds]
---

I was recently tasked with moving a deployment from RHEL8 to RHEL9. As part of this deployment, I needed to execute some SQL statements against an AWS Redshift instance. On RHEL8, this was accomplished by using the `psql` CLI tool. Both RHEL8 and RHEL9 were required to be running with FIPS mode. However, I ran into the following error:

```console
(python-venv) [root@ip-10-6-35-223 ~]# psql -h ${DBHOST} -p 5439 -d "${DBNAME}" -U "${DBUSER}"
psql: error: SSL error: ems not enabled
FATAL:  no pg_hba.conf entry for host "::ffff:10.6.35.223", user "devuser", database "dev", SSL off


(python-venv) [root@ip-10-6-35-223 ~]# psql --version
psql (PostgreSQL) 13.12
```

Other background information:

* The Redshift instance was configured to force SSL and use FIPS SSL
* Connections from Java services and Python services using the SDK continued to work without issue on RHEL9
* The connection worked with RHEL8

When I started looking for additional information SSL and EMS, not a lot of useful results came back. I finally discovered that EMS was "extended master secret" and was able to find <https://www.ietf.org/rfc/rfc7627.html#page-6> as well as a few RedHat articles.


## The Problem

RHEL 9 when running in FIPS mode requires EMS for TLS 1.2 connections or TLS 1.3.


> A RHEL 9.2 and later system running in FIPS mode enforces that any TLS 1.2 connection must use the Extended Master Secret (EMS) extension (RFC 7627) as requires the FIPS 140-3 standard. Thus, legacy clients not supporting EMS or TLS 1.3 cannot connect to RHEL 9 servers running in FIPS mode, RHEL 9 clients in FIPS mode cannot connect to servers that support only TLS 1.2 without EMS. See [TLS Extension "Extended Master Secret" enforced with Red Hat Enterprise Linux 9.2](https://access.redhat.com/solutions/7018256)
{: .prompt-danger }

At the same time, Amazon Redshift only supports TLS 1.2 but does not support extended master secret (EMS).

```console
openssl s_client -starttls postgres -connect ${DBHOST}:5439

---
No client certificate CA names sent
Peer signing digest: SHA512
Peer signature type: RSA
Server Temp Key: ECDH, prime256v1, 256 bits
---
SSL handshake has read 5051 bytes and written 449 bytes
Verification: OK
---
New, (NONE), Cipher is (NONE)
Server public key is 2048 bit
Secure Renegotiation IS supported
Compression: NONE
Expansion: NONE
No ALPN negotiated
SSL-Session:
    Protocol  : TLSv1.2
    Cipher    : 0000
    Session-ID: 
    Session-ID-ctx: 
    Master-Key: 
    PSK identity: None
    PSK identity hint: None
    SRP username: None
    Start Time: 1698680420
    Timeout   : 7200 (sec)
    Verify return code: 0 (ok)
    Extended master secret: no
---
```


## The Workaround

I'm not calling this a fix as it technically breaks FIPS compliance on RHEL9. It is also system wide rather than specific to the `psql` command I'm trying to execute. Ideally, I could set this as an environment variable and only impact the single `psql` command. However, in my case, I'm only executing a few commands to initialize the database - the long running Python and Java services continue to function.

Edit the file `/etc/pki/tls/fips_local.cnf` to set `tls1-prf-ems-check` config option from `1` to `0`.

```ini
[fips_sect]
tls1-prf-ems-check = 0
activate = 1 
```

When finished, set the `0` back to `1`.