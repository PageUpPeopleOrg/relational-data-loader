# RelationalDataLoader
## About
A utility for taking data from MS-SQL and loading it into PostgeSQL


## Usage
Execute  `py rdl.py SOURCE DESTINATION CONFIGURATION-FOLDER [log-level] [full-refresh]`

Where source takes the following formats
** CSV **  `csv://.\test_data\full_refresh`
** MSSQL **  `mssql+pyodbc://dwsource`

In the above example, dwsource is a 64bit ODBC system dsn


Destination takes the following format
** PostgreSQL **  postgresql+psycopg2://postgres:xxxx@localhost/dest_dw


### Examples
#### CSV Source

`py rdl.py csv://.\test_data\full_refresh postgresql+psycopg2://postgres:xxxx@localhost/dest_dw .\configuration\ --log-level INFO --full-refresh yes`
`py rdl.py csv://.\test_data\incremental_refresh postgresql+psycopg2://postgres:xxxx@localhost/dest_dw .\configuration\ --log-level INFO --full-refresh no`


#### MSSQL Source



###Troubleshooting
Run with  `--log-level DEBUG` on the command line.


##Other Notes
###Testing
The test batch files assume there is a user by the name of `postgres` on the system.
It also sends though a nonense password - it is assumed that the target system is running in 'trust' mode.
See https://www.postgresql.org/docs/9.1/static/auth-pg-hba-conf.html for details on trust mode