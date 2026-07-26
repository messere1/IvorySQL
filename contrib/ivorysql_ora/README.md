# ivorysql_ora

`ivorysql_ora` is the Oracle compatibility extension that ships with IvorySQL.
It provides the Oracle-compatible data types, built-in functions, built-in
package subsets and data-dictionary views that the Oracle parser
(`liboracle_parser`) and PL/iSQL rely on.

All objects are created in the `sys` schema. The extension is declared
`superuser = true` and `relocatable = false` in
[`ivorysql_ora.control`](ivorysql_ora.control), so it must be installed by a
superuser and cannot be moved to another schema.

## Installing

### New cluster

`initdb` runs in Oracle mode by default (`-m oracle`). In that mode it adds

```
shared_preload_libraries = 'liboracle_parser, ivorysql_ora'
```

to `ivorysql.conf` and creates the extension in every database cloned from
`template1`, so a freshly initialized cluster needs no extra step:

```sh
initdb -D /path/to/data -m oracle
```

### Existing cluster

A cluster created with `initdb -m pg` does not preload the Oracle parser and
does not have the extension installed. To add it:

1. Add the libraries to `postgresql.conf` (or to an included `ivorysql.conf`)
   and restart the server:

   ```
   shared_preload_libraries = 'liboracle_parser, ivorysql_ora'
   ```

2. Create the extension in each database that needs it, with the session
   switched to the Oracle dialect — the installation script uses Oracle syntax
   such as `CREATE PACKAGE`:

   ```sql
   SET ivorysql.compatible_mode TO oracle;
   CREATE EXTENSION ivorysql_ora;
   ```

Note that `initdb -m oracle` also loads
[`preload_ora_misc.sql`](preload_ora_misc.sql), which creates objects that are
*not* part of the extension (for example `sys.dual` and the `rowid` types).
Those objects are only available in a cluster initialized in Oracle mode.

## Feature groups

| Group | Contents | Source |
| --- | --- | --- |
| Data types | `char(n char)`, `char(n byte)`, `varchar2(n char)`, `varchar2(n byte)`, `date`, `timestamp`, `timestamp with time zone`, `timestamp with local time zone`, `interval year to month`, `interval day to second`, `raw`/`long`, `binary_float`, `binary_double`, plus the Oracle type-precedence rules | [`src/datatype`](src/datatype) |
| Built-in functions | Character, datetime, numeric and miscellaneous Oracle functions | [`src/builtin_functions`](src/builtin_functions) |
| Built-in packages | `DBMS_OUTPUT`, `DBMS_LOCK`, `DBMS_SESSION`, `DBMS_UTILITY`, `UTL_ENCODE`, `UTL_FILE`, `UTL_RAW` | [`src/builtin_packages`](src/builtin_packages) |
| Data dictionary views | `ALL_`/`DBA_`/`USER_` views for procedures, source, arguments, sequences, views, constraint columns and table columns, plus `V$SESSION`, `V$PROCESS` and `V$PARAMETER` | [`src/sysview`](src/sysview) |
| `MERGE` | Oracle-compatible `MERGE` behaviour, installed through planner/executor hooks | [`src/merge`](src/merge) |
| XML functions | `updatexml` and related helpers | [`src/xml_functions`](src/xml_functions) |

The SQL for the extension is assembled at build time by
[`gensql.pl`](gensql.pl) from the per-directory `*--1.0.sql` files listed in
[`ivorysql_ora_merge_sqls`](ivorysql_ora_merge_sqls).

## Implemented package subsets

Only the subprograms listed below are implemented. Anything else from the
corresponding Oracle package is currently unsupported.

| Package | Subprograms | Definition |
| --- | --- | --- |
| `DBMS_OUTPUT` | `ENABLE`, `DISABLE`, `PUT`, `PUT_LINE`, `NEW_LINE`, `GET_LINE`, `GET_LINES` | [`dbms_output--1.0.sql`](src/builtin_packages/dbms_output/dbms_output--1.0.sql) |
| `DBMS_LOCK` | `ALLOCATE_UNIQUE`, `REQUEST`, `RELEASE`, `SLEEP`, plus the `S_MODE`/`X_MODE` and return-code constants | [`dbms_lock--1.0.sql`](src/builtin_packages/dbms_lock/dbms_lock--1.0.sql) |
| `DBMS_SESSION` | `SET_CONTEXT`, `CLEAR_CONTEXT`, `CLEAR_ALL_CONTEXT`, `RESET_PACKAGE` | [`dbms_session--1.0.sql`](src/builtin_packages/dbms_session/dbms_session--1.0.sql) |
| `DBMS_UTILITY` | `FORMAT_ERROR_BACKTRACE`, `FORMAT_ERROR_STACK`, `FORMAT_CALL_STACK` | [`dbms_utility--1.0.sql`](src/builtin_packages/dbms_utility/dbms_utility--1.0.sql) |
| `UTL_ENCODE` | `BASE64_ENCODE`, `BASE64_DECODE` | [`utl_encode--1.0.sql`](src/builtin_packages/utl_encode/utl_encode--1.0.sql) |
| `UTL_FILE` | `FOPEN`, `FOPEN_NCHAR`, `FCLOSE`, `FCLOSE_ALL`, `FCOPY`, `FFLUSH`, `FGETATTR`, `FGETPOS`, `FREMOVE`, `FRENAME`, `FSEEK`, `GET_LINE`, `GET_LINE_NCHAR`, `IS_OPEN`, `NEW_LINE`, `PUT`, `PUTF`, `PUT_LINE`, `PUT_LINE_NCHAR`, `PUT_RAW` | [`utl_file--1.0.sql`](src/builtin_packages/utl_file/utl_file--1.0.sql) |
| `UTL_RAW` | `CAST_TO_RAW`, plus the `BIG_ENDIAN`/`LITTLE_ENDIAN`/`MACHINE_ENDIAN` constants | [`utl_raw--1.0.sql`](src/builtin_packages/utl_raw/utl_raw--1.0.sql) |

`DBMS_SESSION.LIST_CONTEXT` is an exception to the package interface: Oracle
declares it with an `OUT` collection argument, which a PL/iSQL package cannot
return, so the listing is exposed as a set-returning function instead:

```sql
SELECT * FROM sys.dbms_session_list_context() ORDER BY namespace, attribute;
```

## Examples

Application context, readable through `SYS_CONTEXT`:

```sql
CALL dbms_session.set_context('app_ctx', 'user_id', '42');
SELECT sys_context('app_ctx', 'user_id');
 sys_context
-------------
 42

CALL dbms_session.clear_all_context('app_ctx');
```

Namespace and attribute names are folded to upper case, as in Oracle, so
`sys_context('APP_CTX', 'USER_ID')` reads the same entry. `SYS_CONTEXT` returns
`NULL` — rather than raising an error — for an unknown namespace or attribute,
which makes it usable inside row-level security predicates.

File I/O through a directory object:

```sql
INSERT INTO sys.utl_file_directory(dirname, dir) VALUES ('work_dir', '/var/tmp/ivorysql');

DECLARE
    f sys.ora_utl_file_file_type;
BEGIN
    f := utl_file.fopen('work_dir', 'report.txt', 'w');
    utl_file.put_line(f, 'hello');
    utl_file.fclose(f);
END;
/
```

Base64 helpers:

```sql
SELECT utl_encode.base64_encode(utl_raw.cast_to_raw('ivorysql'));
```

## Session reset

The extension installs a `ProcessUtility` hook so that `DISCARD ALL` and
`DISCARD PACKAGES` reset extension-owned session state: the `DBMS_OUTPUT`
buffer and every `DBMS_SESSION` application context entry are cleared after the
command completes. `DBMS_SESSION.RESET_PACKAGE` performs the same context reset
on its own.

This matters for connection pools: a pooled backend that is not reset keeps the
application context of the previous client, and any row-level security policy
built on `SYS_CONTEXT` would then be evaluated with stale values. Issue
`DISCARD ALL` (or at least `DBMS_SESSION.RESET_PACKAGE`) when returning a
connection to the pool.

## Security boundaries

* **Installation is superuser-only.** The extension is marked
  `superuser = true`, and it creates C functions in `sys`.
* **`UTL_FILE` is confined to an allow-list.** Every path is canonicalized and
  then checked against the `sys.utl_file_directory` table, which maps a
  directory-object name (`dirname`) to a filesystem path (`dir`); a path that is
  not under a listed directory is rejected. The table is readable but not
  writable by `PUBLIC`, so only a privileged role can widen the allow-list.
  Note that files are read and written by the *server* process, with the
  privileges of the operating-system user running the server — treat a
  directory entry as a grant of that access to every database user.
* **`utl_file.umask` is superuser-only.** It defaults to `0077`, so files
  created by `UTL_FILE.FOPEN` are not readable by other operating-system users
  unless a superuser relaxes it.
* **Application context is session-local and unauthenticated.** Any user can
  call `DBMS_SESSION.SET_CONTEXT` for any namespace; unlike Oracle there is no
  trusted package that owns a namespace. Do not treat `SYS_CONTEXT` values as a
  security boundary against the database user who sets them — use them to carry
  application identity, and keep the pooling caveat above in mind.

## Building and testing

The extension is built and installed together with the rest of the tree:

```sh
./configure --prefix=/path/to/install
make
make install
```

To build only this directory against an installed server, use PGXS:

```sh
make -C contrib/ivorysql_ora USE_PGXS=1
```

The regression tests run in Oracle mode and therefore use the `oracle-check`
target rather than `check`. From an in-tree build:

```sh
make -C contrib/ivorysql_ora oracle-check
```

A PGXS build does not support `oracle-check`; install the module first and then
run the suites against a running Oracle-mode server:

```sh
make -C contrib/ivorysql_ora USE_PGXS=1 install
make -C contrib/ivorysql_ora USE_PGXS=1 oracle-installcheck
```

The suites are listed in `ORA_REGRESS` in the [`Makefile`](Makefile); inputs
live in [`sql/`](sql) and the expected outputs in [`expected/`](expected).
