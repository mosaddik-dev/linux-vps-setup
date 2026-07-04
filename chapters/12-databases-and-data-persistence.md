# Chapter 12 — Databases & Data Persistence

> *Part III · Running Web Applications — Chapter 12 of 18*

Your app is served securely (Chapter 11) and runs reliably (Chapter 10) — but right now it forgets everything the moment it restarts. Real applications must *remember*: users, posts, orders, sessions, inventory. That memory is a **database**, and adding one introduces a genuinely new set of concerns. A database holds your most valuable and most sensitive asset — the data itself — so *where* it lives, *how* it runs, *who* can reach it, and *how* your app authenticates to it all matter enormously. Get it wrong and you get the internet's most common catastrophe: an exposed database with weak credentials, harvested by bots within hours. This chapter teaches you to choose a database, run it as a hardened, localhost-only service behind everything you've built, create databases and users with least privilege, and connect your application to it *safely*.

---

## Goal

By the end of this chapter you will:

1. Understand what a **database** is, what **persistence** means, and where data physically lives.
2. Understand **relational (SQL)** vs **non-relational (NoSQL)** databases, and where **SQLite** fits.
3. Choose a database for a typical web app and justify it (we'll use **PostgreSQL** as the worked example).
4. Install the database and run it as a hardened **systemd** service **bound to localhost**.
5. Create a **database**, a **least-privilege user**, and grant only the needed rights.
6. Connect your **application** to the database *safely* — secrets in environment/config, never hard-coded.
7. Understand the security threats specific to databases and how everything from Chapters 3–11 protects them.

---

## Background

### What is a database? What is persistence?

A **database** is an organized store of data plus the software (a **database management system**, DBMS) that reads, writes, queries, and protects it. **Persistence** means the data survives beyond the life of any single process — it's written to disk and remains after the app restarts, the DBMS restarts, or the server reboots.

Contrast with what you have now: your app keeps data in **memory (RAM)**, which is *volatile* — gone on restart. A database writes to **disk** (a file or set of files the DBMS manages) so the data is *durable*. When your app needs to remember something, it hands it to the DBMS; when it needs it back, it asks the DBMS. The DBMS also handles concurrency (many requests at once), integrity (no half-written records), and querying (finding data efficiently).

### Relational (SQL) vs non-relational (NoSQL)

The first big fork is the **data model**:

| | **Relational (SQL)** | **Non-relational (NoSQL)** |
|---|---|---|
| **Shape** | Data in **tables** (rows & columns) with a defined **schema**; tables relate to each other via keys. | Flexible: documents (JSON-like), key-value pairs, graphs, or wide-columns. |
| **Language** | **SQL** (Structured Query Language) — a mature, standardized query language. | Varies by product (document queries, key lookups, etc.). |
| **Strengths** | Strong consistency, relationships, complex queries/joins, transactions (**ACID**), decades of tooling. | Flexible/evolving schemas, certain scaling patterns, some specialized shapes (graphs, huge write volumes). |
| **Examples** | **PostgreSQL, MySQL/MariaDB, SQLite** | **MongoDB** (document), **Redis** (key-value/cache), **Cassandra** (wide-column), **Neo4j** (graph) |
| **Best for** | The vast majority of web apps: users, orders, relationships, anything needing correctness. | Specific needs: caching (Redis), flexible documents, extreme scale/shape. |

> 🧠 **ACID** — the four guarantees that make relational databases trustworthy for important data: **A**tomicity (a transaction fully happens or not at all), **C**onsistency (data stays valid), **I**solation (concurrent transactions don't corrupt each other), **D**urability (committed data survives crashes). When money, accounts, or relationships are involved, you want ACID.

**Default guidance:** for a typical web application, start with a **relational** database unless you have a specific reason not to. It gives you correctness, flexibility to query in ways you didn't anticipate, and the largest body of knowledge/tooling. NoSQL solves real problems, but "we might need to scale someday" is rarely one you have *today* — and a relational DB scales further than beginners think.

### Where SQLite fits (and why it's not our main choice here)

**SQLite** is a relational database that is just a **single file** with no separate server process — your app opens the file directly. It's brilliant for: local development, small/low-traffic sites, embedded use, and single-process apps. Its limits: it's not built for many processes writing concurrently across a network, and it doesn't fit a multi-app or multi-server future well.

| | **SQLite** | **Client-server DB (PostgreSQL/MySQL)** |
|---|---|---|
| Runs as | A library reading a file — no daemon | A separate **service** apps connect to over a socket/port |
| Setup | Zero (just a file) | Install, configure, secure, manage a service |
| Concurrency | Limited multi-writer | Built for many concurrent clients |
| Scaling | Single process/host | Multiple apps/servers, replication |
| Best for | Dev, small/simple, embedded | Production web apps, growth, multi-client |

We'll teach a **client-server** database because that's the production-shaped skill (running and securing a DB *service*), and note SQLite as the right tool for simpler cases.

### PostgreSQL vs MySQL/MariaDB (our worked example: PostgreSQL)

The two dominant open-source relational servers:

| | **PostgreSQL** | **MySQL / MariaDB** |
|---|---|---|
| Reputation | Standards-compliant, feature-rich, strong correctness, advanced types (JSON, arrays, full-text), extensible. | Extremely popular, simple, fast for read-heavy workloads, huge shared-hosting presence. |
| Auth model | Roles; peer/ident + password auth; per-DB privileges. | Users defined by user@host; grants. |
| Verdict | ✅ Our example — a superb general-purpose default. | ✅ Also excellent; MariaDB is the community fork of MySQL. |

Both are great. We use **PostgreSQL** for the worked example because it's a robust, correctness-oriented default that scales well and teaches the concepts cleanly; the *principles* (localhost binding, least-privilege users, secrets in config, backups) are identical for MySQL/MariaDB.

### The golden security rule for databases: never expose them

This is the single most important idea in the chapter. **A database should almost never be reachable from the internet.** Your *app* talks to the database; the *internet* talks to your app (through Nginx). So:

```mermaid
flowchart LR
    NET["🌐 Internet"] -- "443 (HTTPS)" --> NGINX["Nginx (Ch.11)"]
    NGINX -- "127.0.0.1:3000" --> APP["Your app (Ch.10)"]
    APP -- "127.0.0.1:5432" --> DB["PostgreSQL<br/>bound to localhost ONLY"]
    NET -. "❌ no direct path" .-> DB
```

The database listens on **`127.0.0.1`** (localhost) only — exactly the binding principle from Chapter 6. The firewall never opens the DB port (5432/3306). This means even if someone found your database port, the network won't route to it from outside. Combined with a strong DB password and least-privilege users, your data has multiple independent walls around it.

> ⚠️ **Why this matters so much:** exposed databases with default/weak credentials are one of the *most common* sources of mass data breaches. Bots continuously scan the internet for open `5432`/`3306`/`27017`/`6379` ports. "Bind to localhost, never open the firewall" defeats that entire attack class before it starts.

### App ↔ database: connection strings and secrets

Your app connects to the DB using a **connection string** (or a set of parameters): host, port, database name, username, password. Example shape:

```
postgresql://appuser:SECRET_PASSWORD@127.0.0.1:5432/myapp_db
```

The **password is a secret**. It must **never** be hard-coded in your source (it would leak into git history, logs, and backups) and never committed to a repository. Instead it lives in the environment or a protected config file that the app reads at runtime — building directly on the `EnvironmentFile` idea from Chapter 10. We'll wire it up that way.

---

## Why is this necessary?

- **Without persistence, the app is a toy.** Anything a user creates vanishes on restart. A database is what makes an application *real* — it remembers.
- **The database holds your crown jewels.** It's the most valuable (business data) and most sensitive (personal data) thing on the server. Its security and integrity dwarf almost everything else — a breach here is the worst-case outcome.
- **Databases are prime targets.** The combination of "high value" and "commonly misconfigured" makes them heavily attacked. Running one *correctly* — localhost-bound, least-privilege, strong secrets — is a core production skill, not an afterthought.
- **It ties the whole stack together.** This chapter connects the app (Ch. 10), the privacy boundary (Ch. 6), least privilege (Ch. 3), secrets handling (Ch. 10), and sets up backups (Ch. 16). It's where all the earlier hardening pays off around your most important asset.

---

## What would happen if we skipped this step?

- **No memory.** The app can't store or retrieve anything durable; it's stateless and can't support users, content, or transactions.
- **You'd be tempted by unsafe shortcuts.** Storing data in flat files by hand invites corruption, race conditions, and no querying — reinventing a database, badly.
- **If you added a DB carelessly, you'd risk the classic breach.** Binding to `0.0.0.0`, opening the firewall "to make it work," or using a weak/default password is exactly how databases get harvested. Skipping the *secure* setup is worse than no database.
- **Backups would have nothing to protect (yet), and secrets would leak.** Hard-coding credentials in code spreads them into git and logs — a compromise waiting to happen.

---

## Alternative approaches

### Which database

| Option | Pros | Cons | Verdict |
|---|---|---|---|
| **PostgreSQL** | Correct, feature-rich, great general default, strong tooling, scales well. | Slightly more concepts than MySQL for total beginners. | ✅ **Recommended** worked example. |
| **MySQL / MariaDB** | Ubiquitous, simple, fast reads, huge ecosystem. | Historically looser defaults; fewer advanced features than PG. | ✅ Also excellent; same principles apply. |
| **SQLite** | Zero setup, single file, perfect for dev/small apps. | Not for high-concurrency/multi-host production. | ➕ Great for simple/small; know when to graduate. |
| **MongoDB (document)** | Flexible schema, JSON-native. | Weaker default consistency historically; not needed for most relational data. | ➕ Use when your data is genuinely document-shaped. |
| **Redis (key-value)** | Blazing fast; ideal as a **cache**/session store/queue. | In-memory (persistence is secondary); not a primary system-of-record for most apps. | ➕ Excellent *alongside* a primary DB, not instead of it. |
| **Managed DB (RDS, Cloud SQL, etc.)** | Provider handles backups, patching, HA, scaling. | Costs more; less control; another vendor. | ➕ Superb for production teams; self-hosting here teaches the fundamentals. |

### Where to run it

| Option | Pros | Cons | Verdict |
|---|---|---|---|
| **Same server, localhost-bound** | Simple, fast (no network hop), free, private by design. | DB and app share resources; single point of failure. | ✅ **Recommended** for a single-server setup — our approach. |
| **Separate DB server (private network)** | Isolation, independent scaling. | More infra; needs private networking + firewalling. | ➕ The next step as you grow. |
| **Managed database service** | Offloads ops (backups/patching/HA). | Cost, vendor lock-in. | ➕ Great when ops burden matters. |

**Our choice:** PostgreSQL on the same server, **bound to `127.0.0.1`**, connected to by the app over localhost. It's the simplest correct production shape and teaches every principle you'll carry to bigger setups.

---

## Commands

> Log in as **`deploy`** (Chapter 5). Use `sudo`. We'll install PostgreSQL, confirm it's localhost-only, create a least-privilege database + user, and wire the app (`appuser` service from Chapter 10) to it via a protected secret. Principles are identical for MySQL/MariaDB — only the command names differ.

### 1 — Install PostgreSQL

```bash
sudo apt update && sudo apt install postgresql
```
- **What it does:** installs the PostgreSQL server (and client tools). Like Nginx, it **starts and enables itself** as a systemd service on install.
- **Expected output:** apt install plan, then setup.
- **How to verify it's running:**
  ```bash
  systemctl status postgresql
  ```
  `active (running)` (technically a wrapper that starts the versioned `postgresql@16-main` service). `q` to exit.
- **Check the version/cluster:** `sudo -u postgres psql -c "SELECT version();"` prints the server version (more on `sudo -u postgres` in Step 3).

### 2 — ⭐ Confirm it listens on localhost only (the critical security check)

```bash
sudo ss -tulpn | grep 5432
```
- **What it does:** shows what's listening on PostgreSQL's default port **5432** (`ss` from Chapter 6).
- **Expected output (good):** the address should be **`127.0.0.1:5432`** (and possibly `::1:5432` for IPv6 localhost) — **not** `0.0.0.0:5432`.
  ```
  tcp  LISTEN 0 244 127.0.0.1:5432  ...  users:(("postgres",...))
  ```
- **Why this matters:** `127.0.0.1` means the database is reachable **only from the server itself** — the internet cannot route to it (Background). Ubuntu's PostgreSQL package defaults to localhost-only, which is exactly what we want. **Do not change this** unless you deliberately need remote access (and then only over a private network with firewalling).
- **The controlling setting** lives in `/etc/postgresql/16/main/postgresql.conf` as `listen_addresses = 'localhost'`. Leave it as localhost.
- **Belt and suspenders:** we also never add a `ufw` rule for 5432 (Chapter 6 default-deny already blocks it). So even a misconfiguration wouldn't expose it. Confirm: `sudo ufw status` should **not** list 5432.
- **Common mistake:** following a tutorial that says "set `listen_addresses = '*'` and open the firewall" — that's how databases get breached. Don't, unless you fully understand and need it.

### 3 — Understand PostgreSQL's admin access model

PostgreSQL installs with a special OS user **`postgres`** and a matching database superuser. On Ubuntu it uses **peer authentication** for local connections: the OS user `postgres` can log in as the DB superuser `postgres` without a password. You access the admin shell like this:

```bash
sudo -u postgres psql
```
- **What it does:** `sudo -u postgres` runs a command *as the OS user `postgres`* (Chapter 3's user model); `psql` is PostgreSQL's interactive SQL shell. Together: open an admin DB session.
- **Expected:** the prompt changes to `postgres=#` — you're now issuing SQL as the superuser.
- **How to leave:** type `\q` and Enter (psql's quit command; backslash commands are psql **meta-commands**).
- **Why this model:** it means the DB superuser isn't exposed via a network password at all — you must already be root/`postgres` on the box. Another layer of least privilege.

### 4 — Create a database and a least-privilege user

We create a dedicated DB for the app and a dedicated login role with a strong password and only the rights it needs — mirroring the "dedicated unprivileged user" idea from Chapter 10, now at the database layer.

First, **generate a strong password** for the DB user (on the server):
```bash
openssl rand -base64 24
```
- **What it does:** prints a strong random string (~32 chars). Copy it — you'll use it in the next command and in the app's secret file. (`openssl` is already present from Chapter 11.)

Now open the admin shell and create the role and database:
```bash
sudo -u postgres psql
```
At the `postgres=#` prompt, run these SQL statements (replace `STRONG_PASSWORD_HERE` with the generated one):
```sql
CREATE DATABASE myapp_db;
CREATE USER appuser WITH ENCRYPTED PASSWORD 'STRONG_PASSWORD_HERE';
GRANT ALL PRIVILEGES ON DATABASE myapp_db TO appuser;
\c myapp_db
GRANT ALL ON SCHEMA public TO appuser;
\q
```
- **Line-by-line:**
  - `CREATE DATABASE myapp_db;` — creates the application's database.
  - `CREATE USER appuser WITH ENCRYPTED PASSWORD '...';` — creates a **login role** with a password (stored hashed). This is a *database* user, distinct from the OS `appuser` from Chapter 10 (reusing the name for clarity is fine).
  - `GRANT ALL PRIVILEGES ON DATABASE myapp_db TO appuser;` — lets `appuser` use that database.
  - `\c myapp_db` — connect to the new DB (needed for the next grant).
  - `GRANT ALL ON SCHEMA public TO appuser;` — on modern PostgreSQL (15+), grant table-creation rights in the default schema so the app can create its tables.
  - `\q` — quit.
- **Least privilege in practice:** `appuser` can fully manage **its own** database (`myapp_db`) but has **no rights** over other databases or the server. For even tighter control in advanced setups, you'd grant only specific privileges (SELECT/INSERT/UPDATE/DELETE) rather than `ALL` — but per-database `ALL` for a dedicated app user is a reasonable, common baseline.
- **Expected output:** each statement echoes `CREATE DATABASE`, `CREATE ROLE`, `GRANT`, etc.
- **Common mistakes:** forgetting the quotes around the password; reusing a weak/guessable password; granting the app user superuser rights (never do that).

### 5 — Test the app user can connect (over localhost, with password)

```bash
psql "postgresql://appuser:STRONG_PASSWORD_HERE@127.0.0.1:5432/myapp_db" -c "\conninfo"
```
- **What it does:** connects *as the app database user* using a **connection string** (Background) and runs `\conninfo`, which reports the connection details.
- **Why we run it:** to prove the credentials and grants work over the localhost TCP connection your app will use — before wiring the app.
- **Expected output:** `You are connected to database "myapp_db" as user "appuser" on host "127.0.0.1" ... port "5432".`
- **Common mistakes & recovery:**
  - `password authentication failed` → wrong password, or PostgreSQL's `pg_hba.conf` requires a different auth method for TCP. On Ubuntu, local TCP (`host` lines for `127.0.0.1`) typically uses `scram-sha-256`/`md5` (password) by default — which is what we want. If it fails, inspect `/etc/postgresql/16/main/pg_hba.conf`.
  - `Connection refused` → the DB isn't running or isn't on 5432 (Step 2).

### 6 — Store the DB credentials as a protected secret (not in code)

We give the credentials to the app via an **`EnvironmentFile`** (Chapter 10) with tight permissions — never hard-coded.

```bash
sudo nano /etc/myapp.env
```
Enter (one `KEY=value` per line):
```
DATABASE_URL=postgresql://appuser:STRONG_PASSWORD_HERE@127.0.0.1:5432/myapp_db
```
Save and exit, then lock it down:
```bash
sudo chown root:appuser /etc/myapp.env
sudo chmod 640 /etc/myapp.env
```
- **What it does:** stores the connection string (with the secret) in a file readable only by `root` and the `appuser` group — **`640`** = owner read/write, group read, others nothing (Chapter 3's permissions doing real security work). The app (running as `appuser`) can read it; no one else can.
- **Why this way:** secrets stay **out of your source code, out of git, out of your shell history, out of logs**. Rotating the password later means editing one file, not redeploying code. This directly applies Chapter 10's secrets best practice.
- **Verify:** `ls -l /etc/myapp.env` → `-rw-r----- root appuser`.
- **Common mistake:** committing credentials to a repo, or leaving the file world-readable (`644`). Check the permissions.

### 7 — Wire the service to the secret

Add the `EnvironmentFile` to your app's systemd unit (Chapter 10), so the app receives `DATABASE_URL` at runtime:

```bash
sudo nano /etc/systemd/system/myapp.service
```
Under `[Service]`, add:
```ini
EnvironmentFile=/etc/myapp.env
```
Then reload systemd and restart the app (Chapter 10 ritual):
```bash
sudo systemctl daemon-reload
sudo systemctl restart myapp
```
- **What it does:** systemd reads `/etc/myapp.env` and injects `DATABASE_URL` into the app's environment. Your application code reads it (e.g. `process.env.DATABASE_URL` in Node, `os.environ["DATABASE_URL"]` in Python) and connects — **no password in the code**.
- **Verify:** `systemctl status myapp` shows `active (running)`; `sudo journalctl -u myapp -e` shows your app connecting to the database successfully (no auth errors). If your example app doesn't actually use the DB, this step is the *pattern* you'll use the moment it does.
- **Common mistakes:** forgetting `daemon-reload` after editing the unit (Chapter 10); the app reading a different env var name than you set.

### 8 — Everyday database management (reference)

```bash
sudo -u postgres psql            # admin shell (superuser)
\l                               # (inside psql) list databases
\du                              # list roles/users
\c myapp_db                      # connect to a database
\dt                              # list tables in the current DB
\q                               # quit
```
```bash
systemctl status postgresql              # is it running?
sudo systemctl restart postgresql        # restart the DB service
sudo journalctl -u postgresql@16-main -e # DB server logs (adjust version)
```
- These mirror the service-management and log-reading skills from Chapters 9–10, now applied to the database. The DB logs also live under `/var/log/postgresql/`.

> 🗄️ **A note on backups (Chapter 16 preview):** a running database is not a backup. The tool `pg_dump` exports a database to a file you can restore; `pg_dumpall` covers the whole cluster. We'll build a proper, scheduled, tested backup strategy in Chapter 16 — but know now that **the data in this database is precisely what you must back up.**

---

## Verification Checklist

You've completed this chapter when **all** of the following are true:

- [ ] You can explain **relational vs NoSQL**, where **SQLite** fits, and why a typical web app starts relational.
- [ ] PostgreSQL is installed and `systemctl status postgresql` shows it running.
- [ ] `sudo ss -tulpn | grep 5432` shows it bound to **`127.0.0.1`** (localhost only), and `sudo ufw status` does **not** list 5432.
- [ ] You created **`myapp_db`** and a least-privilege **`appuser`** role with a strong password, granted only rights over its own database.
- [ ] `psql "postgresql://appuser:...@127.0.0.1:5432/myapp_db" -c "\conninfo"` connects successfully.
- [ ] The connection string lives in **`/etc/myapp.env`** with **`640`** `root:appuser` permissions — **not** in code.
- [ ] The systemd unit has **`EnvironmentFile=/etc/myapp.env`**, and the app restarts and connects cleanly (`journalctl -u myapp`).
- [ ] You understand the golden rule: **the database is never exposed to the internet.**

---

## Troubleshooting

| Symptom | Why it happens | How to fix |
|---|---|---|
| `psql: FATAL: password authentication failed for user "appuser"` | Wrong password, or `pg_hba.conf` auth method mismatch for the TCP connection. | Re-check the password (recreate with `ALTER USER appuser WITH PASSWORD '...';`); inspect `/etc/postgresql/16/main/pg_hba.conf` `host` lines for `127.0.0.1` (should be `scram-sha-256`/`md5`); `sudo systemctl reload postgresql` after edits. |
| `psql: could not connect to server: Connection refused` | DB not running, or not listening on 5432/localhost. | `systemctl status postgresql`; `sudo ss -tulpn | grep 5432`; start it if stopped. |
| `FATAL: database "myapp_db" does not exist` | DB not created, or typo. | `sudo -u postgres psql -c "\l"` to list; `CREATE DATABASE myapp_db;` if missing. |
| App logs `permission denied for schema public` / can't create tables | On PostgreSQL 15+, the app role lacks schema privileges. | Connect to the DB (`\c myapp_db`) and `GRANT ALL ON SCHEMA public TO appuser;` (Step 4). |
| App can't read `DATABASE_URL` | `EnvironmentFile` missing/typo, wrong var name, or forgot `daemon-reload`. | Verify `/etc/myapp.env` and the `EnvironmentFile=` line; `sudo systemctl daemon-reload && sudo systemctl restart myapp`; confirm the app reads the same variable name. |
| I want remote access to the DB | Default is localhost-only (by design). | **Prefer not to.** If truly needed, use a **private network** + `ufw allow from <app-server-ip> to any port 5432`, set `listen_addresses` accordingly, and require TLS — never open 5432 to `Anywhere`. |
| Forgot the app user's password | Passwords are hashed; can't recover. | Reset it: `sudo -u postgres psql -c "ALTER USER appuser WITH ENCRYPTED PASSWORD 'NEW';"`, then update `/etc/myapp.env` and restart the app. |
| `/etc/myapp.env` readable by everyone | Permissions too loose. | `sudo chmod 640 /etc/myapp.env && sudo chown root:appuser /etc/myapp.env`. |

> **First stops for DB problems:** `systemctl status postgresql` (is it up?), `sudo ss -tulpn | grep 5432` (is it listening where expected?), the DB logs (`journalctl -u postgresql@16-main` / `/var/log/postgresql/`), and the app's own logs (`journalctl -u myapp`). Auth and connection issues are almost always password, `pg_hba.conf`, or the env file.

---

## Best Practices

- **Never expose the database to the internet.** Bind to `127.0.0.1`, never open its port in the firewall. This one rule prevents the most common database breaches. Verify with `ss` and `ufw status`.
- **One dedicated, least-privilege DB user per app.** The app role owns only its own database — never a superuser. Contain the blast radius of a leaked app credential.
- **Strong, generated passwords; secrets in a protected file, never in code.** Use `openssl rand`; store the connection string in a `640` `EnvironmentFile` owned appropriately; keep it out of git, logs, and history (Chapter 10).
- **Run the DB as the managed service it is.** Use `systemctl`/`journalctl` to operate and inspect it, like every other service. Let it start on boot.
- **Keep the DB patched.** It's covered by your update habits (Chapters 4 & 7) — a database is exactly the kind of internet-adjacent software that must stay current.
- **Choose the right tool, not the trendy one.** Relational by default; SQLite for small/simple; Redis as a *cache alongside*, not a system-of-record; NoSQL when your data is genuinely that shape. Avoid premature scaling complexity.
- **Plan for backups now, build them in Chapter 16.** The data here is irreplaceable. A running database is *not* a backup — `pg_dump` on a schedule, tested restores, offsite copies. Treat this chapter as creating the thing you must protect.
- **Least privilege at the schema level for sensitive apps.** Where it matters, grant specific `SELECT/INSERT/UPDATE/DELETE` rather than `ALL`, and separate read-only roles for reporting.

---

## Summary

### What you learned

- What a **database** and **persistence** are, and why an app needs durable, disk-backed storage instead of volatile memory.
- **Relational (SQL)** vs **non-relational (NoSQL)** models, the **ACID** guarantees, where **SQLite** (single-file, serverless) fits vs a **client-server** DB, and why relational is the sensible default for most web apps.
- Why **PostgreSQL** is a strong default (with MySQL/MariaDB equally valid), and that the *security principles* are identical across them.
- The **golden rule**: a database is **bound to localhost and never exposed to the internet** — the internet talks to your app, your app talks to the DB — defeating the most common breach class before it starts.
- How to **install PostgreSQL** as a systemd service, **verify localhost-only binding** (`ss`, and no `ufw` rule), use the **`sudo -u postgres psql`** admin model, and create a **database + least-privilege user** with a strong generated password.
- How to connect the **app safely**: a **connection string** with the secret stored in a **`640` `EnvironmentFile`** (never in code), injected via **`EnvironmentFile=` in the systemd unit** — applying Chapter 10's secrets pattern.
- Everyday DB management with `psql` meta-commands and `systemctl`/`journalctl`, and the preview that **this data is exactly what Chapter 16 will back up**.

### What you'll build next

**Chapter 13 — Containers (optional path).** You now have a complete, secure, stateful web stack running *directly* on the server: hardened OS, reverse proxy, HTTPS, an always-on app, and a locked-down database. Chapter 13 steps back to examine a different way to package and run applications — **containers** (Docker and friends): what they are, the problems they solve (consistent environments, isolation, reproducible deploys), how they compare to the systemd-service approach you've mastered, and when they're worth the added complexity. It's marked *optional* deliberately — you can run excellent production systems without containers — but understanding them is essential for modern deployment and for Part IV's discussion of CI/CD. If you'd rather skip straight to shipping code, we can jump to Chapter 14.

> ✅ **Ready to continue?** Confirm and we'll proceed to Chapter 13 (or skip to Chapter 14 — Deployment Strategies — if you'd prefer to defer containers). If PostgreSQL won't start, the app can't connect, or the localhost-binding check surprised you, tell me exactly what you ran and the output of `systemctl status postgresql`, `sudo ss -tulpn | grep 5432`, and `sudo journalctl -u myapp -e`, and we'll fix it before moving on.
