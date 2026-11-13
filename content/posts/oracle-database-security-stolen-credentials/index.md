---
title: "Attackers log in, they don't break in: Oracle database controls after the latest breaches"
date: 2026-11-13T07:00:00-03:00
draft: false
categories: ["Oracle DBA"]
tags: ["oracle", "dba", "security", "unified auditing", "least privilege", "tde", "19c"]
summary: "Recent breaches started with default passwords and leaked credentials. Five Oracle 19c controls that limit what a stolen credential can do."
description: "Recent breaches started with default passwords and leaked credentials. Five Oracle 19c controls that limit what a stolen credential can do."
cover:
  image: "cover.png"
  alt: "They log in, not break in: Oracle 19c security checks"
---

## What the last few weeks looked like

Two incidents disclosed in October and November 2026, both ending at a database:

- **Chainstack.** An internet-exposed Grafana in a development cluster still had the default administrator credentials. From there the attacker picked up machine credentials that reached production and used them to get into the database network. Entry on October 26, database network on October 29, detection on October 31. Customer names, emails and password hashes were exposed. ([The Block](https://www.theblock.co/news/ecosystems/2026-11-12-chainstack-security-breach-438509))
- **Applebee's.** A researcher found the password for an internal database inside the mobile app. ([Privacy Guides](https://discuss.privacyguides.net/t/data-breach-roundup-oct-30-nov-5-2026/42068))

Going by what was disclosed, neither needed a database vulnerability. Someone had a working credential. That is the part a DBA can act on, so this post sticks to five controls in Oracle Database 19c that reduce what a valid but stolen login can do, and how to check each one.

All examples use fictitious names (`APP_USER`, `APP_OWNER`, `APP_SCHEMA.CUSTOMERS`).

## 1. Find the accounts that should not be able to log in

Start with default passwords. The Security Guide gives the query:

```sql
SELECT d.username, u.account_status
FROM   dba_users_with_defpwd d
JOIN   dba_users u ON u.username = d.username
ORDER  BY 2, 1;
```

Anything listed and `OPEN` is a problem today.

Next, look at how each account authenticates and which profile it uses:

```sql
SELECT username, authentication_type, account_status, profile
FROM   dba_users
ORDER  BY username;
```

Schema owners rarely need to log in. If `APP_OWNER` only holds objects and the application connects as `APP_USER`, remove the password:

```sql
ALTER USER app_owner NO AUTHENTICATION;
```

`AUTHENTICATION_TYPE` becomes `NONE` and there is no password left to leak. Deployments that used to connect as the owner need another path (proxy authentication through a named user), so line that up before you run it.

For accounts that do log in, check the profile. `INACTIVE_ACCOUNT_TIME` is `UNLIMITED` in the default profile, which means a forgotten account stays usable forever.

```sql
SELECT profile, resource_name, limit
FROM   dba_profiles
WHERE  resource_name IN ('INACTIVE_ACCOUNT_TIME', 'FAILED_LOGIN_ATTEMPTS',
                         'PASSWORD_LIFE_TIME', 'PASSWORD_VERIFY_FUNCTION')
ORDER  BY profile, resource_name;

ALTER PROFILE app_profile LIMIT INACTIVE_ACCOUNT_TIME 35;
```

The 35 days come from the DISA STIG for 19c. Risk: an account used only by a quarterly job will get locked between runs. Give those accounts their own profile.

From 19.28 on, multi-factor authentication is available for native database users. Worth a look for human DBA accounts.

## 2. Cut what the application account can do

A leaked application password is only as bad as the grants behind it. To know which grants are really used, run privilege analysis. It is part of Enterprise Edition with no extra option, and needs the `CAPTURE_ADMIN` role.

```sql
BEGIN
  DBMS_PRIVILEGE_CAPTURE.CREATE_CAPTURE(
    name        => 'APP_USER_CAPTURE',
    description => 'Privileges actually used by APP_USER',
    type        => DBMS_PRIVILEGE_CAPTURE.G_CONTEXT,
    condition   => 'SYS_CONTEXT(''USERENV'',''SESSION_USER'') = ''APP_USER''');
  DBMS_PRIVILEGE_CAPTURE.ENABLE_CAPTURE(name => 'APP_USER_CAPTURE');
END;
/
```

Leave it on through a representative period, then:

```sql
EXEC DBMS_PRIVILEGE_CAPTURE.DISABLE_CAPTURE('APP_USER_CAPTURE')
EXEC DBMS_PRIVILEGE_CAPTURE.GENERATE_RESULT('APP_USER_CAPTURE')

SELECT *
FROM   dba_unused_privs
WHERE  capture = 'APP_USER_CAPTURE';
```

What shows up as unused is your revoke list. Risk: if the capture window misses month-end or a yearly batch, you will revoke something that is needed later. Keep the revoke script reversible.

## 3. Encrypt the wire

Native network encryption is configured in the server `sqlnet.ora`. The Security Guide's maximum security settings:

```bash
SQLNET.ENCRYPTION_SERVER = REQUIRED
SQLNET.ENCRYPTION_TYPES_SERVER = (AES256)
SQLNET.CRYPTO_CHECKSUM_SERVER = REQUIRED
SQLNET.CRYPTO_CHECKSUM_TYPES_SERVER = (SHA512)
SQLNET.ALLOW_WEAK_CRYPTO_CLIENTS = FALSE
```

Do not go straight to `REQUIRED` in production. Start with `REQUESTED`: clients on the default (`ACCEPTED`) negotiate encryption and nothing is refused. Then verify from a session:

```sql
SELECT network_service_banner
FROM   v$session_connect_info
WHERE  sid = SYS_CONTEXT('USERENV', 'SID');
```

You want to see the AES256 encryption and SHA512 crypto-checksumming adapters in the output. Once every client shows them, move to `REQUIRED`.

## 4. Encrypt at rest, and keep the keys somewhere else

Check where you stand:

```sql
SELECT tablespace_name, encrypted FROM dba_tablespaces;

SELECT con_id, wrl_parameter, status, wallet_type
FROM   v$encryption_wallet;
```

TDE tablespace encryption requires the Advanced Security option. It protects datafiles and backups that leave the server. It does nothing against a session that authenticated with a valid account, because decryption is transparent to authorized users. That is why it comes after items 1 and 2 on this list.

## 5. Make sure you would notice

Chainstack saw the intrusion two days after the attacker reached the database network. Check what your audit trail would show:

```sql
SELECT value FROM v$option WHERE parameter = 'Unified Auditing';

SELECT policy_name, enabled_option, entity_name, success, failure
FROM   audit_unified_enabled_policies;
```

19c runs in mixed mode by default, and databases created with DBCA come with `ORA_SECURECONFIG` and `ORA_LOGON_FAILURES` enabled. If they are missing, enable them:

```sql
AUDIT POLICY ORA_SECURECONFIG;
AUDIT POLICY ORA_LOGON_FAILURES WHENEVER NOT SUCCESSFUL;
```

Then add one policy for the table that would hurt the most:

```sql
CREATE AUDIT POLICY aud_customer_read
  ACTIONS SELECT ON app_schema.customers;

AUDIT POLICY aud_customer_read EXCEPT app_user;
```

Excluding `APP_USER` keeps the volume sane, and it leaves a gap: reads made with the stolen application credential are not recorded by this policy.

Two queries to put on a schedule:

```sql
SELECT event_timestamp, dbusername, userhost, client_program_name,
       action_name, object_schema, object_name, return_code
FROM   unified_audit_trail
WHERE  unified_audit_policies LIKE '%AUD_CUSTOMER_READ%'
ORDER  BY event_timestamp DESC
FETCH FIRST 50 ROWS ONLY;

SELECT dbusername, userhost, return_code, COUNT(*) AS attempts
FROM   unified_audit_trail
WHERE  action_name = 'LOGON'
AND    return_code <> 0
AND    event_timestamp > SYSTIMESTAMP - INTERVAL '1' DAY
GROUP  BY dbusername, userhost, return_code
ORDER  BY attempts DESC;
```

## Troubleshooting

**ORA-12660 after setting `REQUIRED`.** A client configured as `REJECTED`, or one that cannot negotiate the algorithms you listed, fails with `ORA-12660: Encryption or crypto-checksumming parameters incompatible`. Two documented cases: database links from 11g to 19c (MOS Doc ID 3005583.1) and JDBC clients (MOS Doc ID 2395707.1). The `REQUESTED` phase above is how you find them before they become an outage.

## Design considerations

This comes from the incidents, not from Oracle documentation.

A database password inside a client application is public. Whatever account the app uses, assume its credential will leak and size its privileges accordingly (item 2).

## Technical validation

Confirmed in Oracle documentation:

- `DBA_USERS_WITH_DEFPWD` query and the `INACTIVE_ACCOUNT_TIME` default (Security Guide 19c, Configuring Authentication)
- Schema-only accounts and `AUTHENTICATION_TYPE = NONE`; MFA for native users from 19.28 (Security Guide 19c, release changes)
- Privilege analysis in Enterprise Edition with `CAPTURE_ADMIN` (Privilege Analysis FAQ, Database Reference)
- `sqlnet.ora` maximum security settings (Security Guide 19c, chapter 18) and ORA-12660 behavior
- `CREATE AUDIT POLICY` and `AUDIT POLICY ... EXCEPT` syntax, `AUDIT_UNIFIED_ENABLED_POLICIES`, `UNIFIED_AUDIT_TRAIL` columns (SQL Language Reference, Database Reference)
- Predefined policies enabled by default on DBCA-created databases (Oracle Database Security product management material)

Market practice, not an Oracle requirement:

- 35 days of inactivity (DISA STIG for Oracle Database 19c)
- `SQLNET.ENCRYPTION_SERVER = REQUIRED` (CIS Oracle Database 19c Benchmark)

Varies by environment:

- `SQLNET.ALLOW_WEAK_CRYPTO_CLIENTS` depends on your Release Update level
- TDE needs the Advanced Security option
- In a multitenant database, run the checks in each PDB
- Audit volume depends on workload; measure before enabling object policies broadly

Not executed in a lab for this post: every command here was checked against documentation, none was run in this session. Test in non-production first.

## Conclusion

The breaches above were logins, not exploits. On the database side that turns into five checks: who can still log in, what those logins are allowed to do, whether traffic and files are encrypted, and whether you would see an unusual read on the table that matters. Each query in this post runs in minutes. Start with item 1 today.

## References

- Oracle Database Security Guide 19c, Configuring Authentication: https://docs.oracle.com/en/database/oracle/oracle-database/19/dbseg/configuring-authentication.html
- Oracle Database Security Guide 19c, Native Network Encryption and Data Integrity: https://docs.oracle.com/en/database/oracle/oracle-database/19/dbseg/configuring-network-data-encryption-and-integrity.html
- Oracle Database Security Guide 19c, Configuring Audit Policies: https://docs.oracle.com/en/database/oracle/oracle-database/19/dbseg/configuring-audit-policies.html
- Oracle Database Security Guide 19c, release changes: https://docs.oracle.com/en/database/oracle/oracle-database/19/dbseg/release-changes.html
- Privilege Analysis FAQ: https://www.oracle.com/a/tech/docs/dbsec/pa/faq-pa-19c.pdf
- Oracle Database Auditing, security and performance best practices: https://oracle.com/docs/tech/dbsec/oracle-database-auditing-security-and-perf-best-practices.pdf
- DISA STIG Oracle Database 19c, V-270551: https://www.stigviewer.com/stigs/oracle_database_19c/2026-02-25/finding/V-270551
