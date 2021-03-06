# DB Templates Using Flyway

[![MIT License](https://img.shields.io/github/license/Scott-Lau/db-templates-flyway-mysql)][license]

This is a template project for MySQL database scripts management using [Flyway][flyway].

## Documentation

This documentation is also available in Simplified Chinese [中文版链接][readme_zh_cn].

The following documentation is excerpted from [Flyway][flyway_doc] official site with the exception of *removing undo section*.

### Migrations

#### Overview

With Flyway all changes to the database are called **migrations**. Migrations can be either *versioned* or
*repeatable*.

**Versioned migrations** have a *version*, a *description* and a *checksum*. The version must be unique. The description is purely
informative for you to be able to remember what each migration does. The checksum is there to detect accidental changes.
Versioned migrations are the most common type of migration. They are applied in order exactly once.

**Repeatable migrations** have a description and a checksum, but no version. Instead of being run just once, they are
(re-)applied every time their checksum changes.

Within a single migration run, repeatable migrations are always applied last, after all pending versioned migrations 
have been executed. Repeatable migrations are applied in the order of their description.

By default both versioned and repeatable migrations can be written either in **SQL** or in **Java** and can consist of multiple statements.

Flyway automatically discovers migrations on the *filesystem* and on the Java *classpath*.

To keep track of which migrations have already been applied when and by whom, Flyway adds a *schema history table*
to your schema.

#### Versioned Migrations

The most common type of migration is a **versioned migration**. Each versioned migration has a *version*, a *description*
and a *checksum*. The version must be unique. The description is purely
informative for you to be able to remember what each migration does. The checksum is there to detect accidental changes.
Versioned migrations are applied in order exactly once.

Versioned migrations are typically used for:
- Creating/altering/dropping tables/indexes/foreign keys/enums/UDTs/...
- Reference data updates
- User data corrections

Here is a small example:

```sql
CREATE TABLE car (
    id INT NOT NULL PRIMARY KEY,
    license_plate VARCHAR NOT NULL,
    color VARCHAR NOT NULL
);

ALTER TABLE owner ADD driver_license_id VARCHAR;

INSERT INTO brand (name) VALUES ('DeLorean');
```

Each versioned migration must be assigned a **unique version**. Any version is valid as long as it conforms to the usual
dotted notation. For most cases a simple increasing integer should be all you need. However Flyway is quite flexible and
all these versions are valid versioned migration versions:
- 1
- 001
- 5.2
- 1.2.3.4.5.6.7.8.9
- 205.68
- 20130115113556
- 2013.1.15.11.35.56
- 2013.01.15.11.35.56

Versioned migrations are applied in the order of their versions. Versions are sorted numerically as you would normally expect.

#### Repeatable Migrations

**Repeatable migrations** have a description and a checksum, but no version. Instead of being run just once, they are
(re-)applied every time their checksum changes. 

This is very useful for managing database objects whose definition can then simply be maintained in a single file in
version control. They are typically used for
- (Re-)creating views/procedures/functions/packages/...
- Bulk reference data reinserts

Within a single migration run, repeatable migrations are always applied last, after all pending versioned migrations have been executed. Repeatable migrations are applied in the order of their description.

It is your responsibility to ensure the same repeatable migration can be applied multiple times. This usually
involves making use of `CREATE OR REPLACE` clauses in your DDL statements.

Here is an example of what a repeatable migration looks like:

```sql
CREATE OR REPLACE VIEW blue_cars AS 
    SELECT id, license_plate FROM cars WHERE color='blue';
```

#### SQL-based migrations

Migrations are most commonly written in **SQL**. This makes it easy to get started and leverage any existing scripts,
tools and skills. It gives you access to the full set of capabilities of your database and eliminates the need to
understand any intermediate translation layer. 

SQL-based migrations are typically used for
- DDL changes (CREATE/ALTER/DROP statements for TABLES,VIEWS,TRIGGERS,SEQUENCES,...)
- Simple reference data changes (CRUD in reference data tables)
- Simple bulk data changes (CRUD in regular data tables)

##### Naming

In order to be picked up by Flyway, SQL migrations must comply with the following naming pattern:

###### Versioned Migrations

Prefix Version Separator Description Suffix

Such as: V2__Add_new_table.sql, in which 'V' is prefix, '2' is version, '\_\_' is Separator, 'Add_new_table' is description and '.sql' is suffix.

###### Repeatable Migrations

Prefix Separator Description Suffix

Such as: R__Create_view.sql, in which 'R' is prefix, '\_\_' is Separator, 'Create_view' is description and '.sql' is suffix.


The file name consists of the following parts:
- **Prefix**: `V` for versioned (*configurable*),
`U` for undo (*configurable*) and
`R` for repeatable migrations (*configurable*)
- **Version**: Version with dots or underscores separate as many parts as you like (**Not for repeatable migrations**)
- **Separator**: `__` (two underscores) (*configurable*)
- **Description**: Underscores or spaces separate the words
- **Suffix**: `.sql` (*configurable*)

Optionally versioned SQL migrations can also omit both the separator and the description.

The configuration option <code>validateMigrationNaming</code> determines how Flyway handles files that do not correspond with
the naming pattern when carrying out a migration: if true then Flyway will simply ignore all such files, if false then 
Flyway will fail fast and list all files which need to be corrected.

##### Discovery

Flyway discovers SQL-based migrations from one or more directories referenced by the `locations`
property.

- Unprefixed locations or locations with the `classpath:` prefix target the Java classpath.
- Locations with the `filesystem:` prefix search the file system.
- Locations with the `s3:` prefix search AWS S3 buckets. To use AWS S3, the [AWS SDK v2](https://mvnrepository.com/artifact/software.amazon.awssdk/services) and dependencies must be included, and [configured](https://docs.aws.amazon.com/sdk-for-java/v1/developer-guide/credentials.html) for your S3 account.
- Locations with the `gcs:` prefix search Google Cloud Storage buckets. To use GCS, the [GCS library](https://cloud.google.com/storage/docs/reference/libraries#auth-cloud-implicit-java) must be included, and the GCS environment variable <code>GOOGLE_APPLICATION_CREDENTIALS</code> must be set to the credentials file for the service account that has access to the bucket.

<pre class="filetree"><i class="fa fa-folder-open"></i> my-project
  <i class="fa fa-folder-open"></i> src
    <i class="fa fa-folder-open"></i> main
      <i class="fa fa-folder-open"></i> resources
        <span><i class="fa fa-folder-open"></i> db
  <i class="fa fa-folder-open"></i> migration</span>                <i class="fa fa-long-arrow-left"></i> <code>classpath:db/migration</code> 
            <i class="fa fa-file-text"></i> R__My_view.sql
            <i class="fa fa-file-text"></i> U1.1__Fix_indexes.sql
            <i class="fa fa-file-text"></i> U2__Add a new table.sql
            <i class="fa fa-file-text"></i> V1__Initial_version.sql
            <i class="fa fa-file-text"></i> V1.1__Fix_indexes.sql
            <i class="fa fa-file-text"></i> V2__Add a new table.sql
  <span><i class="fa fa-folder-open"></i> my-other-folder</span>                  <i class="fa fa-long-arrow-left"></i> <code>filesystem:/my-project/my-other-folder</code>
    <i class="fa fa-file-text"></i> U1.2__Add_constraints.sql
    <i class="fa fa-file-text"></i> V1.2__Add_constraints.sql</pre>
New SQL-based migrations are **discovered automatically** through filesystem and Java classpath scanning at runtime.
Once you have configured the `locations` you want to use, Flyway will
automatically pick up any new SQL migrations as long as they conform to the configured *naming convention*.

This scanning is recursive. All migrations in non-hidden directories below the specified ones are also picked up.

##### Syntax

Flyway supports all regular SQL syntax elements including:
- Single- or multi-line statements
- Single- (--) or Multi-line (/* */) comments spanning complete lines
- Database-specific SQL syntax extensions (PL/SQL, T-SQL, ...) typically used to define stored procedures, packages, ...

Additionally in the case of Oracle, Flyway also supports *SQL\*Plus commands*.

##### Placeholder Replacement
In addition to regular SQL syntax, Flyway also supports placeholder replacement with configurable pre- and suffixes.
By default it looks for Ant-style placeholders like `${myplaceholder}`. This can be very useful to abstract differences between environments.

#### Java-based migrations

Java-based migrations are a great fit for all changes that can not easily be expressed using SQL.

These would typically be things like
- BLOB &amp; CLOB changes
- Advanced bulk data changes (Recalculations, advanced format changes, ...)

##### Naming

In order to be picked up by Flyway, Java-based Migrations must implement the
`JavaMigration` interface. Most users
however should inherit from the convenience class `BaseJavaMigration`
instead as it encourages Flyway's default naming convention, enabling Flyway to automatically extract the version and
the description from the class name. To be able to do so, the class name must comply with the following naming pattern:

###### Versioned Migrations

Prefix Version Separator Description

Such as: V2__Add_new_table.sql, in which 'V' is prefix, '2' is version, '\_\_' is Separator and 'Add_new_table' is description.

###### Repeatable Migrations

Prefix Separator Description

Such as: R__Create_view.sql, in which 'R' is prefix, '\_\_' is Separator and 'Create_view' is description.


The class name consists of the following parts:
- **Prefix**: `V` for versioned migrations, `U` for undo migrations, `R` for repeatable migrations
- **Version**: Underscores (automatically replaced by dots at runtime) separate as many parts as you like (Not for repeatable migrations)
- **Separator**: `__` (two underscores)
- **Description**: Underscores (automatically replaced by spaces at runtime) separate the words

If you need more control over the class name, you can override the default convention by implementing the
`JavaMigration` interface directly.

This will allow you to name your class as you wish. Version, description and migration category are provided by
implementing the respective methods.

##### Discovery

Flyway discovers Java-based migrations on the Java classpath in the packages referenced by the
`locations` property.

<pre class="filetree"><i class="fa fa-folder-open"></i> my-project
  <i class="fa fa-folder-open"></i> src
    <i class="fa fa-folder-open"></i> main
      <i class="fa fa-folder-open"></i> java
        <span><i class="fa fa-folder-open"></i> db
  <i class="fa fa-folder-open"></i> migration</span>            <i class="fa fa-long-arrow-left"></i> <code>classpath:db/migration</code> 
            <i class="fa fa-file-text"></i> R__My_view
            <i class="fa fa-file-text"></i> U1_1__Fix_indexes
            <i class="fa fa-file-text"></i> V1__Initial_version
            <i class="fa fa-file-text"></i> V1_1__Fix_indexes
  <i class="fa fa-file-text"></i> pom.xml</pre>

New java migrations are **discovered automatically** through classpath scanning at runtime. The
scanning is recursive. Java migrations in subpackages of the specified ones are also picked up.

##### Checksums and Validation

Unlike SQL migrations, Java migrations by default do not have a checksum and therefore do not participate in the
change detection of Flyway's validation. This can be remedied by implementing the 
`getChecksum()` method, which you can then use to provide your own checksum, which will then be
stored and validated for changes.

##### Sample Class
```java
package db.migration;

import org.flywaydb.core.api.migration.BaseJavaMigration;
import org.flywaydb.core.api.migration.Context;
import java.sql.PreparedStatement;

/**
 * Example of a Java-based migration.
 */
public class V1_2__Another_user extends BaseJavaMigration {
    public void migrate(Context context) throws Exception {
        try (PreparedStatement statement = 
                 context
                     .getConnection()
                     .prepareStatement("INSERT INTO test_user (name) VALUES ('Obelix')")) {
            statement.execute();
        }
    }
}
```

Take care that your Java migration does not close the database connection, either explicitly or as a
result of a try-with-resources statement.

##### Spring

If your application already uses Spring and you do not want to use JDBC directly you can easily use Spring JDBC's
`JdbcTemplate` instead:

```java
package db.migration;

import org.flywaydb.core.api.migration.BaseJavaMigration;
import org.flywaydb.core.api.migration.Context;
import java.sql.PreparedStatement;

/**
 * Example of a Java-based migration using Spring JDBC.
 */
public class V1_2__Another_user extends BaseJavaMigration {
    public void migrate(Context context) {
        new JdbcTemplate(new SingleConnectionDataSource(context.getConnection(), true))
                .execute("INSERT INTO test_user (name) VALUES ('Obelix')");
    }
}
```

#### Transactions

By default, Flyway always wraps the execution of an entire migration within a single transaction.

Alternatively you can also configure Flyway to wrap the entire execution of all migrations of a single migration run
within a single transaction by setting the `group` property to `true`.

If Flyway detects that a specific statement cannot be run within a transaction due to technical limitations of your
database, it won't run that migration within a transaction. Instead it will be marked as *non-transactional*.

By default transactional and non-transactional statements cannot be mixed within a migration run. You can however allow
this by setting the `mixed` property to `true`. Note that this is only
applicable for PostgreSQL, Aurora PostgreSQL, SQL Server and SQLite which all have statements that do not run at all
within a transaction. This is not to be confused with implicit transaction, as they occur in MySQL or Oracle, where even
though a DDL statement was run within within a transaction, the database will issue an implicit commit before and after
its execution.

##### Manual override

If necessary, you can manually determine whether or not to execute a migration in a transaction. This is useful for
databases like PostgreSQL and SQL Server where certain statements can only execute outside a transaction.

For Java migrations, the `JavaMigration` interface has a method `canExecuteInTransaction`. This determines whether the execution should take place inside a transaction. You can rely on `BaseJavaMigration`'s default behavior to return `true` or override `canExecuteInTransaction` to execute certain migrations outside a transaction by returning `false`.

For SQL migrations, you can specify the script configuration property `executeInTransaction`.
##### Important Note

If your database cleanly supports DDL statements within a transaction, failed migrations will always be rolled back
(unless they were marked as non-transactional).

If on the other hand your database does NOT cleanly supports DDL statements within a transaction (by for example
issuing an implicit commit before and after every DDL statement), Flyway won't be able to perform a clean rollback in
case of failure and will instead mark the migration as failed, indicating that some manual cleanup may be required.
You may also need to run repair to remove the failed migration entry from the schema history table.

## Query Results

Migrations are primarily meant to be executed as part of release and deployment automation processes and there is rarely
the need to visually inspect the result of SQL queries.

There are however some scenarios where such manual inspection makes sense, and therefore Flyway will display query results in the usual tabular form when a `SELECT` statement (or any other statement that returns results) is executed.

## Schema History Table

To keep track of which migrations have already been applied when and by whom, Flyway adds a special
**schema history table** to your schema. You can think of this table as a complete audit trail of all changes
performed against the schema. It also tracks migration checksums and whether or not the migrations were successful.

#### Schema creation

By default, Flyway will attempt to create the schemas provided by the `schemas` and `defaultSchema` configuration options. This behavior can be toggled with the `createSchemas` configuration option.

This might be useful when you want complete control over how schemas are created.

##### The `createSchemas` option and the Schema History Table

Flyway requires a schema for the schema history table to reside in before running a migration. When `createSchemas` is `false`, it will be impossible for the schema history table to be created, unless a schema already exists for it to reside in.

So, given a configuration like this:

```
flyway.createSchemas=false
flyway.schemas=my_schema
```

The following can happen if `createSchemas` is `false`:

- Run migrate
- `my_schema` *is not* created by Flyway
- Because `my_schema` is the default schema, Flyway attempts to create the schema history table in `my_schema`
- `my_schema` does not exist, so the operation fails

Therefore, when toggling `createSchemas` to `false`, the following setup is recommended:

- Set the default schema to `flyway_history_schema`
  - Either by setting `defaultSchema`, or placing it first in the `schemas` configuration option
- Set `initSql` to create `flyway_history_schema` if it doesn't exist
- Place your other schemas in the `schemas` property

So, given a configuration like this:

```
flyway.createSchemas=false
flyway.initSql=CREATE IF NOT EXISTS flyway_history_schema
flyway.schemas=flyway_history_schema,my_schema
```

The following will happen:

- Run migrate
- `initSql` is executed, so `flyway_history_schema` is created
- Because `flyway_history_schema` is the default schema, Flyway attempts to create the schema history table in `flyway_history_schema`
- `my_schema` *is not* created by Flyway
- Migrations run as normal
- Migrations are free to control creation of `my_schema`

#### Migration States

Migrations are either *resolved* or *applied*. Resolved migrations have been detected by Flyway's filesystem
and classpath scanner. Initially they are **pending**. Once they are executed against the database,
they become applied.

When the migration *succeeds* it is marked as **success** in Flyway's *schema history table*.

When the migration *fails* and the database supports *DDL transactions*, the migration is *rolled back* and
nothing is recorded in the schema history table.

When the migration *fails* and the database doesn't support *DDL transactions*, the migration
is marked as **failed** in the schema history table, indicating manual database cleanup may be required.

Versioned migrations whose effects have been undone by an undo migration are marked as **undone**.

Repeatable migrations whose checksum has changed since they are last applied are marked as **outdated** until
they are executed again. Note also that changing the value of placeholders will cause repeatable migrations to be considered **outdated**.

When Flyway discovers an applied versioned migration with a version that is higher than the highest known version
(this happens typically when a newer version of the software has migrated that schema), that migration is marked as **future**.

When a migration is not found on disk, but is found in the schema history, it is marked as **missing**. By default these will cause *validate* to fail, however they can be marked as **deleted** by using *repair*.

When a migration has had its state changed to deleted by *repair* it is marked as **deleted**.

When using *cherryPick*, if the migration is not in the cherry picked list then it is marked as **ignored**.


License
-------
The project is released under the MIT License. The [MIT license][license] is registered with and approved by the 
[Open Source Initiative][osi].


[home]: https://github.com/Scott-Lau/db-templates-flyway-mysql
[license]: https://opensource.org/licenses/MIT
[osi]: https://opensource.org/
[flyway]: https://flywaydb.org/
[flyway_doc]: https://flywaydb.org/documentation/concepts/migrations
[readme]: https://github.com/Scott-Lau/db-templates-flyway-mysql/blob/master/README.md
[readme_zh_cn]: https://github.com/Scott-Lau/db-templates-flyway-mysql/blob/master/README_zh_cn.md