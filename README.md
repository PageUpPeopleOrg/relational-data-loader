**THIS PROJECT IS ARCHIVED** We started adding a lot of custom code that was specific to our company infrastructure so it became not viable to support this as an opensource implementation.


# RelationalDataLoader

[![Build status](https://ci.appveyor.com/api/projects/status/l7l6are63ir2js4r?svg=true)](https://ci.appveyor.com/project/PageUp/relational-data-loader-fkwak)

## About

A utility for taking data from MS-SQL and loading it into PostgreSQL

## Usage

`py -m rdl --help`

```text
usage: py -m rdl process [-h] [-f [FORCE_FULL_REFRESH_MODELS]]
                           [-l [LOG_LEVEL]] [-p [AUDIT_COLUMN_PREFIX]]
                           source-connection-string
                           destination-connection-string configuration-folder

positional arguments:
  source-connection-string
                        The source connection string as either:
                         - (a) 64bit ODBC system dsn.
                           eg: `mssql+pyodbc://dwsource`.
                         - (b) AWS Lambda Function.
                           eg: `aws-lambda://tenant={databaseIdentifier};function={awsAccountNumber}:function:{functionName}`
  destination-connection-string
                        The destination database connection string. Provide in
                        PostgreSQL + Psycopg format. Eg: 'postgresql+psycopg2:
                        //username:password@host:port/dbname'
  configuration-folder  Absolute or relative path to the models. Eg
                        './models', 'C:/path/to/models'

optional arguments:
  -h, --help            show this help message and exit
  -f [FORCE_FULL_REFRESH_MODELS], --force-full-refresh-models [FORCE_FULL_REFRESH_MODELS]
                        Comma separated model names in the configuration
                        folder. These models would be forcefully refreshed
                        dropping and recreating the destination tables. All
                        others models would only be refreshed if required as
                        per the state of the source and destination tables. Eg
                        'CompoundPkTest,LargeTableTest'. Leave blank or use
                        glob (*) to force full refresh of all models.
  -l [LOG_LEVEL], --log-level [LOG_LEVEL]
                        Set the logging output level. ['CRITICAL', 'ERROR',
                        'WARNING', 'INFO', 'DEBUG']
  -p [AUDIT_COLUMN_PREFIX], --audit-column-prefix [AUDIT_COLUMN_PREFIX]
                        Set the audit column prefix, used in the destination
                        schema. Default is 'rdl_'.


usage: py -m rdl audit [-h] [-l [LOG_LEVEL]] [-p [AUDIT_COLUMN_PREFIX]]
                         destination-connection-string model-type timestamp

positional arguments:
  destination-connection-string
                        The destination database connection string. Provide in
                        PostgreSQL + Psycopg format. Eg: 'postgresql+psycopg2:
                        //username:password@host:port/dbname'
  model-type            Use the command FULL to return full refresh models or
                        the command INCR to return only the incremental models
                        since the timestamp
  timestamp             ISO 8601 datetime with timezone (yyyy-mm-
                        ddThh:mm:ss.nnnnnn+|-hh:mm) used to provide
                        information on all actions since the specified date.
                        Eg '2019-02-14T01:55:54.123456+00:00'.

optional arguments:
  -h, --help            show this help message and exit
  -l [LOG_LEVEL], --log-level [LOG_LEVEL]
                        Set the logging output level. ['CRITICAL', 'ERROR',
                        'WARNING', 'INFO', 'DEBUG']
  -p [AUDIT_COLUMN_PREFIX], --audit-column-prefix [AUDIT_COLUMN_PREFIX]
                        Set the audit column prefix, used in the destination
                        schema. Default is 'rdl_'.
```

_Notes:_

- `destination-connection-string` is a [PostgreSQL Db Connection String](http://docs.sqlalchemy.org/en/latest/dialects/postgresql.html#module-sqlalchemy.dialects.postgresql.psycopg2)

### Examples

See `./tests/integration_tests/test_*.cmd` scripts for usage samples.

## Development

### Alembic

#### To upgrade to the latest schema

```bash
alembic -c rdl/alembic.ini -x $DESTINATION_DB_URL upgrade head
```

#### Updating the schema

Ensure any new tables inherit from the same Base used in `alembic/env.py`

```python
from rdl.entities import Base
```

Whenever you make a schema change, run

```bash
pip install .
alembic -c rdl/alembic.ini -x $DESTINATION_DB_URL revision -m "$REVISION_MESSAGE" --autogenerate
```

check that the new version in `alembic/versions` is correct

#### Downgrading the schema

Whenever you want to downgrade the schema

```bash
alembic -c rdl/alembic.ini -x $DESTINATION_DB_URL history # see the list of revision ids
alembic -c rdl/alembic.ini -x $DESTINATION_DB_URL current # see the current revision id
alembic -c rdl/alembic.ini -x $DESTINATION_DB_URL downgrade -1 # revert back one revision
alembic -c rdl/alembic.ini -x $DESTINATION_DB_URL downgrade $revision_id # revert back to a revision id, found using the history command
```

#### Inaccurate autogenerated revisions

Does your autogenerated revision not look right?

Try editing the function `use_schema` in `alembic/env.py`, this determines what alembic looks for in the database.

[Relevant Documentation](https://alembic.sqlalchemy.org/en/latest/api/runtime.html?highlight=include_schemas#alembic.runtime.environment.EnvironmentContext.configure.params.include_object)

#### New models aren't showing up in upgrade section

Ensure all model classes inherit from the same Base that `alembic/env.py` imports, and that the following class
properties are set

```python
__tablename__ = 'your_mapped_table_name'
__table_args__ = {'schema': Constants.DATA_PIPELINE_EXECUTION_SCHEMA_NAME}
```

Also try importing the models into `alembic/env.py`, eg

```python
from rdl.data_load_tracking import DataLoadExecution
```

#### Alembic won't pick up my change

[Alembic only supports some changes](https://alembic.sqlalchemy.org/en/latest/autogenerate.html#what-does-autogenerate-detect-and-what-does-it-not-detect)

Try adding raw sql in the `upgrade()` and `downgrade()` functions of your revision

```python
op.execute(RAW_SQL)
```

### Linting

Use autopep8 before pushing commits (include the "." for the folder)

`>>>autopep8 --in-place --aggressive --aggressive --recursive --max-line-length=120 .`

Use the following vscode settings by either:

- System wide: Ctrl+Shift+P, "open settings (json)"
- For this project only: add the following to ./vscode/settings.json

```json
{
    "python.pythonPath": (your python venv here),
    "files.trimFinalNewlines": true,
    "files.trimTrailingWhitespace": true,
    "files.insertFinalNewline": true,
    "files.eol": "\n",
    "editor.renderWhitespace": "all",
    "editor.tabCompletion": "on",
    "editor.trimAutoWhitespace": true,
    "editor.insertSpaces": true,
    "editor.rulers": [80, 120],
    "editor.formatOnSave": true
}
```

### Testing

#### Integration

##### Pre-requisites

- PostgreSQL 10+ and SQL2016 _(even SQL Express would do, you'd just have to find and replace connection string references for SQL2016 with SQLEXPRESS)_ are installed

- The test batch files assume there is a user by the name of `postgres` on the system.
  It also sends through a nonsense password -- it is assumed that the target system is running in 'trust' mode.

  _See [Postgres docs](https://www.postgresql.org/docs/9.1/static/auth-pg-hba-conf.html) for details on trust mode._


##### Setup

Run the below scripts on SQL Server to setup a source database.

```
./tests/integration_tests/mssql_source/source_database_setup/create_database.sql
./tests/integration_tests/mssql_source/source_database_setup/create_large_table.sql
./tests/integration_tests/mssql_source/source_database_setup/create_compound_pk.sql
```

Run the below scripts on PostgreSQL Server to setup a target database.

```
./tests/integration_tests/psql_destination/create-db.sql
./tests/integration_tests/psql_destination/setup-db.sql
```

##### Execution

Run the different `*.cmd` test files at `./tests/integration_tests/` location. `test_full_refresh_from_mssql.cmd` is a good start.

#### Unit

_Setup:_

Create a new SQL Server Login/User using the script below. Make sure you update it with your desired password and if you update the username/login, please ensure the changes are reflected in: `./tests/unit_tests/config/connection.json`, which can be created using `./tests/unit_tests/config/connection.json.template`.

```sql
USE master;
GO
IF NOT EXISTS(SELECT * FROM sys.syslogins WHERE NAME = 'rdl_test_user')
    CREATE LOGIN [rdl_test_user] WITH PASSWORD=N'hunter2', CHECK_EXPIRATION=OFF, CHECK_POLICY=OFF;
GO
ALTER SERVER ROLE [dbcreator] ADD MEMBER [rdl_test_user]
GO
```

_Execution:_

Execution is as simply as `python3 run_tests.py`

### Postgres debugging

Ensure the database you are using is in utf8 mode. You cannot change encoding once the database is created.

```sql

CREATE DATABASE "my_database"
    WITH OWNER "postgres"
    ENCODING 'UTF8'
    TEMPLATE template0;

```

### `Destination.Type` Values

The destination.type value controls both the data reader type and the destination column type. These are implemented in ColumnTypeResolver.py.

They are mapped as follows:

| destination.type            | pandas type | sqlalchemy type                      | dw column type | notes                                                                  |
| --------------------------- | ----------- | ------------------------------------ | -------------- | ---------------------------------------------------------------------- |
| string                      | str         | citext.CIText                        | citext         | A case-insensitive string that supports unicode                        |
| int (when nullable = false) | int         | sqlalchemy.Integer                   | int            | An (optionally) signed INT value                                       |
| int (when nullable = true)  | object      | sqlalchemy.Integer                   | int            | An (optionally) signed INT value                                       |
| datetime                    | str         | sqlalchemy.DateTime                  | datetime (tz?) |                                                                        |
| json                        | str         | sqlalchemy.dialects.postgresql.JSONB | jsonb          | Stored as binary-encoded json on the database                          |
| numeric                     | float       | sqlalchemy.Numeric                   | numeric        | Stores whole and decimal numbers                                       |
| guid                        | str         | sqlalchemy.dialects.postgresql.UUID  | uuid           |                                                                        |
| bigint                      | int         | sqlalchemy.BigInteger                | BigInt         | Relies on 64big python. Limited to largest number of ~2147483647121212 |
| boolean                     | bool        | sqlalchemy.Boolean                   | Boolean        |                                                                        |
