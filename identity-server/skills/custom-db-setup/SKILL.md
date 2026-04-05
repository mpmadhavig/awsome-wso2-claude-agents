---
name: custom-db-setup
description: Use when setting up a custom database for WSO2 Identity Server. Triggers on requests like "set up database for IS", "configure MySQL for Identity Server", "connect postgres to WSO2 IS", "set up IS with MSSQL", "connect DB2 to WSO2 IS", "connect Oracle to WSO2 IS", or any request to wire a database to WSO2 Identity Server.
tools: Read, Write, Edit, Bash, Glob, Grep
---

# WSO2 Identity Server — Database Setup

You are helping the user set up and connect a database to WSO2 Identity Server (IS). The database runs in Docker for portability.

## Step 1 — Gather required inputs

Ask the user if not already provided:

- **IS location** — path to the IS zip file OR the already-extracted IS home directory (e.g. `/opt/wso2/wso2is-7.1.0` or `/Downloads/wso2is-7.1.0.zip`)
- **Database type** — `mysql`, `mssql`, `postgres`, `db2`, or `oracle`
- **Database version** — e.g. `8.0`, `9.5` for MySQL; `azure-sql-edge` for MSSQL (ARM/Apple Silicon); `16`, `17` for PostgreSQL; `db2-amd64` for DB2 (amd64 image, requires Rosetta/amd64 emulation on Apple Silicon); `free` for Oracle 23c Free (via `gvenzl/oracle-free` — works on both ARM64 and AMD64)
- **Database name** — default to `regdb` if not provided
- **DB username / password** — defaults: `root` / `root` for MySQL & PostgreSQL; `sa` / `Root@1234` for MSSQL; `db2inst1` / `wso2carbon` for DB2; `wso2carbon` / `wso2carbon` for Oracle (this becomes the Oracle schema/user in `FREEPDB1`)

> **Oracle on Mac:** Use `gvenzl/oracle-free` (Oracle 23c Free) — it is multi-arch and works natively on Apple Silicon. `gvenzl/oracle-xe` (21c XE) is AMD64-only and fails on ARM64 with `ORA-00443: PMON did not start` even with `--shm-size` or `--privileged`. If using Colima, start with `colima start --network-address` so ports are reachable via `localhost`.

## Step 2 — Resolve IS home directory

If the user provided a `.zip` file path, extract it first:

```bash
unzip -q "<zip-path>" -d "<parent-directory>"
```

Then locate the extracted folder (it will be the only directory created). Set `IS_HOME` to that path.

If the user provided an already-extracted directory, use it directly as `IS_HOME`.

Confirm `IS_HOME` exists and contains `repository/conf/deployment.toml`. If not, stop and ask the user to verify the path.

## Step 3 — Check Docker is running

```bash
docker info > /dev/null 2>&1 && echo "Docker OK" || echo "Docker not running"
```

If Docker is not running:
1. Tell the user: "Docker is not running. Please start Docker Desktop and reply here when it's ready."
2. **Wait for the user to confirm** before proceeding — do not continue to Step 4 until the user says Docker is up.
3. Once the user confirms, re-run the check above to verify Docker is actually running before continuing.

## Step 4 — Start the database Docker container

Use the version the user provided. Substitute `<version>` and `<db_name>` with actual values.

### MySQL

```bash
docker run --name wso2is_mysql_<version> \
  -p 3306:3306 \
  -e MYSQL_ROOT_PASSWORD=<password> \
  -e MYSQL_DATABASE=<db_name> \
  -d mysql:<version>
```

Wait for MySQL to be ready (retry up to 30s):

```bash
for i in $(seq 1 30); do
  docker exec wso2is_mysql_<version> mysqladmin ping -u root -p<password> --silent 2>/dev/null && echo "MySQL ready" && break
  echo "Waiting for MySQL... ($i/30)"
  sleep 1
done
```

### PostgreSQL

```bash
docker run --name wso2is_postgres_<version> \
  -p 5432:5432 \
  -e POSTGRES_PASSWORD=<password> \
  -e POSTGRES_DB=<db_name> \
  -d postgres:<version>
```

Wait for PostgreSQL to be ready:

```bash
for i in $(seq 1 30); do
  docker exec wso2is_postgres_<version> pg_isready -U postgres 2>/dev/null && echo "PostgreSQL ready" && break
  echo "Waiting for PostgreSQL... ($i/30)"
  sleep 1
done
```

### MSSQL (SQL Server)

MSSQL requires ACCEPT_EULA and a strong SA password (min 8 chars, upper+lower+digit+special).

**Azure SQL Edge** (ARM-compatible — use on Apple Silicon M1/M2/M3 or when standard image is unavailable; version label: `azure-sql-edge`):

```bash
docker run --name wso2is_mssql_azure-sql-edge \
  -p 1433:1433 \
  -e MSSQL_SA_PASSWORD=<password> \
  -e ACCEPT_EULA=Y \
  -d mcr.microsoft.com/azure-sql-edge:latest
```

> Note: Azure SQL Edge uses `MSSQL_SA_PASSWORD` (not `SA_PASSWORD`). The image does **not** ship `sqlcmd` — use the host `sqlcmd` (install via `brew install sqlcmd`) to connect on `localhost,1433`.

Wait for MSSQL / Azure SQL Edge to be ready (takes longer — up to 60s):

**Azure SQL Edge** (uses host `sqlcmd` — Azure SQL Edge image does not include `sqlcmd`):
```bash
for i in $(seq 1 60); do
  sqlcmd -S localhost,1433 -U sa -P "<password>" -C \
    -Q "SELECT 1" > /dev/null 2>&1 && echo "MSSQL ready" && break
  echo "Waiting for MSSQL... ($i/60)"
  sleep 1
done
```

### DB2

> **Note:** `ibmcom/db2-amd64` is an x86_64 image. On Apple Silicon it runs under Rosetta emulation via `--platform=linux/amd64`. The image is large (~3.5 GB) — the Docker/Rancher Desktop VM must have sufficient disk space before pulling.

**Before starting the container, verify the VM has enough free disk space:**

```bash
docker run --rm alpine df -h /
```

The output shows the VM's root filesystem. You need at least **8 GB free**. If not:

- **Docker Desktop**: Settings → Resources → Advanced → Virtual disk limit → increase and Apply & Restart
- **Rancher Desktop**: Preferences → Virtual Machine → Hardware → Disk size → increase and Apply (VM will restart)

After increasing, also prune unused Docker data:
```bash
docker system prune -f
```

The `DBNAME` env var auto-creates the database on first start.

```bash
docker run --platform=linux/amd64 -itd --name wso2is_db2_amd \
  --privileged=true \
  -p 50000:50000 \
  -e LICENSE=accept \
  -e DB2INST1_PASSWORD=<password> \
  -e DB2INSTANCE=db2inst1 \
  -e DBNAME=<db_name> \
  ibmcom/db2-amd64
```

DB2 takes significantly longer to initialise (~2–3 minutes). Wait for it:

```bash
for i in $(seq 1 120); do
  docker exec wso2is_db2_amd su - db2inst1 -c "db2 connect to <db_name>" \
    > /dev/null 2>&1 && echo "DB2 ready" && break
  echo "Waiting for DB2... ($i/120)"
  sleep 3
done
```

### Oracle 23c Free

> **Use `gvenzl/oracle-free`, not `gvenzl/oracle-xe`.** The XE (21c) image is AMD64-only and fails on Apple Silicon with `ORA-00443: PMON did not start`. The `oracle-free` image is multi-arch (ARM64 + AMD64) and works on all platforms. Oracle 23c Free uses `FREEPDB1` as the PDB service name.

The `ORACLE_PASSWORD` env var sets the password for the built-in `SYS` and `SYSTEM` accounts. A dedicated WSO2 user will be created in Step 5.

```bash
docker run -d \
  --name wso2is_oracle_xe \
  -p 1521:1521 \
  -e ORACLE_PASSWORD=<oracle_sys_password> \
  -v oracle-data:/opt/oracle/oradata \
  gvenzl/oracle-free
```

Wait for Oracle to be ready (takes 3–5 minutes — poll up to 5 min):

```bash
for i in $(seq 1 60); do
  docker logs wso2is_oracle_xe 2>/dev/null | grep -q "DATABASE IS READY TO USE" && echo "Oracle ready" && break
  echo "Waiting for Oracle... ($i/60)"
  sleep 5
done
```

> If `localhost` doesn't work after the container is ready (common with Colima), get the Colima VM IP: `colima list` — use the `ADDRESS` value as the host in all subsequent commands and in deployment.toml.

**If a container with the same name already exists**, check its state:

```bash
docker ps -a --filter "name=wso2is_<db>_<version>" --format "{{.Status}}"
```

- If `Exited`: start it with `docker start wso2is_<db>_<version>` and wait for readiness.
- If `Up`: reuse it (skip creation).
- Otherwise: create a new container with a different name (append `-2` or ask user).

## Step 5 — Create the database / schema

For MySQL, PostgreSQL, and DB2, the database is auto-created by the environment variables in Step 4 (`-e MYSQL_DATABASE`, `-e POSTGRES_DB`, `-e DBNAME`).

For **MSSQL**, create the database manually after the container is ready.

Azure SQL Edge (use host `sqlcmd`):
```bash
sqlcmd -S localhost,1433 -U sa -P "<password>" -C \
  -Q "IF NOT EXISTS (SELECT name FROM sys.databases WHERE name = '<db_name>') CREATE DATABASE [<db_name>]"
```

For **Oracle**, create a dedicated user/schema in `FREEPDB1` (the pluggable database — never use `FREE`, the CDB root). Oracle schemas are tied to users, so `<username>` also becomes the schema name.

> **Important:** If `<oracle_sys_password>` contains special characters like `@`, quote it in the Easy Connect string: `'sys/"<password>"@localhost/FREEPDB1 as sysdba'`. Unquoted `@` is interpreted as the connection string delimiter.

```bash
docker exec wso2is_oracle_xe bash -c "
echo 'CREATE USER <username> IDENTIFIED BY \"<password>\";
GRANT CREATE SESSION TO <username>;
GRANT DBA TO <username>;
ALTER USER <username> QUOTA UNLIMITED ON USERS;
EXIT;' \
| sqlplus -s 'sys/\"<oracle_sys_password>\"@localhost/FREEPDB1 as sysdba'"
```

> All Oracle grants must target `FREEPDB1`, not `FREE`. Connecting as `sys` requires `as sysdba`.

## Step 6 — Install the JDBC driver

The JDBC driver JAR must be placed in `<IS_HOME>/repository/components/lib/`.

Check if a compatible driver already exists:

```bash
ls "<IS_HOME>/repository/components/lib/" | grep -i "<driver-prefix>"
```

- MySQL driver prefix: `mysql-connector`
- PostgreSQL driver prefix: `postgresql`
- MSSQL driver prefix: `mssql-jdbc`
- DB2 driver prefix: `db2jcc`
- Oracle driver prefix: `ojdbc`

If a driver is **already present**, inform the user and skip download.

If **not present**, tell the user to download the JDBC driver and copy it to `<IS_HOME>/repository/components/lib/`:

### MySQL
Download `mysql-connector-j-<version>.jar` from:
`https://dev.mysql.com/downloads/connector/j/`

Or use Maven/Homebrew if available:
```bash
# If Maven is available
mvn dependency:get -Dartifact=com.mysql:mysql-connector-j:8.0.33:jar -Ddest="<IS_HOME>/repository/components/lib/"
```

### PostgreSQL
Download `postgresql-<version>.jar` from:
`https://jdbc.postgresql.org/download/`

```bash
# If curl is available
curl -L "https://jdbc.postgresql.org/download/postgresql-42.7.3.jar" \
  -o "<IS_HOME>/repository/components/lib/postgresql-42.7.3.jar"
```

### MSSQL
Download from:
`https://learn.microsoft.com/en-us/sql/connect/jdbc/download-microsoft-jdbc-driver-for-sql-server`

Extract and copy the `mssql-jdbc-<version>.jre11.jar` to `<IS_HOME>/repository/components/lib/`.

### DB2
Download `db2jcc4.jar` from IBM's JDBC driver page or copy it directly out of the running container (it is bundled in the image):

```bash
docker cp wso2is_db2_amd:/opt/ibm/db2/V11.5/java/db2jcc4.jar \
  "<IS_HOME>/repository/components/lib/db2jcc4.jar"
```

### Oracle
Download `ojdbc11.jar` from Maven Central (Oracle JDBC is published there since 19c):

```bash
# If Maven is available
mvn dependency:get \
  -Dartifact=com.oracle.database.jdbc:ojdbc11:23.4.0.24.05:jar \
  -Ddest="<IS_HOME>/repository/components/lib/"
```

Or download directly:
```bash
curl -L "https://repo1.maven.org/maven2/com/oracle/database/jdbc/ojdbc11/23.4.0.24.05/ojdbc11-23.4.0.24.05.jar" \
  -o "<IS_HOME>/repository/components/lib/ojdbc11-23.4.0.24.05.jar"
```

After any download, confirm the file is in place:

```bash
ls "<IS_HOME>/repository/components/lib/" | grep -i "<driver-prefix>"
```

## Step 7 — Check if database is already initialized

Before running the init scripts, check whether the database already contains tables.

### MySQL

```bash
TABLE_COUNT=$(docker exec wso2is_mysql_<version> mysql -u root -p<password> -sNe \
  "SELECT COUNT(*) FROM information_schema.tables WHERE table_schema='<db_name>';" 2>/dev/null)
echo "Tables found: $TABLE_COUNT"
```

### PostgreSQL

```bash
TABLE_COUNT=$(docker exec wso2is_postgres_<version> psql -U postgres -d <db_name> -tAc \
  "SELECT COUNT(*) FROM information_schema.tables WHERE table_schema='public';" 2>/dev/null)
echo "Tables found: $TABLE_COUNT"
```

### MSSQL (Azure SQL Edge — host sqlcmd)

```bash
TABLE_COUNT=$(sqlcmd -S localhost,1433 -U sa -P "<password>" -d <db_name> -C -h -1 \
  -Q "SET NOCOUNT ON; SELECT COUNT(*) FROM sys.tables" 2>/dev/null | tr -d ' ')
echo "Tables found: $TABLE_COUNT"
```

### DB2

> **Important:** `su - db2inst1 -c "..."` loses the DB2 connection context between semicolon-separated commands. Use the `bash -c` wrapper below to keep the connection alive across both statements in a single shell invocation.

```bash
TABLE_COUNT=$(docker exec wso2is_db2_amd bash -c \
  "su - db2inst1 -c 'db2 connect to <db_name>; db2 SELECT COUNT\(*\) FROM syscat.tables WHERE tabschema = UPPER\(CURRENT_USER\)'" \
  2>/dev/null | grep -E "^[[:space:]]*[0-9]" | tr -d ' ')
echo "Tables found: $TABLE_COUNT"
```

### Oracle

> **Note:** Use `echo` with double-quoted SQL inside the `bash -c` block — `printf` with nested single-quote escaping breaks the string literal inside `UPPER('...')`.

```bash
TABLE_COUNT=$(docker exec wso2is_oracle_xe bash -c "
echo \"SET HEADING OFF
SET FEEDBACK OFF
SET PAGESIZE 0
SELECT COUNT(*) FROM all_tables WHERE OWNER = UPPER('<username>');
EXIT;\" | sqlplus -s '<username>/<password>@localhost/FREEPDB1'" 2>/dev/null | grep -E "^[[:space:]]*[0-9]" | tr -d ' ')
echo "Tables found: $TABLE_COUNT"
```

**If `TABLE_COUNT > 0`**, the database already has tables. Ask the user:

> "The database `<db_name>` already exists and contains $TABLE_COUNT table(s). How would you like to proceed?
> 1. **Drop and recreate** — delete all existing data and start fresh
> 2. **Use a different name** — enter a new database name to create a clean database"

Wait for the user's choice before continuing.

**Option 1 — Drop and recreate:**

MySQL:
```bash
docker exec wso2is_mysql_<version> mysql -u root -p<password> \
  -e "DROP DATABASE \`<db_name>\`; CREATE DATABASE \`<db_name>\`;"
```

PostgreSQL:
```bash
docker exec wso2is_postgres_<version> psql -U postgres \
  -c "DROP DATABASE \"<db_name>\"; CREATE DATABASE \"<db_name>\";"
```

MSSQL (Azure SQL Edge):
```bash
sqlcmd -S localhost,1433 -U sa -P "<password>" -C \
  -Q "ALTER DATABASE [<db_name>] SET SINGLE_USER WITH ROLLBACK IMMEDIATE; DROP DATABASE [<db_name>]; CREATE DATABASE [<db_name>];"
```

DB2:
```bash
docker exec wso2is_db2_amd su - db2inst1 -c \
  "db2 connect to <db_name> && db2 drop database <db_name> && db2 create database <db_name> using codeset UTF-8 territory us"
```

Oracle (drop and recreate the user/schema in FREEPDB1):
```bash
docker exec wso2is_oracle_xe bash -c "
echo 'DROP USER <username> CASCADE;
CREATE USER <username> IDENTIFIED BY \"<password>\";
GRANT CREATE SESSION TO <username>;
GRANT DBA TO <username>;
ALTER USER <username> QUOTA UNLIMITED ON USERS;
EXIT;' \
| sqlplus -s 'sys/\"<oracle_sys_password>\"@localhost/FREEPDB1 as sysdba'"
```

**Option 2 — Use a different name:**

Set `<db_name>` to the new name the user provided, then create the database:

MySQL:
```bash
docker exec wso2is_mysql_<version> mysql -u root -p<password> \
  -e "CREATE DATABASE IF NOT EXISTS \`<new_db_name>\`;"
```

PostgreSQL:
```bash
docker exec wso2is_postgres_<version> psql -U postgres \
  -c "CREATE DATABASE \"<new_db_name>\";"
```

MSSQL (Azure SQL Edge):
```bash
sqlcmd -S localhost,1433 -U sa -P "<password>" -C \
  -Q "IF NOT EXISTS (SELECT name FROM sys.databases WHERE name = '<new_db_name>') CREATE DATABASE [<new_db_name>]"
```

DB2:
```bash
docker exec wso2is_db2_amd su - db2inst1 -c \
  "db2 create database <new_db_name> using codeset UTF-8 territory us"
```

Oracle (create a new user/schema — the new name becomes both the username and schema):
```bash
docker exec wso2is_oracle_xe bash -c "
echo 'CREATE USER <new_username> IDENTIFIED BY \"<password>\";
GRANT CREATE SESSION TO <new_username>;
GRANT DBA TO <new_username>;
ALTER USER <new_username> QUOTA UNLIMITED ON USERS;
EXIT;' \
| sqlplus -s 'sys/\"<oracle_sys_password>\"@localhost/FREEPDB1 as sysdba'"
```

Use `<new_db_name>` as `<db_name>` for all subsequent steps (init scripts, deployment.toml).

## Step 8 — Run the database initialization scripts

WSO2 IS ships with SQL scripts inside `<IS_HOME>/dbscripts/`. Run both the Carbon registry script and the Identity script.

### MySQL

```bash
docker exec -i wso2is_mysql_<version> mysql -u root -p<password> <db_name> \
  < "<IS_HOME>/dbscripts/mysql.sql"

docker exec -i wso2is_mysql_<version> mysql -u root -p<password> <db_name> \
  < "<IS_HOME>/dbscripts/identity/mysql.sql"
```

### PostgreSQL

```bash
docker exec -i wso2is_postgres_<version> psql -U postgres -d <db_name> \
  < "<IS_HOME>/dbscripts/postgresql.sql"

docker exec -i wso2is_postgres_<version> psql -U postgres -d <db_name> \
  < "<IS_HOME>/dbscripts/identity/postgresql.sql"
```

### MSSQL

**Azure SQL Edge** (use host `sqlcmd`):
```bash
sqlcmd -S localhost,1433 -U sa -P "<password>" -d <db_name> -C \
  -i "<IS_HOME>/dbscripts/mssql.sql"

sqlcmd -S localhost,1433 -U sa -P "<password>" -d <db_name> -C \
  -i "<IS_HOME>/dbscripts/identity/mssql.sql"
```

### DB2

> **Critical — two gotchas confirmed in practice:**
> 1. **Use `-td/ -vf`, not `-tvf`** — WSO2 DB2 scripts use `/` as the statement delimiter. `-tvf` defaults to `;` and will treat the entire file as one giant statement, failing with `SQL0104N`.
> 2. **Connection context is lost between `&&`-chained commands** — `su - db2inst1 -c "db2 connect ... && db2 -td/ -vf ..."` drops the connection between commands. Use a shell script written into the container instead.

Copy scripts into the container, then run via a helper shell script:

```bash
docker cp "<IS_HOME>/dbscripts/db2.sql" wso2is_db2_amd:/tmp/db2.sql
docker cp "<IS_HOME>/dbscripts/identity/db2.sql" wso2is_db2_amd:/tmp/db2_identity.sql

# Write and run the Carbon/shared DB script
docker exec wso2is_db2_amd bash -c "cat > /tmp/run_db2.sh << 'SCRIPT'
#!/bin/bash
db2 connect to <db_name>
db2 -td/ -vf /tmp/db2.sql
db2 connect reset
SCRIPT
chmod +x /tmp/run_db2.sh"

docker exec wso2is_db2_amd su - db2inst1 -c "bash /tmp/run_db2.sh" 2>&1 | tail -20

# Write and run the Identity DB script
docker exec wso2is_db2_amd bash -c "cat > /tmp/run_db2_identity.sh << 'SCRIPT'
#!/bin/bash
db2 connect to <db_name>
db2 -td/ -vf /tmp/db2_identity.sql
db2 connect reset
SCRIPT
chmod +x /tmp/run_db2_identity.sh"

docker exec wso2is_db2_amd su - db2inst1 -c "bash /tmp/run_db2_identity.sh" 2>&1 | tail -20
```

> **Note on IS 7.x DB2 scripts:** `dbscripts/db2.sql` contains the full shared schema — Carbon registry, user management (UM_*), org hierarchy, etc. It is NOT just the registry. `dbscripts/identity/db2.sql` is the identity-specific schema (IDN_*, OAuth, SAML, etc.). Both are required. Together they create ~243 tables on a fresh IS 7.2.x install.

### Oracle

Copy scripts into the container, then run via `sqlplus` as the WSO2 user:

```bash
docker cp "<IS_HOME>/dbscripts/oracle.sql" wso2is_oracle_xe:/tmp/oracle.sql
docker cp "<IS_HOME>/dbscripts/identity/oracle.sql" wso2is_oracle_xe:/tmp/oracle_identity.sql

# Run the Carbon/shared schema script
docker exec wso2is_oracle_xe bash -c "
echo 'WHENEVER SQLERROR CONTINUE
@/tmp/oracle.sql
EXIT;' | sqlplus -s '<username>/<password>@localhost/FREEPDB1'" 2>&1 | tail -30

# Run the Identity schema script
docker exec wso2is_oracle_xe bash -c "
echo 'WHENEVER SQLERROR CONTINUE
@/tmp/oracle_identity.sql
EXIT;' | sqlplus -s '<username>/<password>@localhost/FREEPDB1'" 2>&1 | tail -30
```

> `WHENEVER SQLERROR CONTINUE` (rather than `EXIT`) lets the script proceed past "object already exists" errors (ORA-00955, ORA-02260, etc.), which are safe to ignore on re-runs.

If scripts fail with errors other than "object already exists", show them to the user.

## Step 9 — Update deployment.toml

Read the current deployment.toml first:

```
<IS_HOME>/repository/conf/deployment.toml
```

Locate the `[database.identity_db]` and `[database.shared_db]` sections. If they don't exist, append them to the end of the file.

Replace or add them using the correct config for each database type:

### MySQL

```toml
[database.identity_db]
type = "mysql"
hostname = "localhost"
name = "<db_name>"
port = "3306"
username = "<username>"
password = "<password>"

[database.shared_db]
type = "mysql"
hostname = "localhost"
name = "<db_name>"
port = "3306"
username = "<username>"
password = "<password>"
```

### PostgreSQL

```toml
[database.identity_db]
type = "postgre"
hostname = "localhost"
name = "<db_name>"
port = "5432"
username = "<username>"
password = "<password>"

[database.shared_db]
type = "postgre"
hostname = "localhost"
name = "<db_name>"
port = "5432"
username = "<username>"
password = "<password>"
```

### MSSQL

```toml
[database.identity_db]
type = "mssql"
url = "jdbc:sqlserver://127.0.0.1:1433;databaseName=<db_name>;SendStringParametersAsUnicode=false;trustServerCertificate=true"
username = "<username>"
password = "<password>"

[database.shared_db]
type = "mssql"
url = "jdbc:sqlserver://127.0.0.1:1433;databaseName=<db_name>;SendStringParametersAsUnicode=false;trustServerCertificate=true"
username = "<username>"
password = "<password>"
```

### DB2

```toml
[database.identity_db]
type = "db2"
url = "jdbc:db2://localhost:50000/<db_name>:currentSchema=DB2INST1;"
username = "db2inst1"
password = "<password>"

[database.shared_db]
type = "db2"
url = "jdbc:db2://localhost:50000/<db_name>:currentSchema=DB2INST1;"
username = "db2inst1"
password = "<password>"
```

### Oracle

Use the JDBC thin URL pointing to `FREEPDB1` (the pluggable database for Oracle 23c Free). The `<username>` is also the Oracle schema name.

```toml
[database.identity_db]
type = "oracle"
url = "jdbc:oracle:thin:@//localhost:1521/FREEPDB1"
username = "<username>"
password = "<password>"

[database.shared_db]
type = "oracle"
url = "jdbc:oracle:thin:@//localhost:1521/FREEPDB1"
username = "<username>"
password = "<password>"
```

> If `localhost` is unreachable (Colima without `--network-address`), replace `localhost` with the Colima VM IP from `colima list`.

**Edit rules:**
- If `[database.identity_db]` already exists in the file, replace the entire block. Do not duplicate it.
- If it does not exist, append both `[database.identity_db]` and `[database.shared_db]` blocks to the end.
- Use the `Edit` tool to replace existing blocks, or `Write` to append to the file.
- After editing, read the relevant section back and show the user the final config.

## Step 10 — Summary and Start IS

Print a clear summary:

```
✅ Database setup complete

  Docker container : wso2is_<db>_<version>
  DB type          : <db_type>
  DB name          : <db_name>  (<N> tables created)
  Host:Port        : localhost:<port>
  Username         : <username>

  deployment.toml  : <IS_HOME>/repository/conf/deployment.toml  ← updated
  JDBC driver      : <IS_HOME>/repository/components/lib/<driver.jar>
```

If any step failed, clearly state which step and what went wrong. Do not start IS if any step failed.

After printing the summary, start WSO2 IS:

```bash
sh "<IS_HOME>/bin/wso2server.sh"
```

Run this in the background and tail the log, watching for either:
- `WSO2 Carbon started in` line — indicates IS started successfully
- `FATAL` in startup — indicates a failure

```bash
sh "<IS_HOME>/bin/wso2server.sh" > /tmp/wso2is_startup.log 2>&1 &
# Wait up to 120 seconds for IS to start (poll every 10s)
for i in $(seq 1 12); do
  sleep 10
  if grep -q "WSO2 Carbon started in" /tmp/wso2is_startup.log 2>/dev/null; then
    echo "WSO2 IS started successfully"
    grep "WSO2 Carbon started in" /tmp/wso2is_startup.log
    break
  fi
  if grep -q "FATAL" /tmp/wso2is_startup.log 2>/dev/null; then
    echo "IS startup error detected:"
    grep -i "fatal" /tmp/wso2is_startup.log | tail -20
    break
  fi
  echo "Waiting for IS to start... ($i/12)"
done
```

If IS has not started within 120 seconds, read `<IS_HOME>/repository/logs/wso2carbon.log` for more detail:

```bash
tail -50 "<IS_HOME>/repository/logs/wso2carbon.log"
```

Look for ERROR or FATAL lines and report them to the user. Continue polling `/tmp/wso2is_startup.log` for the `WSO2 Carbon started in` line until IS is up or a fatal error is confirmed.

Once IS is up, print the startup line from the log so the user can see IS is running.

---

## Quick Reference — DB Versions & Docker Images

| DB Type         | Tested Versions (IS 7.x) | Docker Image                                          | Port  |
|-----------------|--------------------------|-------------------------------------------------------|-------|
| MySQL           | 9.5, 8.4, 8.0, 5.7       | `mysql:<version>`                                    | 3306  |
| PostgreSQL      | 17.x, 16.x, 15.x         | `postgres:<version>`                                 | 5432  |
| MSSQL (ARM/M1+) | azure-sql-edge            | `mcr.microsoft.com/azure-sql-edge:latest`            | 1433  |
| DB2             | 11.5 (amd64)             | `ibmcom/db2-amd64` (needs `--platform=linux/amd64`)  | 50000 |
| Oracle          | 23c Free (multi-arch)    | `gvenzl/oracle-free` (ARM64 + AMD64 native; **do not use** `gvenzl/oracle-xe` on Apple Silicon) | 1521 |

## Notes

- The `[database.shared_db]` and `[database.identity_db]` can point to the same database — this is the common dev/test setup. For production, they are typically separate databases.
- For MSSQL, the SA password must meet SQL Server complexity requirements: min 8 chars, mix of uppercase, lowercase, digit, and special character. Default used here: `Root@1234`.
- If the user is on Apple Silicon (M1/M2/M3), use `mcr.microsoft.com/azure-sql-edge:latest` instead of the standard MSSQL image — it has native ARM support and uses `MSSQL_SA_PASSWORD` (not `SA_PASSWORD`). The image does not include `sqlcmd`; use the host `sqlcmd` (`brew install sqlcmd`) connecting to `localhost,1433`.
- PostgreSQL type string in deployment.toml is `"postgre"` (not `"postgresql"`) — this is correct per WSO2 documentation.
- DB scripts are idempotent for table creation errors. If tables already exist from a prior run, those errors are safe to ignore.
- DB2 runs as the `db2inst1` OS user inside the container; all DB2 commands must be run via `su - db2inst1 -c "..."` or as that user.
- The `ibmcom/db2-amd64` image is large (~3.5 GB). Before pulling, verify free disk space with `docker run --rm alpine df -h /` — you need at least 8 GB free in the VM. If not enough space:
  - **Docker Desktop**: Settings → Resources → Advanced → Virtual disk limit
  - **Rancher Desktop**: Preferences → Virtual Machine → Hardware → Disk size
  After increasing, run `docker system prune -f` to free up any unused layers.
- The `db2jcc4.jar` driver is bundled inside the `ibmcom/db2-amd64` image at `/opt/ibm/db2/V11.5/java/db2jcc4.jar` — use `docker cp` to extract it rather than downloading separately.
- **DB2 script delimiter:** WSO2 DB2 scripts use `/` as the statement terminator. Always run them with `db2 -td/ -vf <script>`. Using `db2 -tvf` (which defaults to `;`) will fail with `SQL0104N` — the entire file is treated as one invalid statement.
- **DB2 connection context:** `su - db2inst1 -c "cmd1 && cmd2"` loses the DB2 connection between commands. Write a shell script into the container and run it with `su - db2inst1 -c "bash /tmp/script.sh"` to keep the session alive across `db2 connect` + `db2 -td/ -vf`.
- **Oracle schema = user:** Oracle doesn't have separate "databases" within the PDB — each user owns their own schema. The WSO2 `<username>` is both the login credential and the schema that holds all WSO2 tables.
- **Oracle PDB vs CDB:** Always connect to `FREEPDB1` (the pluggable database for Oracle 23c Free), not `FREE` (the CDB root). The WSO2 user, grants, and all data live in `FREEPDB1`.
- **Oracle on Mac/Colima:** Start Colima with `colima start --network-address` so Docker ports are reachable via `localhost`. Without it, use the VM IP from `colima list` as the host everywhere (Docker run, sqlplus, JDBC URL).
- **Oracle init scripts:** `WHENEVER SQLERROR CONTINUE` in sqlplus allows scripts to skip "object already exists" errors (ORA-00955, ORA-02260) safely. Use this on re-runs.
- **Oracle JDBC driver:** `ojdbc11.jar` is available on Maven Central at `com.oracle.database.jdbc:ojdbc11`. No Oracle account required.
- **Oracle password with `@`:** In sqlplus Easy Connect strings, `@` is the host delimiter. If the SYS password contains `@` (e.g. `Oracle@1234`), quote it: `'sys/"Oracle@1234"@localhost/FREEPDB1 as sysdba'`. Unquoted, sqlplus will misparse the connection string.
- **`gvenzl/oracle-xe` on Apple Silicon:** Fails with `ORA-00443: background process "PMON" did not start`. This is a known ARM64/Rosetta incompatibility — not fixable with `--shm-size` or `--privileged`. Use `gvenzl/oracle-free` instead.
