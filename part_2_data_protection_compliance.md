# Securing PostgreSQL: Towards Compliance with the Kenya Data Protection Act 2019

Watch the following video as part of this lab:

[![data_protection_in_Kenya_interview](https://raw.githubusercontent.com/course-files/RelationalAlgebra/refs/heads/main/assets/images/data_protection_in_Kenya_interview.png)](https://youtu.be/_dVRRgxqSQg)  

[https://youtu.be/_dVRRgxqSQg](https://youtu.be/_dVRRgxqSQg)

---

## Overview

Data security is not optional — it is a legal obligation. The **Kenya Data Protection Act, 2019 (No. 24 of 2019)**  (download link: [https://new.kenyalaw.org/akn/ke/act/2019/24/](https://new.kenyalaw.org/akn/ke/act/2019/24/) ) imposes binding duties on every organization that collects, stores, or processes personal data. A PostgreSQL database holding customer names, contact details, order histories, and employee records is unambiguously subject to this Act.

You must learn how to translate those legal obligations into concrete, executable technical controls. By the end of this part of the lab, your PostgreSQL instance should satisfy the baseline security requirements of the DPA 2019, and the controls should also be substantially aligned with the **EU General Data Protection Regulation (GDPR)** (download link: [https://gdpr-info.eu](https://gdpr-info.eu) or for the original PDF: [https://eur-lex.europa.eu/legal-content/EN/TXT/PDF/?uri=CELEX:32016R0679&from=EN](https://eur-lex.europa.eu/legal-content/EN/TXT/PDF/?uri=CELEX:32016R0679&from=EN) ) — the international benchmark against which many data protection frameworks are measured.

> **Prerequisites:** You must have completed Part 1 (PostgreSQL Installation). All commands in this lab are executed via SSH into the Ubuntu Server VM created in that lab, unless otherwise stated.

---

## Legal Framework

### Kenya Data Protection Act 2019 — Relevant Provisions

The Kenya Data Protection Act 2019 is a national statute, therefore, it is read as follows:

```text
s.41(4)(c) DPA 2019
│  │  │  │
│  │  │  └── paragraph (c) — lowest division
│  │  └───── subsection (4) — subdivision of section
│  └──────── section 41 — primary division of the Act
└─────────── "s." abbreviation for section
```

Section 41, subsection 4, paragraph (c) of the Kenya Data Protection Act 2019

| Section | Requirement | Technical Control in This Lab |
| --------- | ------------- | ------------------------------- |
| s.25(b) | Processed lawfully, fairly and transparently | Role-based access, audit logging |
| s.25(c) | Collected for explicit, specified purposes; not further processed incompatibly | Role permissions scoped to purpose |
| s.25(d) | Adequate, relevant, and limited to what is necessary | Least-privilege roles; revoke PUBLIC |
| s.25(g) | Not kept longer than necessary | Data retention policy (Part G) |
| s.25(h) | Not transferred outside Kenya without adequate safeguards | pg_hba.conf network restrictions |
| s.41(4)(c) | Pseudonymization and encryption of personal data | TLS in transit (Part A) |
| s.41(4)(d) | Ability to restore availability and access to data | Encrypted backups (Part H) |
| s.42(1)(c) | Transmission of data over ICT networks requires security measures | TLS enforcement (Part A) |
| s.43(1)(a) | Notify the Data Commissioner within 72 hours of a breach | Audit logging (Part D) |
| s.43(8) | Record facts, effects, and remedial action of each breach | Audit log retention |
| s.44–47 | Sensitive personal data requires additional safeguards | Row-level security (Part E) |

### GDPR — Equivalent Articles

The GDPR is not a national statute. It is an EU Regulation — a supranational legal instrument. EU legislative instruments are architecturally divided into articles, not sections. Therefore, "Art. 5(1)(f) GDPR" is read aloud as:

```text
Art. 5(1)(f) GDPR
│    │  │  │
│    │  │  └── subparagraph f — subdivision of paragraph
│    │  └───── paragraph 1 — subdivision of article
│    └───── Article 5 — primary division
└─────────── "Art." abbreviation for Article
```

Article 5, paragraph 1, subparagraph f of the General Data Protection Regulation

| GDPR Article | Requirement | Maps to DPA 2019 |
| ------------- | ------------- | ----------------- |
| Art. 5 | Principles of data protection | s.25 |
| Art. 25 | Data protection by design and by default | s.41 |
| Art. 32 | Security of processing: encryption, confidentiality, integrity, availability, resilience | s.41–42 |
| Art. 33 | 72-hour breach notification to supervisory authority | s.43 |
| Art. 83 | Administrative fines up to €20M or 4% of global annual turnover | s.63 (up to KES 5M or 1% of turnover) |

> **Examination note:** The DPA 2019 and GDPR are not identical, but their core structure and principles are closely aligned. Kenya's Act was explicitly drafted to achieve interoperability with GDPR and other international frameworks, facilitating cross-border data transfers. A system compliant with the controls in this lab should tend towards satisfying both.

---

## Setting the PostgreSQL Version Variable

All commands in this lab reference your PostgreSQL version. Set it once at the start of your SSH session and use it throughout the lab:

```bash
PG_VER=$(ls /etc/postgresql/ | sort -V | tail -1)
echo "PostgreSQL version: $PG_VER"
```

You should see something like `PostgreSQL version: 18`. If the directory contains multiple versions, the command selects the most recent. Every subsequent command in this lab that references `${PG_VER}` will use this value.

---

## Part A — Encryption in Transit (TLS/SSL)

**Legal basis:** s.41(4)(c) DPA 2019 requires encryption of personal data. s.42(1)(c) specifically addresses transmission of data over ICT networks. GDPR Art. 32(1)(a) requires "encryption of personal data" as an appropriate technical measure. GDPR Art. 32(1)(b) requires "the ongoing confidentiality, integrity, availability and resilience of processing systems and services."

**The problem being solved:** Without TLS, all data transmitted between your application server (or pgAdmin) and the PostgreSQL server — including query results containing customer names, phone numbers, and order details — travels as plaintext. Any device on the same network can intercept and read it. This is not a theoretical risk; tools such as [Wireshark](https://www.wireshark.org/download.html) make intercepting traffic being transmitted over a network trivial.

---

### Difference between OpenSSL and OpenSSH

**OpenSSL is a cryptographic library that developers embed into applications to implement secure protocols**. It provides the encryption engine for HTTPS, email security, VPNs, and certificate management, **but it does not provide connectivity itself**.

On the other hand, **OpenSSH is a complete suite of tools for secure remote access and file transfer**. It handles logins, file copying, tunneling, and port forwarding between machines, using its own independent cryptographic implementation (though it can also be configured to work with OpenSSL).

**In essence:** **OpenSSL** is the cryptographic toolkit that secures data in transit across various protocols, while **OpenSSH** is the connectivity toolset that secures remote machine access and operations—they serve different purposes, operate independently, and are not dependent on one another despite occasional integration.

Analogy: OpenSSL answers, "How do I encrypt this data?" while OpenSSH answers "How do I securely reach that machine?"

### A.1. Install OpenSSL in Linux

Most Linux distributions ship with OpenSSL by default because system utilities depend on it. However, assuming that there is no OpenSSL installed, below are the installation steps in a Linux Server (Ubuntu server).

Execute:

```shell
sudo apt update
sudo apt install openssl
```

Verify installation:

```shell
openssl version
which openssl
```

---

### Create a minimal OpenSSL config

This configuration creates a 2048-bit RSA, SHA-256 certificate for a local
Ubuntu server, with SANs allowing it to be recognized under the specified
hostname and local IP address.

It contains the following sections:

- **`[req]` - Certificate request settings:** Controls how OpenSSL creates the
  certificate request/certificate.
- **`[dn]` - Distinguished Name settings:** Specifies the details of the entity
  for which the certificate is being generated, such as country, state, locality,
  organization, organizational unit, and common name.
- **`[v3_req]` - X.509 v3 extensions:** Defines the extensions to be included
  in the certificate, such as Subject Alternative Names (SANs).
- **`[alt_names]` - Subject Alternative Names:** Lists the alternative DNS names
  and IP addresses that the certificate should be valid for.

Execute:

```shell
vim /home/student/openssl.cnf
```

Press `i` for **Insert Mode** in `vim` and paste the following inside the openssl.cnf file:

```text
[ req ]
default_bits       = 2048
prompt             = no
default_md         = sha256
x509_extensions    = v3_req
distinguished_name = dn

[ dn ]
C  = KE
ST = Nairobi County
L  = Nairobi
O  = Example Enterprises
OU = IT Department
CN = localhost

[ v3_req ]
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = localhost
DNS.2 = classlab
IP.1  = 127.0.0.1
```

Press `Esc` and then type `:wq` to write the changes to the file and quit `Vim`.

Move the configuration file to a shared location temporarily and allow the `postgres` user to access it:

```bash
sudo cp /home/student/openssl.cnf /tmp/openssl.cnf
# sudo chmod 644 /tmp/openssl.cnf
chmod u=rw,g=r,o=r /tmp/openssl.cnf
```

### A.2. Generate the public certificate using OpenSSL

Run the following command in the bash terminal to generate the public certificate using OpenSSL:

```shell
# Generate a self-signed X.509 certificate and private key
# Valid for 365 days, using the created configurations
sudo -u postgres openssl req -new -x509 -days 365 -nodes \
  -keyout /var/lib/postgresql/${PG_VER}/main/server.key \
  -out /var/lib/postgresql/${PG_VER}/main/server.crt \
  -config /tmp/openssl.cnf
```

Verify the files exist with correct ownership:

```bash
sudo ls -al /var/lib/postgresql/${PG_VER}/main/ | grep server
```

Expected output:

```text
-rw-r--r-- 1 postgres postgres 1234 ... server.crt
-rw------- 1 postgres postgres 1704 ... server.key
```

Set the correct file permissions. PostgreSQL will refuse to start SSL if the private key file is readable by anyone other than the `postgres` user:

```bash
sudo chmod u=rw,go= /var/lib/postgresql/${PG_VER}/main/server.key
sudo chmod u=rw,go=r /var/lib/postgresql/${PG_VER}/main/server.crt
sudo chown postgres:postgres \
  /var/lib/postgresql/${PG_VER}/main/server.crt \
  /var/lib/postgresql/${PG_VER}/main/server.key
```

> **Production note:** A self-signed certificate causes clients to see a certificate warning because no trusted Certificate Authority (CA) has verified its authenticity. In production, use a certificate from Let's Encrypt (free), a commercial CA such as DigiCert, or your organization's internal CA. The technical configuration steps are identical — only the certificate files differ.

The commands above do the following 2 tasks:

1. Creates a new private key → **server.key**
This is secret. **YOU SHOULD NOT SHARE IT PUBLICLY.**

2. Creates a new public certificate → **server.crt**
This is the self-signed certificate.
It contains the “public half” of the server's identity.
Other servers use it to set up encrypted communication.

Key Points to Note:

1. With a Self-Signed Certificate (our current setup for educational purposes):
We generate and issue **both the private key (.key) and the certificate (.crt)** ourselves.
A browser would say: “I do not know this Certificate Authority that issued this
certificate (you issued the certificate yourself), so I cannot trust this identity.”
**Encryption still works (data is scrambled), but identity is not trusted.**
Anyone could generate a certificate for localhost or even google.com if it is self-signed.

2. With a Trusted Certificate Authority (real-world setup)
You create a Certificate Signing Request (CSR)
This file contains your domain name (e.g., `yourdomain.co.ke`) and your public key (certificate).
You generate the public key (certificate) from your private key.
You then send the CSR to a Certificate Authority (CA)

Examples of CAs: [Let’s Encrypt (free)](https://letsencrypt.org/), [DigiCert](https://www.digicert.com/), [GlobalSign](https://www.globalsign.com/en), etc.

The CA confirms that you actually own `yourdomain.co.ke`.

The CA then signs your CSR. This produces a certificate (`yourdomain.crt`) that says:
“The CA vouches that the owner of this public key (certificate) really owns `yourdomain.co.ke`.”

The difference is that browsers trust your public certificate because they already trust the CA.

---

### A.3 — Enable SSL in PostgreSQL

```bash
sudo -u postgres psql -c "ALTER SYSTEM SET ssl = on;"
```

This writes to `postgresql.auto.conf`, which is evaluated after `postgresql.conf`. A restart is required because `ssl` is a startup parameter:

```bash
sudo systemctl restart postgresql
sudo systemctl status postgresql
```

Confirm SSL is now active:

```bash
sudo -u postgres psql -c "SHOW ssl;"
```

Expected output: `on`

---

### A.4 — Enforce SSL for Remote Connections

Enabling SSL allows clients to use it but it does not make it required by default. To make SSL mandatory for all connections from the Host-Only network, change the `host` keyword in `pg_hba.conf` to `hostssl`.

```bash
sudo vim /etc/postgresql/${PG_VER}/main/pg_hba.conf
```

Find the line you added in Part 1 (near the bottom):

```text
host       all    all    192.168.56.0/24    scram-sha-256
```

Change it to:

```text
hostssl    all    all    192.168.56.0/24    scram-sha-256
```

Your settings should, therefore, look like this (depending on the IP address of the VirtualBox Host-Only adapter):

```text
# DO NOT DISABLE!
# If you change this first entry you will need to make sure that the
# database superuser can access the database using some other method.
# Noninteractive access to all databases is required during automatic
# maintenance (custom daily cronjobs, replication, and similar tasks).
#
# Database administrative login by Unix domain socket
local   all             postgres                                peer

# TYPE  DATABASE        USER            ADDRESS                 METHOD

# "local" is for Unix domain socket connections only
local   all             all                                     peer
# IPv4 local connections:
host    all             all             127.0.0.1/32            scram-sha-256
# IPv6 local connections:
host    all             all             ::1/128                 scram-sha-256
# Allow replication connections from localhost, by a user with the
# replication privilege.
local   replication     all                                     peer
host    replication     all             127.0.0.1/32            scram-sha-256
host    replication     all             ::1/128                 scram-sha-256

hostssl all             all             192.168.56.0/24         scram-sha-256
```

The difference:

| Keyword | Meaning |
| --------- | --------- |
| `host` | Accepts both encrypted (SSL) and unencrypted connections |
| `hostssl` | Accepts **only** SSL-encrypted connections; unencrypted attempts are rejected |
| `hostnossl` | Accepts **only** unencrypted connections (you should not use this option) |

Reload the configuration (no restart required for pg_hba.conf changes):

```bash
sudo systemctl reload postgresql
```

---

### A.5 — Test SSL Enforcement

**Note:** Replace `192.168.56.104` with your VM's actual Host-Only IP address.

**Test 1:** Connect from within the VM with SSL explicitly required. This should succeed:

```bash
psql "user=postgres host=192.168.56.104 port=5432 dbname=postgres sslmode=require"
```

Inside psql, verify the connection is encrypted:

```sql
\conninfo
```

Expected output (partial):

```text
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, ...)
```

```sql
\q
```

**Test 2:** Attempt to connect without SSL. This should fail after enforcing `hostssl`:

```bash
psql "user=postgres host=192.168.56.104 port=5432 dbname=postgres sslmode=disable"
```

Expected error:

```text
psql: error: connection to server at "192.168.56.104", port 5432 failed:
FATAL: no pg_hba.conf entry for host "192.168.56.104", user "postgres",
database "postgres", no encryption
```

This error confirms that unencrypted connections are now rejected. A potential interceptor who captures traffic cannot read it even if they are on the same network.

**Test 3:** Connect from pgAdmin on your laptop.

In pgAdmin, open the existing server connection to the VM. pgAdmin negotiates SSL automatically. The connection should succeed. To confirm SSL is enabled: right-click on the `postgres` database, then select `PSQL Tool`. Execute the following:

```shell
\conninfo
```

The expected output is:

```text
postgres=# \conninfo
            Connection Information
      Parameter       |         Value
----------------------+------------------------
 Database             | postgres
 Client User          | postgres
 Host                 | 192.168.56.104
 Server Port          | 5432
 Options              |
 Protocol Version     | 3.0
 Password Used        | true
 GSSAPI Authenticated | false
 Backend PID          | 1953
 SSL Connection       | true
 SSL Library          | OpenSSL
 SSL Protocol         | TLSv1.3
 SSL Key Bits         | 256
 SSL Cipher           | TLS_AES_256_GCM_SHA384
 SSL Compression      | false
 ALPN                 | postgresql
 Superuser            | on
 Hot Standby          | off
(18 rows)
```

Execute the following to confirm that SSL is enabled:

```sql
SELECT ssl, version, cipher, bits FROM pg_stat_ssl WHERE pid = pg_backend_pid();
```

**Test 4:** Connect from DataGrip on your laptop

DataGrip does not auto-negotiate SSL the way pgAdmin does. You have to tell it explicitly how to handle the certificate. The settings that matter are under the SSH/SSL tab of your data source configuration.

Go to Database panel → your data source → Properties (F4) → SSH/SSL tab → SSL section.

Setting: Use SSL  
Value: Checked  
Reason: Enables SSL negotiation

Setting: SSL mode  
Value: Require  
Reason: Equivalent to `sslmode=require` This enforces encryption but does not verify the CA

To confirm that SSL is active: right-click the connection → **New** → **Query Console** tab. Then execute  the same query as you did in pgAdmin:

```sql
SELECT ssl, version, cipher, bits
FROM pg_stat_ssl
WHERE pid = pg_backend_pid();
```

---

**NOTE:** You need to load the synthetic data as guided here before you proceed
to Part B: [Synthetic Data](part_3_synthetic_data.md)

---

## Part B — Role-Based Access Control and Least Privilege

**Legal basis:** s.25(d) DPA 2019 requires that processing be "adequate, relevant, limited to what is necessary." s.41(1)(a) requires implementing data protection principles in an effective manner. GDPR Art. 5(1)(c) is the equivalent provision.

---

### B.1 — Revoking the Default PUBLIC Permissions (Already Completed)

This step was completed when you executed the `siwaka_dishes.sql` DDL script. The following two statements were already run inside that script:

```sql
REVOKE ALL ON DATABASE siwaka_dishes FROM PUBLIC;
REVOKE CREATE ON SCHEMA public FROM PUBLIC;
```

**What these statements do and why they matter:**

By default, PostgreSQL grants a built-in pseudo-role called `PUBLIC` — which every database user belongs to automatically — the ability to connect to any database and to create objects inside the `public` schema. This means a freshly created user with no explicit permissions could still connect and create tables. That is not acceptable in a system that processes personal data.

The first statement removes the default `CONNECT` privilege, so no user can access the database unless explicitly granted `CONNECT`. This is similar to a firewall rule that blocks all incoming connections unless explicitly allowed. The second removes the ability to create objects in the `public` schema, so only users who have been deliberately granted `CREATE` can define tables, views, or functions.

Verify that this is in place:

```bash
sudo -u postgres psql -d siwaka_dishes -c \
  "SELECT has_schema_privilege('public', 'public', 'CREATE') AS public_can_create;"
```

Expected result: `false`. If the result is `true`, the DDL script did not execute completely — re-run the two REVOKE statements manually.

---

### B.2 — The Role Structure (Already Completed)

This step was also completed in the DDL script. Four roles were created, each modelling a distinct tier of access:

```sql
CREATE USER siwaka_dishes_db_admin    WITH PASSWORD 'xxxx';
CREATE USER siwaka_dishes_app_runtime WITH PASSWORD 'xxxx';
CREATE USER siwaka_dishes_analytics   WITH PASSWORD 'xxxx';
CREATE USER siwaka_dishes_backup      WITH PASSWORD 'xxxx';
```

The script then granted permissions appropriate to each role and used `ALTER DEFAULT PRIVILEGES` to ensure that any tables created in the future automatically inherit the same permission structure. This is the correct production pattern and is worth understanding in detail.

| Role | Access Granted | Intended User |
| ------ | --------------- | --------------- |
| `siwaka_dishes_db_admin` | Owns the schema and all tables. Full DDL rights. | The DBA — connects via SSH and psql for maintenance, schema changes, and administration. |
| `siwaka_dishes_app_runtime` | `SELECT, INSERT, UPDATE, DELETE` on all tables. Sequence access for auto-generated IDs. | The **application server** — the Python + Django, Node.js + Express.js, PHP + Laravel, Ruby + Rails, Java + Spring Boot, or other backend process that the Information System runs as. It connects continuously and performs all CRUD operations on behalf of users. |
| `siwaka_dishes_analytics` | `SELECT` on all tables. Read-only. | Reporting dashboards, BI tools, auditors, and data analysts who query the database directly. |
| `siwaka_dishes_backup` | `SELECT` on all tables. Read-only. | The automated backup process (`pg_dump`), which needs to read all data but must never modify it. |

**Why this four-role pattern is used in production:**

This is a **tier-based** role model. It separates privileges by the *nature of the activity* rather than the *job title of the person*. The critical insight, documented in the DDL script's comments, is: **assume credentials will eventually leak; therefore, design damage containment in advance.**

If the application server's credentials (`app_runtime`) are compromised, an attacker gains CRUD access to the `siwaka_dishes` database only. They cannot drop tables, alter schemas, access other databases, or create new users. The **blast radius** is contained to data manipulation within one database. If the `analytics` credentials are compromised, the attacker gains read access only — they can see data but cannot modify or delete it.

**How this relates to business-function roles such as cashier, chef, and manager:**

You may encounter role designs where each job title in the business gets its own database role — for example, `siwaka_cashier`, `siwaka_chef`, and `siwaka_manager`. Those are **business-function roles** and model what a job title is permitted to do. The roles in our DDL are **infrastructure roles** and model how a system component interacts with the database.

In a production system, both layers typically coexist:

- The **application runtime** credential (`siwaka_dishes_app_runtime`) is what the backend server uses for all automated queries. The application code itself enforces business rules — a cashier's session in the web application cannot modify staff records because the **application logic** prevents it, not because the database role prevents it.
- **Human** users who need direct SQL access — auditors, managers, DBAs — are given credentials mapped to `analytics` (read-only) or `db_admin` (administrative). Individual humans should almost never connect as `app_runtime` because that credential is not auditable at the individual level.

The row-level security policies in Part E of this lab add a further layer of data isolation directly in the database, which complements but does not replace the above.

Verify the role structure is in place:

```bash
sudo -u postgres psql -d siwaka_dishes -c "\du"
```

Confirm you see `siwaka_dishes_analytics`, `siwaka_dishes_app_runtime`, and `siwaka_dishes_backup`, and `siwaka_dishes_db_admin` listed.

---

### B.3 — Create Named Human Users

The four roles from B.2 are credentials for **system processes**, not for individual people. When a human needs direct database access — for auditing, incident investigation, or operational reporting — they must connect under their own named account, not a shared system credential.

This is a requirement of s.43(8) of the DPA 2019, which mandates that you record exactly who accessed what data during a breach investigation. A shared credential makes individual attribution impossible.

Create named accounts and assign them to the appropriate tier role:

```bash
sudo -u postgres psql -d siwaka_dishes
```

```sql
-- Auditor: read-only access for compliance review.
-- Maps to the analytics tier — can SELECT but cannot modify any data.
CREATE USER auditor_atwoli
  WITH ENCRYPTED PASSWORD 'Auditor@Siwaka2026!'
  VALID UNTIL '2026-12-31';

GRANT CONNECT ON DATABASE siwaka_dishes TO auditor_atwoli;
GRANT siwaka_dishes_analytics TO auditor_atwoli;

\q
```

> **Why employees do not get individual database accounts:** Individual staff members such as cashiers, chefs, etc. interact with the system through a web or mobile application, not through a direct database connection. The application authenticates them at the application layer and connects to the database as `siwaka_dishes_app_runtime` or a specific overall account.
>
> Giving every individual employee a personal database credential creates an unmanageable number of accounts and those individuals have no legitimate reason to run raw SQL against the database. If an employee needs to run a report, they should do so through the application or via a business intelligence tool that connects with the `analytics` role, not by connecting directly to the database with their own account.

---

### B.4 — Block the Superuser from Remote Connections

The `postgres` superuser bypasses all permission checks, all row-level security policies, and all audit controls. It must never be reachable from the network. All superuser administration must go through SSH first and then connect locally.

```bash
sudo vim /etc/postgresql/${PG_VER}/main/pg_hba.conf
```

Add the following line **immediately before** the `hostssl all all 192.168.x.0/24` line:

```text
# Block remote superuser access — DPA 2019 s.41, GDPR Art.32
hostssl    all    postgres    192.168.56.0/24    reject
```

The relevant section of the file should now read:

```text
hostssl    all    postgres    192.168.56.0/24    reject
hostssl    all    all         192.168.56.0/24    scram-sha-256
```

> **Critical:** `pg_hba.conf` is evaluated top to bottom and stops at the first matching rule. The `reject` rule for `postgres` must appear before the general `all` rule. If the order is reversed, the general rule matches first and the reject rule is never reached.

Reload the configuration:

```bash
sudo systemctl reload postgresql
```

---

### B.5 — Verify Permissions

**Test 1:** Confirm the auditor has read access but cannot write:

```bash
psql "host=127.0.0.1 dbname=siwaka_dishes user=auditor_atwoli sslmode=require"
```

```sql
-- This should succeed
SELECT COUNT(*) FROM customer_order;

-- This should fail — the role has SELECT only
DELETE FROM customer WHERE customer_number = 1;
-- Expected: ERROR: permission denied for table customer

\q
```

**Test 2:** Confirm the `app_runtime` credential can perform CRUD but cannot alter the schema:

```bash
psql "host=127.0.0.1 dbname=siwaka_dishes user=siwaka_dishes_app_runtime sslmode=require"
```

```sql
-- This should succeed
SELECT COUNT(*) FROM product;

-- This should fail — app_runtime cannot create or alter tables
CREATE TABLE test_table (id INT);
-- Expected: ERROR: permission denied for schema public

\q
```

**Test 3:** Confirm the superuser is rejected from remote connections.

From your **laptop's terminal (using psql)** or via **pgAdmin** or **DBeaver** or **DataGrip**, attempt to connect to the VM:

```bash
psql "host=192.168.56.103 dbname=siwaka_dishes user=postgres sslmode=require"
```

Replace `192.168.56.103` with your VM's actual Host-Only IP address.

Expected:

```text
FATAL: pg_hba.conf rejects connection for host "192.168.56.103",
user "postgres", database "siwaka_dishes", SSL on
```

The `postgres` superuser is now reachable only from within the server itself via a Unix socket or loopback. All remote administration must go through SSH first — adding SSH as a mandatory authentication layer before any database interaction is possible.

---

## Part C — Blocking the postgres Superuser from Remote Connections (Verification)

Verify that the reject rule from Part B is working correctly:

From your laptop's terminal, attempt to connect as `postgres` to the VM:

```bash
psql "host=192.168.56.104 dbname=postgres user=postgres sslmode=require"
```

Replace the IP with your VM's actual Host-Only address.

Expected:

```text
psql: error: connection to server at "192.168.56.104", port 5432 failed:
FATAL: pg_hba.conf rejects connection for host "192.168.56.104",
user "postgres", database "postgres", SSL on
```

The `postgres` superuser is now reachable only from within the server itself (via Unix socket or loopback with peer/password authentication). All remote administration must go through SSH first, then psql locally — adding SSH as a mandatory authentication layer.

---

## Part D — Audit Logging with pgAudit

**Legal basis:** s.43(1)(a) DPA 2019 requires notifying the Data Commissioner of a personal data breach within 72 hours. This is only possible if you have audit logs showing what data was accessed, when, and by whom. s.43(8) requires recording "the facts relating to the breach, its effects, and the remedial action taken." Without audit logging, you cannot satisfy either requirement. GDPR Art. 33 contains an identical 72-hour notification obligation.

**The problem being solved:** PostgreSQL's default logging records server events (connections, errors, crashes) but not individual SQL statements. If an employee runs `SELECT * FROM customer` at 2 am and exports the results, the default PostgreSQL logs contain no record of it. `pgaudit` fills this gap by recording every SQL statement executed, against which database objects, by which user, at what time.

---

### D.1 — Install pgAudit

```bash
sudo apt update
sudo apt install -y postgresql-${PG_VER}-pgaudit
```

Verify the package installed:

```bash
dpkg -l | grep pgaudit
```

---

### D.2 — Configure pgAudit

pgAudit must be loaded at server startup via `shared_preload_libraries`. This requires a full restart (not just a reload):

```bash
sudo -u postgres psql -c "ALTER SYSTEM SET shared_preload_libraries = 'pgaudit';"
```

> **Note:** When you later add the `passwordcheck` module (Part F), you will need to update this to `'pgaudit,passwordcheck'`. The order matters — place `pgaudit` first.

Set the audit logging level. For DPA 2019 compliance, record data writes, schema changes, and role changes at a minimum:

```bash
sudo -u postgres psql -c "ALTER SYSTEM SET pgaudit.log = 'write, ddl, role';"
```

| `pgaudit.log` value | What it records |
| ------------------- | ---------------- |
| `read` | SELECT, COPY FROM |
| `write` | INSERT, UPDATE, DELETE, TRUNCATE, COPY TO |
| `ddl` | CREATE, ALTER, DROP on any database object |
| `role` | GRANT, REVOKE, CREATE/ALTER/DROP ROLE |
| `function` | Function and procedure calls |
| `all` | Everything above |

For a restaurant system with customer personal data, `'write, ddl, role'` is the practical minimum. A health or financial system would typically use `'all'`.

Restart PostgreSQL to load the library:

```bash
sudo systemctl restart postgresql
sudo systemctl status postgresql
```

---

### D.3 — Enable pgAudit per Database

pgAudit must be activated as an extension in each database that should be audited:

```bash
sudo -u postgres psql -d siwaka_dishes -c "CREATE EXTENSION IF NOT EXISTS pgaudit;"
```

Verify the extension is active:

```bash
sudo -u postgres psql -d siwaka_dishes -c "SELECT extname, extversion FROM pg_extension WHERE extname = 'pgaudit';"
```

---

### D.4 — Enable Connection Logging

Beyond statement logging (pgaudit), you can also log who connects and disconnects to the database:

```bash
sudo -u postgres psql -c "ALTER SYSTEM SET log_connections = on;"
sudo -u postgres psql -c "ALTER SYSTEM SET log_disconnections = on;"
sudo -u postgres psql -c "ALTER SYSTEM SET log_line_prefix = '%m [%p] %quser=%u db=%d client=%r session=%c line=%l app=\"%a\" ';"
sudo -u postgres psql -c "SELECT pg_reload_conf();"
```

The `log_line_prefix` format inserts the timestamp, process ID, user, database, application name, and client IP into every log line. This is the minimum information needed for a breach investigation.

| Escape | Meaning | Example |
| ------ | ------- | ------- |
| `%m` | Timestamp with milliseconds and time zone (**when**) | `2026-09-19 05:46:38.430 EAT` |
| `%p` | Process ID of the server process | `[10135]` |
| `%q` | Not printed; hides the rest of the prefix on background-process lines | — |
| `%u` | Database user (**who**) | `user=postgres` |
| `%d` | Database name (**where**) | `db=siwaka_dishes` |
| `%r` | Client IP and port (**from where**); `[local]` for Unix socket connections | `client=192.168.56.1(53412)` |
| `%c` | Session ID; links all lines of one session | `session=6aadf734.2797` |
| `%l` | Line number within the session (**order**) | `line=5` |
| `%a` | Application name set by the client (**which tool**); not trustworthy | `app="psql"` |

---

### D.5 — Test Audit Logging

Open a second SSH terminal and watch the log file in real time:

```bash
# Terminal 2 — watch the log
PG_VER=$(ls /etc/postgresql/ | sort -V | tail -1)
echo "PostgreSQL version: $PG_VER"

sudo tail -f /var/log/postgresql/postgresql-${PG_VER}-main.log
```

In Terminal 1, perform some actions on the database:

```bash
sudo -u postgres psql -d siwaka_dishes
```

```sql
-- Insert a test order
INSERT
INTO public.customer(customer_name,
                     contact_first_name,
                     contact_last_name,
                     phone,
                     address_line1,
                     address_line2,
                     postal_code,
                     county,
                     sub_county,
                     status)
VALUES ('Test User',
        'Test',
        'User',
        '0720123456',
        '67 Ole Sangale Road',
        'MF 92 Apt 8',
        '00100',
        'Nairobi',
        'Langata',
        1);

-- Delete it (Assuming the insertion above was for customer_number 301)
DELETE FROM customer WHERE customer_number = 301;
\q
```

In Terminal 2, you should see log entries similar to:

```text
2026-09-19 06:02:36.915 EAT [10312] user=postgres db=siwaka_dishes client=[local] session=6aadfb3e.2848 line=4 app="psql" LOG:  AUDIT: SESSION,1,1,WRITE,INSERT,,,"INSERT
        INTO public.customer(customer_name,
                             contact_first_name,
                             contact_last_name,
                             phone,
                             address_line1,
                             address_line2,
                             postal_code,
                             county,
                             sub_county,
                             status)
        VALUES ('Test User',
                'Test',
                'User',
                '0720123456',
                '67 Ole Sangale Road',
                'MF 92 Apt 8',
                '00100',
                'Nairobi',
                'Langata',
                1)",<not logged>


2026-09-19 06:02:36.971 EAT [10312] user=postgres db=siwaka_dishes client=[local] session=6aadfb3e.2848 line=5 app="psql" LOG:  AUDIT: SESSION,2,1,WRITE,DELETE,,,DELETE FROM customer WHERE customer_number = 301,<not logged>


2026-09-19 06:02:36.983 EAT [10312] user=postgres db=siwaka_dishes client=[local] session=6aadfb3e.2848 line=6 app="psql" LOG:  disconnection: session time: 0:00:14.965 user=postgres database=siwaka_dishes host=[local]
```

Each audit log entry has two parts: the **prefix** (explained in the table above) and the **pgAudit record**, which follows `LOG:  AUDIT:`. The pgAudit record contains:

- `SESSION` — session-level auditing (as opposed to object-level auditing)
- `1,1` — statement number and sub-statement number within the session. A sub-statement is a statement PostgreSQL runs internally, e.g., `2,2` for a cascading delete.
- `WRITE` — the audit class matched
- `INSERT` / `DELETE` — the SQL command
- `,,` — object type and object name. These are empty by default; set `pgaudit.log_relation = on` to record them, e.g., `TABLE,public.customer`.
- The full SQL statement. It is enclosed in double quotes when it spans several lines or contains commas.
- `<not logged>` — the statement's bound parameters, which are not recorded unless `pgaudit.log_parameter = on`.

The final line (`disconnection: ...`) is not a pgAudit entry. It comes from `log_disconnections = on` and records the session's duration, user, database, and host.

> **Note:** The audit log records the full statement, including the personal data inside it (name, phone number, address). The log files are therefore personal data under the DPA 2019 and must be protected accordingly.

---

### D.6 — Understanding Audit Log Retention

The audit log is only useful if it is retained long enough to investigate incidents. The DPA 2019 does not specify a retention period for audit logs, but standard practice is 12 months online and 7 years in archive. On Ubuntu, PostgreSQL logs rotate weekly by default. You can configure the rotation according to your needs.

```bash
sudo -u postgres psql -c "ALTER SYSTEM SET log_rotation_age = '1d';"
sudo -u postgres psql -c "ALTER SYSTEM SET log_rotation_size = '100MB';"
sudo -u postgres psql -c "SELECT pg_reload_conf();"
```

In this case, we command PostgreSQL to create a log every day, e.g.,

```text
postgresql-2026-06-01.log
postgresql-2026-06-02.log
postgresql-2026-06-03.log
...
```

We also command PostgreSQL to create a new log file when the current one reaches 100MB in size. This prevents a single log file from growing indefinitely and consuming all disk space, e.g.,

```text
postgresql-2026-06-07_000.log  (100 MB)
postgresql-2026-06-07_001.log  (100 MB)
postgresql-2026-06-07_002.log
```

In a production environment, audit logs are shipped to a **centralized and secure, write-once log management system** (such as [Elasticsearch](https://www.elastic.co/), [Splunk](https://www.splunk.com/), or [AWS CloudWatch](https://aws.amazon.com/cloudwatch/)) so that **a compromised database server cannot be used to erase evidence of the breach**.

---

## Part E — Row-Level Security (RLS)

**Legal basis:** s.44–47 DPA 2019 impose additional requirements on sensitive personal data. Even within a database where a user has `SELECT` privilege on a table, they should see only the rows relevant to their role. s.25(d) reinforces data minimisation. GDPR Art. 5(1)(c) is the equivalent provision.

**The problem being solved:** The **Assistant Manager** role can view orders. But an Assistant Manager at a branch in Nairobi should not be able to view orders from a branch in Mombasa, as per the rules of this specific business. Column-level privileges control which columns are visible; row-level security controls which rows are visible.

We create the Assistant Manager accounts (**Note:** these are not individual accounts for an employee, e.g., Kiprono's account or Nyaga's account. They are shared accounts for the role of Assistant Manager at a particular branch.)

Connect to the database:

```bash
sudo -u postgres psql -d siwaka_dishes
```

```sql
-- For the assistant manager working in Nairobi
CREATE USER siwaka_dishes_app_runtime_ass_mgr_nbi
  WITH ENCRYPTED PASSWORD 'ass_mgrNBI@Siwaka2026!'
  VALID UNTIL '2026-12-31';

GRANT CONNECT ON DATABASE siwaka_dishes
  TO siwaka_dishes_app_runtime_ass_mgr_nbi;
GRANT siwaka_dishes_app_runtime
  TO siwaka_dishes_app_runtime_ass_mgr_nbi;

-- For the assistant manager working in Mombasa
CREATE USER siwaka_dishes_app_runtime_ass_mgr_mba
  WITH ENCRYPTED PASSWORD 'ass_mgrMombasa@Siwaka2026!'
  VALID UNTIL '2026-12-31';

GRANT CONNECT ON DATABASE siwaka_dishes
  TO siwaka_dishes_app_runtime_ass_mgr_mba;
GRANT siwaka_dishes_app_runtime
  TO siwaka_dishes_app_runtime_ass_mgr_mba;

-- For managers
CREATE USER siwaka_dishes_app_runtime_mgr
  WITH ENCRYPTED PASSWORD 'mgr@Siwaka2026!'
  VALID UNTIL '2026-12-31';

GRANT CONNECT ON DATABASE siwaka_dishes
  TO siwaka_dishes_app_runtime_mgr;
GRANT siwaka_dishes_app_runtime
  TO siwaka_dishes_app_runtime_mgr;

\q
```

---

### E.1 — Enable Row-Level Security on the customer_order Table

```bash
sudo -u postgres psql -d siwaka_dishes
```

```sql
-- Enable RLS on the customer_order table
ALTER TABLE customer_order ENABLE ROW LEVEL SECURITY;

-- By default, enabling RLS means NO rows are visible to non-superusers.
-- We now explicitly define who can see what.

-- Policy 1: Assistant managers see only orders from their assigned branch.
-- The application sets app.current_branch when the user logs in.
-- branch_code is an INTEGER in this schema, so we cast the session
-- variable from TEXT to INT before comparing.
CREATE POLICY branch_isolation_policy
  ON customer_order
  FOR ALL
  TO siwaka_dishes_app_runtime_ass_mgr_nbi,
     siwaka_dishes_app_runtime_ass_mgr_mba
  USING (
    branch_code = current_setting('app.current_branch', TRUE)::INT
  );

-- Policy 2: Managers see all orders (no branch restriction).
CREATE POLICY manager_full_access_policy
  ON customer_order
  FOR ALL
  TO siwaka_dishes_app_runtime_mgr
  USING (TRUE);
```

> **Why the `::INT` cast is required:** `current_setting()` always returns a value of type `TEXT`. The `branch_code` column in the `customer_order` table is an `INT`. PostgreSQL will not automatically coerce TEXT to INT in a comparison, so without the explicit cast the policy either errors or never matches. The cast also handles the case where the setting has not been set — `current_setting('app.current_branch', TRUE)` returns `NULL` when the setting is absent, and `NULL::INT` remains `NULL`, meaning no rows match. This is the safest default behaviour: a user who has not had their branch context set by the application sees nothing rather than everything.

---

### E.2 — Test Row-Level Security

Before running the tests, find the actual `branch_code` values in the data:

```sql
SELECT branch_code, county, sub_county FROM branch;
```

Use the `branch_code` values from the output in the SET commands below.

**Test 1:** As `postgres` superuser (bypasses RLS), all orders are visible:

```sql
SELECT COUNT(*) FROM customer_order;
-- Returns: total number of orders across all branches
\q
```

**Test 2:** As the Nairobi assistant manager, only orders from the configured branch are visible.

```bash
psql "host=127.0.0.1 dbname=siwaka_dishes \
      user=siwaka_dishes_app_runtime_ass_mgr_nbi sslmode=require"
```

```sql
-- Without setting the branch context, no orders are visible.
-- current_setting returns NULL, which matches nothing.
SELECT COUNT(*) FROM customer_order;
-- Returns: 0

-- Simulate the application setting the branch context on login.
-- Replace [branch_code_1] with an actual integer from your branch table.
SET app.current_branch = '[branch_code_1]';

SELECT COUNT(*) FROM customer_order;
-- Returns: only orders belonging to that branch

-- Attempt to see another branch's orders by changing the context.
-- The database allows the SET command but the RLS policy still applies.
SET app.current_branch = '[branch_code_2]';

SELECT COUNT(*) FROM customer_order;
-- Returns: only orders for branch_code_2

\q
```

**Test 3:** As the manager, all orders are visible regardless of context:

```bash
psql "host=127.0.0.1 dbname=siwaka_dishes \
      user=siwaka_dishes_app_runtime_mgr sslmode=require"
```

```sql
SELECT COUNT(*) FROM customer_order;
-- Returns: total number of orders across all branches
-- The manager_full_access_policy uses USING (TRUE), so all rows match.

\q
```

> **How this works in an application:** When a staff member logs in, the application backend authenticates them, determines their branch from the HR system (e.g., from the `employee` table, branch_code 3 = `Nairobi, Kasarani` branch), and immediately runs `SET app.current_branch = '3';` before executing any query. The application controls context; the database enforces isolation. RLS is enforced at the database engine level and cannot be bypassed by modifying the application query.

---

### E.3 — Enable RLS on the customer Table

Customer personal data — including names, phone numbers, and address details — is unambiguously personal data under s.2 of the DPA 2019. An assistant manager should be able to look up only the customers who have placed orders at their branch, not the entire customer base.

```bash
sudo -u postgres psql -d siwaka_dishes
```

```sql
ALTER TABLE customer ENABLE ROW LEVEL SECURITY;

-- Assistant managers can only see customers who have placed an order
-- at their assigned branch. The subquery joins customer to customer_order
-- on customer_number, which is the correct column name in this schema.
CREATE POLICY customer_branch_isolation_policy
  ON customer
  FOR SELECT
  TO siwaka_dishes_app_runtime_ass_mgr_nbi,
     siwaka_dishes_app_runtime_ass_mgr_mba
  USING (
    customer_number IN (
      SELECT DISTINCT customer_number
      FROM customer_order
      WHERE branch_code = current_setting('app.current_branch', TRUE)::INT
    )
  );

-- Managers see all customers.
CREATE POLICY customer_manager_policy
  ON customer
  FOR ALL
  TO siwaka_dishes_app_runtime_mgr
  USING (TRUE);

\q
```

**Test 1:** As the Nairobi assistant manager, only customers who have placed orders at their branch are visible.

```bash
psql "host=127.0.0.1 dbname=siwaka_dishes \
      user=siwaka_dishes_app_runtime_ass_mgr_nbi sslmode=require"
```

```sql
-- Without setting the branch context, no customers are visible.
SELECT COUNT(*) FROM customer;
-- Returns: 0

-- Simulate the application setting the branch context on login.
SET app.current_branch = '2';

SELECT COUNT(*) FROM customer;
-- Returns: only customers who have placed orders at branch 2

\q
```

**Test 2:** As the manager that has access to all customer data:

```bash
psql "host=127.0.0.1 dbname=siwaka_dishes \
      user=siwaka_dishes_app_runtime_mgr sslmode=require"
```

```sql
SELECT COUNT(*) FROM customer;
-- Returns: total number of customers across all branches
-- The manager_full_access_policy uses USING (TRUE), so all rows match.

\q
```

---

## Part F — Password Policies

**Legal basis:** s.41(4)(b) DPA 2019 requires establishing and maintaining "appropriate safeguards against identified risks." Weak passwords are a documented, foreseeable risk. GDPR Art. 32(1)(b) requires "the ability to ensure the ongoing confidentiality of processing systems." Brute-forced or guessed credentials directly undermine confidentiality.

---

### F.1 — Enable the passwordcheck Module

`passwordcheck` is a PostgreSQL contributed module that intercepts `CREATE ROLE ... PASSWORD` and `ALTER ROLE ... PASSWORD` statements and rejects passwords that do not meet minimum complexity requirements:

- Minimum 8 characters
- If fewer than 16 characters: must contain both alphabetic and non-alphabetic characters

> `passwordcheck` is included in `postgresql-contrib`.

Install the `postgresql-contrib` package if you haven't already:

```bash
sudo apt update
sudo apt install -y postgresql-contrib
```

Verify the `passwordcheck` module is available:

```bash
ls /usr/lib/postgresql/${PG_VER}/lib/ | grep passwordcheck
```

You should see an output similar to:

```text
passwordcheck.so
```

or (requires more time to search the entire `/usr` directory):

```bash
find /usr -name "passwordcheck.so" 2>/dev/null
```

You should see an output similar to:

```text
/usr/lib/postgresql/18/lib/passwordcheck.so
```

Add it to `shared_preload_libraries`. Since pgaudit was set in Part D, both must be included together:

```bash
sudo -u postgres psql
```

One way of loading libraries in PostgreSQL is via `ALTER SYSTEM`, which writes to `postgresql.auto.conf`:

```bash
# We will NOT use this command because of a known issue with
# comma-separated values in postgresql.auto.conf
# ALTER SYSTEM SET shared_preload_libraries = 'pgaudit, passwordcheck';
```

> **Important:** The comma-separated list must include every module you need. Setting it again replaces the previous value — it does not append. Omitting `pgaudit` here would silently disable audit logging.
>
> A known issue with PostgreSQL is that when `ALTER SYSTEM` writes a value containing a comma to `postgresql.auto.conf`, it wraps it in double quotes to treat it as a single list element — which, unfortunately, corrupts the value for `shared_preload_libraries`.

When you execute :

```bash
sudo vim /var/lib/postgresql/${PG_VER}/main/postgresql.auto.conf
```

You will see content similar to the following if you use
`ALTER SYSTEM SET shared_preload_libraries = 'pgaudit, passwordcheck';` The
the result will be that the value in `postgresql.auto.conf` is wrapped in double quotes, which can cause issues.

```text
# Do not edit this file manually!
# It will be overwritten by the ALTER SYSTEM command.
ssl = 'on'
pgaudit.log = 'write, ddl, role'
log_connections = 'on'
log_disconnections = 'on'
log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '
log_rotation_age = '1d'
log_rotation_size = '100MB'
shared_preload_libraries = '"pgaudit, passwordcheck"'
```

Notice the value for `shared_preload_libraries` in `postgresql.auto.conf` wrapped in double quotes.

Reset the `shared_preload_libraries` value that we set earlier with `pgaudit`.

```bash
sudo -u postgres psql
```

```sql
ALTER SYSTEM RESET shared_preload_libraries;
```

Then edit the `shared_preload_libraries` setting from `postgresql.conf` **NOT** from `postgresql.auto.conf`:

```bash
sudo vim /etc/postgresql/${PG_VER}/main/postgresql.conf
```

Find the line for `shared_preload_libraries` and update it from:

```ini
#shared_preload_libraries = '' # (change requires restart)
```

to

```ini
shared_preload_libraries = 'pgaudit, passwordcheck' # (change requires restart)
```

Restart PostgreSQL:

```bash
sudo systemctl restart postgresql
sudo systemctl status postgresql
```

Verify that the libraries are loaded after successfully restarting PostgreSQL:

```bash
sudo -u postgres psql
```

Execute the following after connecting to psql:

```bash
SHOW shared_preload_libraries;
```

---

### F.2 — Test Password Policy Enforcement

```bash
sudo -u postgres psql -d siwaka_dishes
```

```sql
-- Attempt to create a user with a trivially weak password
CREATE USER test_weak_policy WITH PASSWORD '1234';
-- Expected: ERROR: password is too short

CREATE USER test_weak_policy WITH PASSWORD 'onlylowercase';
-- Expected: ERROR: password must contain both letters and nonletters

-- A compliant password: long enough, mixed characters
CREATE USER test_strong_policy WITH PASSWORD 'Nairobi@2025!';
-- Expected: CREATE ROLE

-- Clean up the test users
DROP USER test_weak_policy;
DROP USER test_strong_policy;
\q
```

---

### F.3 — Set Password Expiry on Existing Users

The DPA 2019 does not prescribe a password rotation interval, but regular rotation limits the window of exposure if a credential is compromised. Set expiry dates on all accounts created in this lab:

```bash
sudo -u postgres psql -d siwaka_dishes
```

```sql
ALTER ROLE auditor_atwoli VALID UNTIL '2026-10-31';
ALTER ROLE siwaka_dishes_app_runtime_ass_mgr_nbi VALID UNTIL '2026-12-31';
ALTER ROLE siwaka_dishes_app_runtime_ass_mgr_mba VALID UNTIL '2026-12-31';
ALTER ROLE siwaka_dishes_app_runtime_mgr VALID UNTIL '2026-12-31';

ALTER ROLE siwaka_dishes_analytics VALID UNTIL '2026-12-31';
ALTER ROLE siwaka_dishes_app_runtime VALID UNTIL '2026-12-31';
ALTER ROLE siwaka_dishes_backup VALID UNTIL '2026-12-31';
ALTER ROLE siwaka_dishes_db_admin VALID UNTIL '2026-12-31';

-- Verify all expiry dates are set
SELECT rolname, rolvaliduntil
FROM pg_roles
WHERE rolname IN (
  'auditor_atwoli',
  'siwaka_dishes_app_runtime_ass_mgr_nbi',
  'siwaka_dishes_app_runtime_ass_mgr_mba',
  'siwaka_dishes_app_runtime_mgr',
  'siwaka_dishes_analytics',
  'siwaka_dishes_app_runtime',
  'siwaka_dishes_backup',
  'siwaka_dishes_db_admin'
)
ORDER BY rolname;

\q
```

When a user's password expires, they see:

```text
FATAL: password authentication failed for user "auditor_atwoli"
DETAIL: User "auditor_atwoli" does not have a valid password.
```

Then the database administrator resets it:

```sql
ALTER ROLE auditor_atwoli
  WITH ENCRYPTED PASSWORD 'NewAtwoli@2027!'
  VALID UNTIL '2027-08-31';
```

---

## Part G — Data Retention Policy

**Legal basis:** s.39(1) DPA 2019 states that a data controller "shall retain personal data only as long as may be reasonably necessary to satisfy the purpose for which it is processed." s.39(2) requires deletion, erasure, anonymisation or pseudonymisation at the end of the retention period. GDPR Art. 5(1)(e) is equivalent: "storage limitation."

**Important:** A retention policy is first a governance document, then a technical implementation. You cannot implement retention correctly without first deciding what the policy is. The steps below implement both.

---

### G.1 — Document the Retention Policy

Create a formal retention policy table within the database. This serves as an auditable record of the organisation's retention decisions:

```bash
sudo -u postgres psql -d siwaka_dishes
```

```sql
CREATE TABLE IF NOT EXISTS data_retention_policy (
    table_name          TEXT        NOT NULL,
    personal_data       BOOLEAN     NOT NULL,
    sensitive_data      BOOLEAN     NOT NULL DEFAULT FALSE,
    retention_days      INTEGER     NOT NULL,
    legal_basis         TEXT        NOT NULL,
    retention_action    TEXT        NOT NULL CHECK (
                          retention_action IN ('DELETE', 'ANONYMISE', 'ARCHIVE')
                        ),
    policy_owner        TEXT        NOT NULL,
    last_reviewed       DATE        NOT NULL DEFAULT CURRENT_DATE,
    PRIMARY KEY (table_name)
);

INSERT INTO data_retention_policy
VALUES ('customer', TRUE, FALSE, 1095,
        'DPA 2019 s.39 — retain for 3 years after last order; longer if active customer',
        'ANONYMISE', 'Data Protection Officer', CURRENT_DATE),

       ('customer_order', TRUE, FALSE, 2555,
        'Kenya Revenue Authority — 7-year requirement for financial records',
        'ARCHIVE', 'Data Protection Officer', CURRENT_DATE),

       ('order_detail', FALSE, FALSE, 2555,
        'Same as customer_order — financial records',
        'ARCHIVE', 'Data Protection Officer', CURRENT_DATE),

       ('employee', TRUE, FALSE, 2555,
        'Employment Act 2007 — 7 years after termination',
        'ARCHIVE', 'Data Protection Officer', CURRENT_DATE),

       ('branch', FALSE, FALSE, 9999,
        'Operational data — retained indefinitely while business operates',
        'ARCHIVE', 'DBA', CURRENT_DATE);

ALTER TABLE data_retention_policy
OWNER TO siwaka_dishes_db_admin;

SELECT * FROM data_retention_policy;

\q
```

---

### G.2 — Implement a Retention Procedure

Create a stored procedure that a DBA or scheduled job can execute to apply the retention policy. Rather than deleting customer records outright — which may violate other legal obligations such as the requirement to retain financial records — this procedure **anonymizes** expired records by replacing personal identifiers with irreversible placeholders.

```bash
sudo -u postgres psql -d siwaka_dishes
```

```sql
CREATE OR REPLACE PROCEDURE apply_customer_retention_policy()
LANGUAGE plpgsql
AS $$
DECLARE
  v_retention_days INTEGER;
  v_cutoff_date    DATE;
  v_rows_affected  INTEGER;
BEGIN
  -- Retrieve the configured retention period from the policy table
  SELECT retention_days INTO v_retention_days
  FROM data_retention_policy
  WHERE table_name = 'customer';

  v_cutoff_date := CURRENT_DATE - v_retention_days;

  -- Anonymize customers whose last order predates the retention cutoff
  -- and who have no orders after the cutoff date.
  -- customer_number is the correct primary key column name in this schema.
  UPDATE customer
  SET
    customer_name       = 'ANONYMIZED',
    contact_first_name  = 'ANONYMIZED',
    contact_last_name   = 'ANONYMIZED',
    phone               = 'ANONYMIZED'
  WHERE
    customer_number NOT IN (
      SELECT DISTINCT customer_number
      FROM customer_order
      WHERE order_date > v_cutoff_date
    )
    AND contact_first_name != 'ANONYMIZED';
    -- Avoids re-processing records that have already been anonymized

  GET DIAGNOSTICS v_rows_affected = ROW_COUNT;

  RAISE NOTICE 'Retention policy applied: % customer record(s) anonymized. Cutoff date: %',
    v_rows_affected, v_cutoff_date;
END;
$$;

-- Test the procedure (reports 0 rows on fresh data — this is correct)
CALL apply_customer_retention_policy();

\q
```

> **In production**, this procedure would be scheduled via `pg_cron` or a system cron job to run monthly. The execution and its output would be captured in the audit log (pgaudit records the procedure call) and the results stored in an operational log table. The anonymization is irreversible by design — once applied, the personal identifiers cannot be recovered, which is the intent of s.39(2) DPA 2019.

---

### G.3 — Scheduling, Testing, and Verifying the Retention Policy

---

#### Part 1: Schedule the Procedure via `pg_cron`

`pg_cron` is a background worker that runs scheduled SQL jobs inside PostgreSQL itself.
It should ship with `postgresql-contrib` by default, however, you can still install it explicitly:

```bash
sudo apt install -y postgresql-${PG_VER}-cron
```

##### Step 1 — Add `pg_cron` to `shared_preload_libraries`

You already edited `/etc/postgresql/18/main/postgresql.conf` in the previous section
to add `passwordcheck`. Extend that same line to include `pg_cron`:

```bash
sudo vim /etc/postgresql/${PG_VER}/main/postgresql.conf
```

Find and update the line:

```ini
shared_preload_libraries = 'pgaudit, passwordcheck, pg_cron' # (change requires restart)
```

##### Step 2 — Tell `pg_cron` which database to use

In the same `postgresql.conf` file, add the following line (anywhere after the
`shared_preload_libraries` setting):

```ini
cron.database_name = 'siwaka_dishes'
```

This directs `pg_cron` to store its job metadata inside `siwaka_dishes` and to
execute jobs there by default.

##### Step 3 — Restart PostgreSQL

```bash
sudo systemctl restart postgresql
```

Confirm the server came back up cleanly:

```bash
sudo systemctl status postgresql
```

##### Step 4 — Create the `pg_cron` Extension

Connect to `siwaka_dishes` as a superuser and create the extension:

```bash
sudo -u postgres psql -d siwaka_dishes
```

```sql
CREATE EXTENSION IF NOT EXISTS pg_cron;
```

Verify it loaded:

```sql
SHOW shared_preload_libraries;
```

or

```sql
SELECT name, default_version, installed_version
FROM pg_available_extensions
-- WHERE name = 'pg_cron';
ORDER BY name ASC;
```

##### Step 5 — Schedule the Retention Procedure

Schedule the procedure to run at **03:00 on the first day of every month**:

```sql
SELECT cron.schedule(
    'monthly-customer-retention',          -- job name
    '0 3 1 * *',                           -- cron expression: 03:00 on day 1 of every month
    'CALL apply_customer_retention_policy()'
);
```

##### Step 6 — Verify the Scheduled Job

```sql
SELECT
    jobid,
    jobname,
    schedule,
    command,
    active
FROM cron.job;
```

You should see one row with the job name `monthly-customer-retention` and
`active = true`.

```text
 jobid |          jobname           | schedule  |                command                 | active
-------+----------------------------+-----------+----------------------------------------+--------
     1 | monthly-customer-retention | 0 3 1 * * | CALL apply_customer_retention_policy() | t
```

> **To unschedule the job** if ever needed:
>
> ```sql
> SELECT cron.unschedule('monthly-customer-retention');
> ```

---

#### Part 2: Create a Test Customer

Connect to the database:

```bash
sudo -u postgres psql -d siwaka_dishes
```

Insert a test customer whose personal details you will later confirm are anonymized:

```sql
INSERT INTO customer (
    customer_name,
    contact_first_name,
    contact_last_name,
    phone,
    address_line1,
    postal_code,
    county,
    sub_county,
    status
)
VALUES (
    '[Business] Nyaga Holdings Ltd',
    'Kizito',
    'Nyaga',
    '+254700000001',
    '14 Moi Avenue',
    '00100',
    'Nairobi',
    'Westlands',
    1
);
```

Confirm the record was inserted:

```sql
SELECT
    customer_number,
    customer_name,
    contact_first_name,
    contact_last_name,
    phone
FROM customer
ORDER BY customer_number DESC
LIMIT 1;
```

Expected output:

```text
 customer_number |         customer_name         | contact_first_name | contact_last_name |     phone
-----------------+-------------------------------+--------------------+-------------------+---------------
             310 | [Business] Nyaga Holdings Ltd | Kizito             | Nyaga             | +254700000001
```

Store the `customer_number` for later reference.

```sql
SELECT customer_number
FROM customer
ORDER BY customer_number DESC
LIMIT 1
\gset
```

Confirm that the `customer_number` is stored in the `:customer_number` variable:

```sql
SELECT :customer_number;
```

---

#### Part 3: Create a Past Order Using a Database Transaction

The retention procedure anonymizes customers whose **most recent order predates
the configured retention cutoff**. You must, therefore, create an order dated
further in the past than the retention window.

The transaction below demonstrates four key control-flow constructs:

| Construct | Purpose |
| --- | --- |
| `BEGIN` | Starts an explicit transaction block |
| `SAVEPOINT` | Marks a point you can roll back to without aborting the whole transaction |
| `ROLLBACK TO SAVEPOINT` | Undoes work back to the named savepoint, leaving earlier work intact |
| `COMMIT` | Makes all remaining changes permanent |

Connect to the database if you are not already:

```bash
sudo -u postgres psql -d siwaka_dishes
```

Execute the following block **exactly as written** — the scenario intentionally
makes a mistake and corrects it using a savepoint:

```sql
BEGIN;

INSERT INTO customer_order (
    order_date,
    required_date,
    order_status_id,
    customer_number,
    branch_code
)
VALUES (
    CURRENT_DATE - INTERVAL '5 years',
    CURRENT_DATE - INTERVAL '5 years' + INTERVAL '7 days',
    2,
    :customer_number,
    1
);

SAVEPOINT sp_order_placed;

UPDATE customer_order
SET required_date = CURRENT_DATE + INTERVAL '365 days'
WHERE order_number = (
    SELECT MAX(order_number)
    FROM customer_order
);

ROLLBACK TO SAVEPOINT sp_order_placed;

COMMIT;
```

Verify the order was saved without the erroneous order date:

```sql
SELECT
    order_number,
    order_date,
    required_date,
    order_status_id,
    customer_number
FROM customer_order
WHERE order_number = (
    SELECT MAX(order_number)
    FROM customer_order
);
```

Expected output (similar to **NOT** identical to):

```text
 order_number |     order_date      |    required_date    | order_status_id | customer_number 
--------------+---------------------+---------------------+-----------------+-----------------
         2506 | 2021-06-07 00:00:00 | 2021-06-14 00:00:00 |               2 |             302
```

> The `required_date` column is `CURRENT_DATE - INTERVAL '5 years' + INTERVAL '7 days',`  
> and NOT `CURRENT_DATE + INTERVAL '365 days'` — proof that `ROLLBACK TO SAVEPOINT`  
> undid the `UPDATE` while the `COMMIT` preserved the `INSERT`.

---

#### Part 4: Apply the Retention Policy and Confirm Anonymization

##### Step 1 — Check the configured retention period

```sql
SELECT table_name, retention_days
FROM data_retention_policy
WHERE table_name = 'customer';
```

The order you inserted is five years old, so it will fall outside any retention
window of 3 years or fewer.

##### Step 2 — Call the procedure

Confirm the customer data before anonymization:

```sql
SELECT
    customer_number,
    customer_name,
    contact_first_name,
    contact_last_name,
    phone
FROM customer
WHERE customer_number = :customer_number;
```

Expected output:

```text
  customer_number |         customer_name         | contact_first_name | contact_last_name |     phone     
-----------------+-------------------------------+--------------------+-------------------+---------------
             302 | [Business] Nyaga Holdings Ltd | Kizito             | Nyaga             | +254700000001
```

Call the procedure to apply the retention policy. **NOTE:** this is the same procedure that `pg_cron` will execute on schedule, so you are effectively testing the scheduled job manually here.

```sql
CALL apply_customer_retention_policy();
```

You should see a `NOTICE` similar to:

```text
NOTICE:  Retention policy applied: 1 customer record(s) anonymized. Cutoff date: 2023-06-08
```

##### Step 3 — Confirm the customer record has been anonymized

```sql
SELECT
    customer_number,
    customer_name,
    contact_first_name,
    contact_last_name,
    phone
FROM customer
WHERE customer_number = :customer_number;
```

Expected output:

```text
 customer_number | customer_name | contact_first_name | contact_last_name |   phone
-----------------+---------------+--------------------+-------------------+------------
             310 | ANONYMIZED    | ANONYMIZED         | ANONYMIZED        | ANONYMIZED
```

All four personal identifier columns have been replaced with the irreversible
placeholder `ANONYMIZED`. The row itself is retained (preserving referential
integrity with the order history and satisfying financial record-keeping
obligations), but no personal data remains recoverable as per the retention policy.

Note that the retention policy did not specify that `address_line1`, `postal_code`, `county`, `sub_county`, or `status` should be anonymized, so those columns remain unchanged. This is intentional. A policy can target only direct personal identifiers, not all data in the table.

We can confirm this:

```sql
SELECT *
FROM customer
WHERE customer_number = :customer_number;
```

##### Step 4 — Confirm the procedure is idempotent

**Idempotent** means that running the same procedure multiple times has the same effect as running it once. In this case, once a record is anonymised, subsequent runs of the procedure should not change it further.

Call the procedure a second time:

```sql
CALL apply_customer_retention_policy();
```

Expected output:

```text
NOTICE:  Retention policy applied: 0 customer record(s) anonymized. Cutoff date: 2023-06-08
```

Zero rows are affected because the `WHERE` clause filters out records where
`contact_first_name != 'ANONYMIZED'` — records already anonymized are skipped
on every subsequent run. This is the correct behaviour.

---

#### Summary of G.3

| Step | What was demonstrated |
| --- | --- |
| `pg_cron` setup | Scheduling a stored procedure to run automatically inside PostgreSQL |
| Test customer | Inserting representative personal data to test against |
| Transaction with savepoint | `BEGIN`, `SAVEPOINT`, `ROLLBACK TO SAVEPOINT`, `COMMIT` working together |
| Retention procedure | Anonymization of expired records, idempotency of repeated execution |

> **Audit note:** because `pgaudit` is active, every `CALL apply_customer_retention_policy()`
> — whether triggered manually or by `pg_cron` — is recorded in the PostgreSQL
> log with the calling user, timestamp, and database name, providing a full
> audit trail for compliance purposes.

---

## Part H — Backup Security

**Legal basis:** s.41(4)(d) DPA 2019 requires "the ability to restore the availability and access to personal data in a timely manner in the event of a physical or technical incident." The backup itself contains all the personal data in the database — an unencrypted backup file is as dangerous as an unencrypted database. GDPR Art. 32(1)(c) is equivalent.

---

### H.1 — Verify and Complete Backup User Permissions

Before running `pg_dump`, confirm that `siwaka_dishes_backup` has `SELECT` on every table currently in the schema — including any created after the initial DDL script was executed.

**Why this step is necessary:** The `ALTER DEFAULT PRIVILEGES FOR ROLE siwaka_dishes_db_admin` statement in the DDL script grants `SELECT` automatically to `siwaka_dishes_backup` for tables created by `siwaka_dishes_db_admin`. However, the `data_retention_policy` table was created in Part G while connected as `postgres`, not as `siwaka_dishes_db_admin`. PostgreSQL's default privileges are role-specific — they do not apply when a different role creates the object. The backup user was therefore silently excluded from that table.

The reliable fix is to re-run `GRANT SELECT ON ALL TABLES` as a catch-all before every backup run. This is also good operational practice: it ensures that any table added by any role — not just `siwaka_dishes_db_admin` — is covered.

```bash
sudo -u postgres psql -d siwaka_dishes
```

```sql
-- Re-grant SELECT on all current tables to backup and analytics users.
-- This is safe to run multiple times — granting an existing privilege is not an error.
GRANT SELECT ON ALL TABLES IN SCHEMA public TO siwaka_dishes_backup;

GRANT
    USAGE,
    SELECT
ON ALL SEQUENCES IN SCHEMA public
TO siwaka_dishes_backup;

-- The backup user also needs to bypass RLS policies to read all rows. This is done by granting the BYPASSRLS attribute, which allows the user to see all data regardless of any row-level security policies in place. This is essential for a backup user to ensure that the backup captures the complete dataset without being restricted by RLS rules.

ALTER USER siwaka_dishes_backup BYPASSRLS;

-- Verify the backup user can see both a table and a sequence
SET ROLE siwaka_dishes_backup;
SELECT COUNT(*) FROM data_retention_policy;
SELECT COUNT(*) FROM customer;
SELECT last_value FROM branch_branch_code_seq;
-- Expected: both return results with no permission error

RESET ROLE;
\q
```

> **Root cause note for future reference:** Any table created outside of `siwaka_dishes_db_admin` — whether created as `postgres`, `student`, or any other role — will not inherit the default privileges defined in the DDL script. The safest habit is to always create tables while connected as `siwaka_dishes_db_admin`. When that is not possible (as with the retention policy table created during a lab exercise), re-running `GRANT SELECT ON ALL TABLES` corrects the gap. Add this re-grant to your backup script so that it runs automatically before every dump.

---

### H.2 — Create and Encrypt a Database Backup

`gpg` (GNU Privacy Guard) is pre-installed on Ubuntu. Symmetric encryption with AES-256 requires only a passphrase — no PKI infrastructure is needed for this lab. PKI (Public Key Infrastructure) is a more complex encryption method that uses a pair of keys — public and private — for encryption and decryption. Symmetric encryption uses a single passphrase for both operations. It is simpler to implement while still providing strong security when a strong passphrase is used.

```bash
# Create the backup directory with restricted permissions
mkdir -p ~/backups
chmod 700 ~/backups
```

Run the dump as `siwaka_dishes_backup`. This is the correct user for backups — it has read-only access and cannot modify the database even if the backup process is compromised.

We exclude the `cron` schema because it contains only metadata for scheduled jobs, not application data. Excluding it reduces the backup size and eliminates unnecessary sensitive information about job schedules and configurations.

```bash
pg_dump \
  -U siwaka_dishes_backup \
  -W \
  -h 127.0.0.1 \
  --exclude-schema=cron \
  siwaka_dishes \
  > ~/backups/siwaka_dishes_$(date +%Y%m%d).sql
```

Flag reference:

| Flag | Meaning |
| ------ | --------- |
| `-U siwaka_dishes_backup` | Connect as the dedicated backup role, not the superuser |
| `-W` | Prompt for the password explicitly |
| `-h 127.0.0.1` | Connect via TCP on the loopback interface. Using `127.0.0.1` explicitly ensures TCP is used rather than a Unix socket, which is consistent with how pg_hba.conf is configured for password authentication |
| `siwaka_dishes` | The database to dump |

Verify the unencrypted dump was created and note its size:

```bash
ls -alh ~/backups/
```

Encrypt the dump with AES-256 symmetric encryption. AES-256 stands for **Advanced Encryption Standard using a 256-bit encryption key**.

```bash
gpg --symmetric \
    --cipher-algo AES256 \
    --output ~/backups/siwaka_dishes_$(date +%Y%m%d).sql.gpg \
    ~/backups/siwaka_dishes_$(date +%Y%m%d).sql
```

You will be prompted for a passphrase. Choose a strong one and store it in a secure password manager — the backup cannot be decrypted without it.

Securely delete the unencrypted dump. `shred` overwrites the file's data blocks before unlinking it, preventing recovery by filesystem forensics tools:

```bash
shred -vfz ~/backups/siwaka_dishes_$(date +%Y%m%d).sql && \
  rm ~/backups/siwaka_dishes_$(date +%Y%m%d).sql
```

Verify only the encrypted file remains:

```bash
ls -alh ~/backups/
```

---

### H.3 — Verify the Encrypted Backup Can Be Restored

An unverified backup is not a backup — it is a "hope" 🙂.  
**Always test restoration before declaring the backup process complete**.

```bash
# Decrypt the backup
gpg --decrypt \
    ~/backups/siwaka_dishes_$(date +%Y%m%d).sql.gpg \
    > /tmp/siwaka_dishes_restore_test.sql

# Create a clean test database to restore into
sudo -u postgres psql -c "CREATE DATABASE siwaka_dishes_restore_test;"

# Restore the dump into the test database
psql -U postgres -h 127.0.0.1 -d siwaka_dishes_restore_test \
  < /tmp/siwaka_dishes_restore_test.sql

# Verify the restoration succeeded
sudo -u postgres psql -d siwaka_dishes_restore_test \
  -c "SELECT COUNT(*) FROM customer;"

# Expected: a non-zero count matching the original database
# This should be 301 without Row-Level Security.

# Clean up (Tear Down) — drop the test database and shred the decrypted file
sudo -u postgres psql -c "DROP DATABASE siwaka_dishes_restore_test;"

shred -vfz /tmp/siwaka_dishes_restore_test.sql && \
  rm /tmp/siwaka_dishes_restore_test.sql
```

If the count matches the count in the production database, the backup is verified:

```bash
# Compare against the production database
sudo -u postgres psql -d siwaka_dishes \
  -c "SELECT COUNT(*) FROM customer;"
```

Both counts must be identical.

---

## Compliance Summary Checklist

Use this checklist to verify all controls are in place before submission. This is a similar checklist that you would use in an audit or compliance review — it maps each control to the specific legal requirement it addresses, and provides the exact command to verify it in PostgreSQL.

| # | Control | Command to Verify | DPA 2019 | GDPR |
| --- | --------- | ------------------- | ---------- | ------ |
| 1 | SSL enabled | `sudo -u postgres psql -c "SHOW ssl;"` → `on` | s.41(4)(c), s.42(1)(c) | Art. 32 |
| 2 | SSL enforced for remote connections | Check `pg_hba.conf` for `hostssl` on the `192.168.56.0/24` line | s.41(4)(c) | Art. 32 |
| 3 | Connection uses TLSv1.3 | `\conninfo` shows `protocol: TLSv1.3` | s.42 | Art. 32 |
| 4 | PUBLIC schema creation revoked | `SELECT has_schema_privilege('public','public','CREATE');` → `false` | s.25(d), s.41 | Art. 25 |
| 5 | Tier-based roles created | `\du` shows `siwaka_dishes_db_admin`, `siwaka_dishes_app_runtime`, `siwaka_dishes_analytics`, `siwaka_dishes_backup` | s.25(d) | Art. 5(1)(c) |
| 6 | Superuser rejected from remote | Attempt `psql -h 192.168.56.x -U postgres` → rejected | s.41 | Art. 32 |
| 7 | pgAudit installed and active | `SELECT extname FROM pg_extension WHERE extname = 'pgaudit';` | s.43(1)(a), s.43(8) | Art. 33 |
| 8 | Write/DDL/role operations logged | Perform an INSERT and check the PostgreSQL log | s.43 | Art. 33 |
| 9 | Row-level security on customer_order | `SELECT relrowsecurity FROM pg_class WHERE relname = 'customer_order';` → `true` | s.25(d), s.44–47 | Art. 5(1)(c) |
| 10 | RLS policies correctly defined | `SELECT policyname, roles, cmd FROM pg_policies WHERE tablename = 'customer_order';` | s.25(d), s.44–47 | Art. 5(1)(c) |
| 11 | RLS on customer table | `SELECT relrowsecurity FROM pg_class WHERE relname = 'customer';` → `true` | s.25(d), s.44–47 | Art. 5(1)(c) |
| 12 | passwordcheck enforced | Attempt `CREATE USER x WITH PASSWORD '123';` → error | s.41(4)(b) | Art. 32 |
| 13 | Password expiry set | `SELECT rolname, rolvaliduntil FROM pg_roles WHERE rolvaliduntil IS NOT NULL;` | s.41 | Art. 32 |
| 14 | Retention policy documented | `SELECT * FROM data_retention_policy;` | s.39 | Art. 5(1)(e) |
| 15 | Anonymisation procedure exists | `\df apply_customer_retention_policy` | s.39(2) | Art. 5(1)(e) |
| 16 | Encrypted backup created | `ls -lh ~/backups/*.gpg` | s.41(4)(d) | Art. 32(1)(c) |
| 17 | Backup restoration verified | Restore test completed and `SELECT COUNT(*) FROM customer_order;` returns a non-zero result | s.41(4)(d) | Art. 32(1)(c) |

---

## Lab Deliverables

Compile the following into a single PDF report and submit via the course portal:

| # | Deliverable | What to Include |
| --- | ------------- | ---------------- |
| 1 | **TLS Verification** | Screenshot of `\conninfo` showing `TLSv1.3` inside psql. Screenshot of the failed `sslmode=disable` connection attempt. |
| 2 | **Role Structure** | Screenshot of `\du` in psql showing all tier-based roles and the named human users created in this lab. |
| 3 | **Permission Testing** | Screenshot of `auditor_atwoli` failing to DELETE from the `customer` table. Screenshot of `siwaka_dishes_app_runtime` failing to CREATE a new table. |
| 4 | **Superuser Rejection** | Screenshot of the failed remote connection attempt as `postgres` from your laptop's terminal. |
| 5 | **Audit Log Entry** | Screenshot of a pgAudit log entry for an INSERT operation, showing the full log format including timestamp, user, database, and SQL statement. |
| 6 | **Row-Level Security** | Screenshot showing `customer_order` count as `0` without branch context, and a non-zero count after `SET app.current_branch = '[value]';`. |
| 7 | **Password Policy** | Screenshot of the error when attempting to create a user with a weak password. |
| 8 | **Compliance Checklist** | A completed version of the checklist table above, with the actual terminal output for each verification command pasted in. |
| 9 | **Written Reflection (500 words)** | Answer: *"Your organisation has discovered that an employee used their legitimate database credentials to export the customer table at 11pm on a Sunday. Walk through exactly how the controls implemented in this lab would: (a) detect the incident, (b) limit the data exposed, and (c) enable you to comply with Section 43 of the DPA 2019."* |

---

## Troubleshooting

| Symptom | Likely Cause | Resolution |
| --------- | ------------- | ------------ |
| `SSL SYSCALL error: EOF detected` when connecting | Server certificate permission error | Confirm `server.key` is `600` and owned by `postgres`. Check PostgreSQL logs: `sudo tail /var/log/postgresql/postgresql-${PG_VER}-main.log` |
| pgAdmin fails to connect after enabling `hostssl` | pgAdmin not negotiating SSL | In pgAdmin: right-click server → Properties → SSL → set **SSL mode** to `require`. Click Save. |
| `FATAL: no pg_hba.conf entry ... no encryption` | `hostssl` enforced but client not using SSL | Add `sslmode=require` to psql connection string or set pgAdmin SSL mode to `require` |
| `ERROR: permission denied for table ...` when testing roles | Expected — the test is working correctly | Confirm the error message references the table you expected to be blocked. |
| RLS returning 0 rows even after SET command | `branch_code` is INT but the SET value does not cast correctly | Confirm you are setting a numeric string, e.g., `SET app.current_branch = '1';` not `SET app.current_branch = 'NBI';`. Run `SELECT branch_code FROM branch;` to find valid integer values. |
| RLS returning 0 rows unexpectedly | `app.current_branch` not set in session | Run `SET app.current_branch = '[integer]';` before querying. The setting resets at the end of each session. |
| `ERROR: invalid input syntax for type integer` in RLS | The `::INT` cast failed on the value | The value set in `app.current_branch` is not a valid integer string. Confirm with `SHOW app.current_branch;`. |
| pgaudit entries not appearing in log | Extension not created in this database | Run `CREATE EXTENSION IF NOT EXISTS pgaudit;` while connected to `siwaka_dishes`. Confirm `shared_preload_libraries` includes `pgaudit` and PostgreSQL was restarted. |
| `ALTER SYSTEM` settings not taking effect | PostgreSQL not restarted after changing `shared_preload_libraries` | `shared_preload_libraries` requires a **restart**, not just a reload. Run `sudo systemctl restart postgresql`. |
| `FATAL: password authentication failed` for expired user | Password has passed its `VALID UNTIL` date | An administrator runs: `ALTER ROLE username WITH ENCRYPTED PASSWORD 'NewPass!' VALID UNTIL 'date';` |
| `gpg: no valid OpenPGP data found` when decrypting | File corrupted or wrong encryption method used | Re-encrypt the backup using `--symmetric`. Ensure the output filename ends in `.gpg`. |
| passwordcheck not enforcing rules | Module not loaded or PostgreSQL not restarted | Check `SHOW shared_preload_libraries;` — it must contain `passwordcheck`. Restart if the value is missing. |
| `ERROR: unrecognized configuration parameter "pgaudit.log"` | pgaudit not in `shared_preload_libraries` | Set `shared_preload_libraries = 'pgaudit,passwordcheck'` and restart PostgreSQL. |
| Procedure errors: column does not exist | Old column names used from a different schema | Confirm column names with `\d customer` in psql. This schema uses `customer_number`, `contact_first_name`, `contact_last_name`, and `phone`. |

---

## Further Reading

These resources are recommended for students who wish to go beyond the lab:

- **Kenya Data Protection Act 2019 (full text):** [https://new.kenyalaw.org/akn/ke/act/2019/24/](https://new.kenyalaw.org/akn/ke/act/2019/24/) — No. 24 of 2019
- **EU General Data Protection Regulation (GDPR)** (download link: [https://gdpr-info.eu](https://gdpr-info.eu) or for the original PDF: [https://eur-lex.europa.eu/legal-content/EN/TXT/PDF/?uri=CELEX:32016R0679&from=EN](https://eur-lex.europa.eu/legal-content/EN/TXT/PDF/?uri=CELEX:32016R0679&from=EN) )
- **Office of the Data Protection Commissioner (Kenya):** [https://www.odpc.go.ke](https://www.odpc.go.ke)
- **PostgreSQL Documentation:** [https://www.postgresql.org/docs/current/](https://www.postgresql.org/docs/current/)
- **pgAudit Documentation:** [https://www.pgaudit.org](https://www.pgaudit.org)
- **CIS PostgreSQL Benchmark:** The Center for Internet Security publishes a free hardening guide for PostgreSQL. It is the most comprehensive checklist available. [https://www.cisecurity.org/benchmark/postgresql/](https://www.cisecurity.org/benchmark/postgresql/)

---
