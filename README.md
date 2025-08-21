# Oracle to PostgreSQL Database Migration Runbook

This guide details the steps to migrate an Oracle database to PostgreSQL. The migration process includes structure migration and data migration.


## Prerequisites

Before beginning the migration process, ensure the following software and environment configurations are in place:

1. **Install AWS Schema Conversion Tool (SCT) Software**
   AWS SCT is required for structure migration.

2. **Install Ora2pg Software**
   Ora2pg is used for data migration.

3. **Validate PostgreSQL Server**
   Ensure PostgreSQL is set up and validated on the destination server.

4. **Connector Module**
   Ensure the necessary connector modules for Oracle and PostgreSQL are installed and functional.

5. **Scheduler Module**
   Verify that the scheduler module is configured for periodic migrations, if needed.

6. **PostgreSQL Database Version Validation**
   Ensure that the target PostgreSQL database version is compatible with the migration.

7. **Oracle Database Version Validation**
   Verify the source Oracle database version to ensure compatibility with migration tools.

8. **Verify Database Connectivity**
   Confirm that both source (Oracle) and target (PostgreSQL) databases are accessible from the migration environment.

9. **Get Row Count of Each Table from Oracle Database**
   Run a query to get the row count of each table for validation after migration.

   ```sql
   SELECT table_name, num_rows FROM all_tables WHERE owner = '<schema>';
   ```

10. **Perform Stats Gathering on Oracle Database**
    Run statistics gathering for optimal migration performance.

    ```sql
    EXEC DBMS_STATS.GATHER_SCHEMA_STATS('<schema>');
    ```

11. **Configure PostgreSQL Database for Performance Optimization**
    Configure parameters like `shared_buffers`, `work_mem`, `maintenance_work_mem`, and others for performance.


# Migration Steps

### Structure Migration (Estimated Time: 4 Hours)

#### Prerequisites for Target PostgreSQL Database

Before beginning the structure migration, ensure the following steps are completed on the target PostgreSQL database:

Create Role/User on Target PostgreSQL Database
Create a new role or user with appropriate privileges for the migration.

```bash
CREATE ROLE <username> WITH LOGIN PASSWORD '<password>';
```

Create Tablespace on Target PostgreSQL Instance. Create a tablespace where the migrated data will be stored. This step is optional but recommended for managing storage efficiently.
```bash
CREATE TABLESPACE <tablespace_name> OWNER <username> LOCATION '<path>';
```

Create Database on Target PostgreSQL Instance
Create the new database where the Oracle schema will be migrated.
```bash
CREATE DATABASE <database_name> OWNER <username> ENCODING 'UTF8' TABLESPACE <tablespace_name>;
GRANT ALL ON DATABASE <database_name> TO <username>;
GRANT CONNECT ON DATABASE <database_name> TO <username>;
GRANT ALL PRIVILEGES ON DATABASE <database_name> TO <username>;
ALTER DATABASE <database_name> SET aws_oracle_ext.tz = 'UTC';
```

After the database is created, connect to the newly created database as the specified user to perform schema creation.
```bash
psql -U <username> -d <database_name> -h <host> -p <port>
```

Create Desired Schema on Target PostgreSQL Database

```bash
CREATE SCHEMA <schema_name> AUTHORIZATION <username>;
CREATE SCHEMA <schema_name>;
ALTER SCHEMA <schema_name> OWNER TO <username>;
ALTER ROLE <username> SET search_path = <schema_name>;
ALTER DATABASE <database_name> SET search_path = <schema_name>;
GRANT USAGE ON SCHEMA <schema_name> TO <username>;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA <schema_name> TO <username>;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA <schema_name> TO <username>;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA <schema_name> TO <username>;
```

### Step 1. Structure Migration (Estimated Time: 4 Hours)

#### **Option 1: Backup and Restore (Schema Only)**

**Expected Time: 10 minutes**

1. **Backup UAT Database (Schema Only)**

    ```bash
    pg_dump -U <user> -d <database> -h <host> -p <port> --schema-only --no-owner --no-tablespaces --verbose -F p -f <backup-file>.sql
    ```

2. **Restore the Structure to Target Environment**

   ```bash
   psql -U <user> -d <database> -h <host> -p <port> -f <backup-file>.sql
   ```

#### **Option 2: Structure Migration Using AWS Schema Conversion Tool (SCT)**

**Expected Time: 4 hours**

1. Open **SCT Tool** installed on your desktop or virtual desktop infrastructure (VDI).

2. **Configure New Project** in SCT.

3. **Configure Source Database Connection** (Oracle):

   * Input your Oracle connection details (hostname, port, username, password, SID/Service name).

4. **Create Mapping Between Oracle and PostgreSQL**:

   * Define mapping rules for data types, schemas, and object names.

5. **Configure Migration Rules**:

   * Configure object-level migration rules for things like constraints, indexes, and triggers.

6. **Select Objects to Migrate**:

   * Select tables, sequences, functions, procedures, packages, and other objects to migrate.

7. **Apply to Database**:

   * Apply the migration to the target PostgreSQL database.

8. **Validate Objects**:

   * Validate that the selected objects are correctly reflected in the PostgreSQL schema.

Repeat steps 6-8 for each table/object as necessary.

### Step 2. Data Migration (Using Ora2pg)

**Estimated Time: 4 Hours**

1. **Connect to the VM** where Ora2pg is installed.
    Create or define path
    ```bash
    mkdir -p /migration/<schema>
    BASE_DIR=/migration/<schema>
    ```

2. **Validate Ora2pg Tool Installation**
   Ensure that Ora2pg is correctly installed and that the database connectivity is established.

   ```bash
   ora2pg --version
   ```

3. **Configure the `ora2pg.conf` File**
   Update the configuration file with source Oracle database connection details and target PostgreSQL configuration.

   Example:

    ```bash
    # Oracle connection settings
    ORACLE_HOME    /usr/lib/oracle/12.2/client64
    ORACLE_DSN     dbi:Oracle:host=oracle_host;sid=oracle_sid;port=1521
    ORACLE_USER    <oracle_user>
    ORACLE_PWD     <oracle_password>

    # PostgreSQL connection settings
    PG_VERSION      16
    PG_DSN          dbi:Pg:host=pg_host;dbname=pg_db;port=5432
    PG_USER         <pg_user>
    PG_PWD          <pg_password>

    # Schema mapping configuration
    ORACLE_SCHEMA   <oracle-schema>
    DEFAULT_SCHEMA  <postgresql-schema>
    PG_SCHEMA       <postgresql-schema>

    # Data export settings
    TYPE            COPY

    # Parallelization & Performance
    EXPORT_SCHEMA    0
    PARALLEL_TABLES  8
    JOBS             4
    ORACLE_COPIES    4
    PARALLEL_PROCESS 1
    DATA_LIMIT       5000
    COMMIT_LEVEL     1000
    COPY_FREEZE      0
    COMPRESS         1
    TIMING           1
    QUIET            1
    USE_IDENTITY     1
    USE_ORACLE_PKEY_FOR_UNIQUE 1

    # Referential Integrity (Parent-Child)
    DROP_FKEY        0
    DISABLE_TRIGGERS 1
    DISABLE_SEQUENCE 1
    TRUNCATE_TABLE   1

    # Logging and debugging
    VERBOSE          1
    LOGFILE          dss.log
    DEBUG            1
    STOP_ON_ERROR    0
    MAX_ERROR        100
    ```

4. **Create tables.info**
   Create tables.info file which will be used in migration.sh to run the data migration for tables. 
   Note: Ensure you reference parent and child tables in sequence to avoide the error during migrations.

   ```bash
   vim tables.info
   TABLE_1
   TABLE_2
   PARENT_TABLE_3
   CHILD_TABLE_4
   TABLE_5
   ```

5. **Create and run migration script**
    Create and update the migration.sh file according to the path or environment

    ```bash
    #!/bin/bash
    set -euo pipefail

    # Paths
    BASE_DIR="/migration/<schema>"
    CONFIG="$BASE_DIR/ora2pg.conf"
    TABLES_FILE="$BASE_DIR/tables.info"
    OUT_DIR="$BASE_DIR/output"
    LOG_DIR="$BASE_DIR/logs"
    TIMESTAMP=$(date +%Y%m%d_%H%M%S)
    LOG="$LOG_DIR/migration_$TIMESTAMP.log"

    # Ensure directories exist
    mkdir -p "$OUT_DIR" "$LOG_DIR"

    # Read tables.info into comma-separated list (maintains order)
    TABLE_LIST=$(paste -sd, "$TABLES_FILE")

    # Function: Data migration (COPY) using your UAT-style command
    migrate_data() {
    echo "$(date) [INFO] Starting data migration" | tee -a "$LOG"
    ora2pg \
        -t COPY \
        -c "$CONFIG" \
        -n DSS \
        -a "$TABLE_LIST" \
        -P 8 \
        -j 4 \
        2>&1 | tee -a "$LOG"
    }

    # Main
    echo "$(date) [INFO] Migration started" | tee -a "$LOG"
    migrate_data
    echo "$(date) [INFO] Migration completed successfully" | tee -a "$LOG"
    ```
    This Script will use tables.info to migrate the data from the source (oracle) database to target (PostgreSQL) database you defined in ora2pg.conf file. Wait for the script execution gets completed. This will take time according to the no of rows in each tables.

5. **Create and Run Reconciliation Script**
   After the migration, run create the reconciliation.sh script to validate the data in both databases and ensure that row counts match:

    ```bash
    #!/bin/bash
    # reconciliation.sh
    # Compares record count, column count, index count, and constraint count
    # for each table listed in tables.info between Oracle (upper-case names)
    # and PostgreSQL (lower-case names in schema <postgres-schema>).

    set -euo pipefail

    # Connection strings (adjust as needed)
    ORACLE_CONN="<username>/<password>@<host>:<port>/<sid/service>"
    PG_CONN="host=<host> port=<port> dbname=<database> user=<user/role> password=<password>"

    TABLES_FILE="/migration/<schema>/tables.info"

    # Print header
    printf "%-30s %10s %10s %10s %10s %10s %10s %10s %10s\n" \
    "TABLE" \
    "ORA_ROWS" "PG_ROWS" \
    "ORA_COLS" "PG_COLS" \
    "ORA_IDX"  "PG_IDX" \
    "ORA_CONS" "PG_CONS"

    while IFS= read -r tbl; do
    # Oracle table name is upper-case
    ora_tbl="$tbl"
    # PostgreSQL table name is lowercase
    pg_tbl=$(echo "$tbl" | tr '[:upper:]' '[:lower:]')

    # Oracle: row count
    ora_rows=$(sqlplus -s "$ORACLE_CONN" <<-EOF
        SET PAGESIZE 0 FEEDBACK OFF VERIFY OFF HEADING OFF;
        SELECT COUNT(*) FROM DSS.$ora_tbl;
        EXIT;
    EOF
    )

    # Oracle: column count
    ora_cols=$(sqlplus -s "$ORACLE_CONN" <<-EOF
        SET PAGESIZE 0 FEEDBACK OFF VERIFY OFF HEADING OFF;
        SELECT COUNT(*) 
        FROM all_tab_cols 
        WHERE owner='DSS' 
        AND table_name=UPPER('$ora_tbl');
        EXIT;
    EOF
    )

    # Oracle: index count
    ora_idx=$(sqlplus -s "$ORACLE_CONN" <<-EOF
        SET PAGESIZE 0 FEEDBACK OFF VERIFY OFF HEADING OFF;
        SELECT COUNT(*) 
        FROM all_indexes 
        WHERE owner='DSS' 
        AND table_name=UPPER('$ora_tbl');
        EXIT;
    EOF
    )

    # Oracle: constraint count
    ora_cons=$(sqlplus -s "$ORACLE_CONN" <<-EOF
        SET PAGESIZE 0 FEEDBACK OFF VERIFY OFF HEADING OFF;
        SELECT COUNT(*) 
        FROM all_constraints 
        WHERE owner='DSS' 
        AND table_name=UPPER('$ora_tbl');
        EXIT;
    EOF
    )

    # PostgreSQL: row count
    pg_rows=$(psql "$PG_CONN" -t -c \
        "SELECT COUNT(*) 
        FROM <postgres-schema>.\"$pg_tbl\";"
    )

    # PostgreSQL: column count
    pg_cols=$(psql "$PG_CONN" -t -c \
        "SELECT COUNT(*) 
        FROM information_schema.columns 
        WHERE table_schema='<postgres-schema>' 
            AND table_name='$pg_tbl';"
    )

    # PostgreSQL: index count
    pg_idx=$(psql "$PG_CONN" -t -c \
        "SELECT COUNT(*) 
        FROM pg_indexes 
        WHERE schemaname='<postgres-schema>' 
            AND tablename='$pg_tbl';"
    )

    # PostgreSQL: constraint count
    pg_cons=$(psql "$PG_CONN" -t -c \
        "SELECT COUNT(*) 
        FROM information_schema.table_constraints 
        WHERE constraint_schema='<postgres-schema>' 
            AND table_name='$pg_tbl';"
    )

    # Trim whitespace
    ora_rows=${ora_rows//[[:space:]]/}
    ora_cols=${ora_cols//[[:space:]]/}
    ora_idx=${ora_idx//[[:space:]]/}
    ora_cons=${ora_cons//[[:space:]]/}
    pg_rows=${pg_rows//[[:space:]]/}
    pg_cols=${pg_cols//[[:space:]]/}
    pg_idx=${pg_idx//[[:space:]]/}
    pg_cons=${pg_cons//[[:space:]]/}

    # Print results
    printf "%-30s %10s %10s %10s %10s %10s %10s %10s %10s\n" \
        "$tbl" \
        "$ora_rows" "$pg_rows" \
        "$ora_cols" "$pg_cols" \
        "$ora_idx"  "$pg_idx" \
        "$ora_cons" "$pg_cons"

    done < "$TABLES_FILE"

    ```

4. **Validate the rowcount for tables**
   Chech the output of the reconciliation.sh, It will gives below information about source and target database for each table from tables.info
    ```bash
    record count 
    column count 
    index count 
    constraint count
    ```

# Known Issues and Solutions

1. **Manual Data Migration for Each Table**

   * Previously, the data migration was manual for each table. This process has now been automated with a script that migrates all tables in one go.

2. **Sequence Adjustment After Data Migration**

   * After migrating data, sequences may need to be adjusted to match the last value in the source Oracle database. Oracle sequences cannot be directly fetched using `SELECT sequence_name.nextval FROM dual`. Instead, fetch the last value using the `DBA_SEQUENCES` table and adjust the sequence.

   Example Query to Get the Last Value of a Sequence in Oracle:

   ```sql
   SELECT sequence_name, last_number FROM dba_sequences WHERE sequence_name = '<sequence_name>';
   ```

   Then, adjust the sequence in PostgreSQL:

   ```sql
   SELECT setval('<sequence_name>', <last_number>, true);
   ```

3. **Sequence Migration Issue**

   * Ensure that sequences are properly migrated and adjusted. If not, run the sequence adjustment query after the data migration process.

# Post-Migration Validation

1. **Validate Table Structures**:
   Check that all tables, indexes, and constraints have been correctly migrated to the PostgreSQL database.

2. **Data Validation**:
   Verify that data matches between the source Oracle and the target PostgreSQL database.

3. **Test Application**:
   Perform tests on the application that relies on the database to ensure it functions correctly with the new PostgreSQL backend.


# Additional Notes

* The time estimates for each stage are approximations and may vary depending on the size of the database and network latency.
* It is recommended to run the migration first in a test or UAT environment before performing it in production.


# GRANT AWS_ORACLE_EXTENSION

```sql
GRANT USAGE ON SCHEMA aws_oracle_ext TO <user>;
```

# PGCRON

please confirm the following configurations to ensure that **pg\_cron** can successfully run jobs on PostgreSQL

1. **Verify the following parameters in the `postgresql.conf` file:**

   ```
   shared_preload_libraries = 'pg_cron'
   cron.database_name = 'appdbuat' (database name on which scheduler/job will configur)
   ```

2. **Verify the following entry in the `pg_hba.conf` file** to allow connections only from the local server:

   ```
   host    all    all    127.0.0.1/32    trust
   ```

3. **Create the `pg_cron` extension** by connecting to the database specified in the `cron.database_name` parameter:

   ```sql
   CREATE EXTENSION pg_cron;
   ```

4. **Create job** 

   ```sql
   SELECT cron.schedule(
   'job_tag_as_expired',
   '*/15 * * * *',
   'CALL sp_tag_as_expired()'
   );
   ```

5. **Update job** you cannot directly update an existing job using a command like UPDATE. Instead, the typical approach is

      #### a. Find the job ID:

      ```sql
      SELECT jobid, schedule, command FROM cron.job WHERE jobname = 'job_tag_as_expired';
      ```

      #### b. Delete the existing job:

      ```sql
      SELECT cron.unschedule(jobid);
      ```

      #### c. Create a new job with updated:

      ```sql
      SELECT cron.schedule(
      'job_tag_as_expired',
      '*/15 * * * *',
      'CALL sp_tag_as_expired()'
      );
      ```
