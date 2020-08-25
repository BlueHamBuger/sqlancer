# SQLancer for Citus (PostgreSQL extension)

SQLancer (Synthesized Query Lancer) is a tool to automatically test Database Management Systems (DBMS) in order to find logic bugs in their implementation. More information about the tool can be found in the [SQLancer README](https://github.com/sqlancer/sqlancer).

The Citus implementation of SQLancer supports the Ternary Logic Query Partitioning (TLP) test oracle.

# Setting up

Instructions for setting up SQLancer are described in [SQLancer - Getting Started](https://github.com/sqlancer/sqlancer#getting-started).

Requirements for Citus:
* PostgreSQL & Citus - The steps required to build Citus from source are described in [Contributing to Citus](https://github.com/citusdata/citus/blob/master/CONTRIBUTING.md).
Optional Tools for Citus:
* [pgenv](https://github.com/thanodnl/pgenv) (for easier management of PostgreSQL versions)
* [citus_dev](https://github.com/citusdata/tools/tree/develop/citus_dev) (for easier configuration of Citus environment)

# Using SQLancer

The following commands run the Citus implementation of SQLancer using Ternary Logic Query Partitioning (TLP):

```
cd target
java -jar SQLancer-0.0.1-SNAPSHOT.jar --num-threads 4 citus --oracle QUERY_PARTITIONING
```

How to configure the run and how to find the output logs is explained in [SQLancer - Using SQLancer](https://github.com/sqlancer/sqlancer#using-sqlancer).

The `--repartition` flag is a boolean optional argument specific to the Citus implementation (and therefore should be used after `citus` on the command line) that enables [repartition joins](https://docs.citusdata.com/en/v9.3/develop/api_guc.html?highlight=repartition%20join#citus-enable-repartitioned-insert-select-boolean). It is set to `true` by default.

## Interpreting output logs

### Current logs

If the `--log-each-select` option is enabled, each database being tested has a corresponding `-cur.log` file that is populated with all SQL statements sent to the database.

### Error logs

When a bug is found in a database being tested, a corresponding `.log` file is created and is populated with all SQL statements necessary to reproduce the bug. 

1. At the top of the file is the (commented-out) error message, which provides information about the panic error/logic bug detected.
2. Below that are (commented-out) lines that give more information about the specific thread being run, including the seed value (which can be passed in as a command line flag in a later run to reproduce the same thread run).
3. Then, the steps to create the Citus database cluster are provided as commented-out lines. (Following these steps are equivalent to running `citus_dev make XXX` or following the [Citus Docs instructions](https://docs.citusdata.com/en/v9.3/installation/single_machine_debian.html) for setting up a single-machine cluster.)
4. The rest of the file (not commented-out) contains the SQL statements that prepare the testing database. 
5. If the bug detected is a logic bug (the error was raised by the TLP Oracle), then the pair of buggy SELECT statements whose result sets mismatch are also appended to the end of the file as commented-out lines. 

It is important to note that these `.log` files are valid sources of SQL commands that can be passed in with the `-f` flag to the `psql` command. As long as the empty database that the file is being passed into is created with Citus support and the proper worker nodes as described in step 3, this will reproduce the state that the testing database was in when the error was detected. Then, the SQL statement(s) that caused the error can be executed to reproduce the error itself.

Once a bug is identified, it is also possible to check whether the bug is particular to Citus or was inherited from PostgreSQL, since Citus is a PostgreSQL extension. For this, a copy of the `.log` file can be made where all Citus-specific statements (distributing a table, creating a reference table etc.) are removed. Executing this file on an empty database would produce the “vanilla” state that the database would be in without any Citus functionalities. Then, the SQL statement(s) that caused the error can be executed here to check whether the error is reproduced in “vanilla” PostgreSQL as well.

# Maintaining & Contributing

The instructions for setting up a development environment for contributing to SQLancer are explained in [SQLancer - Development](https://github.com/sqlancer/sqlancer/blob/master/CONTRIBUTING.md).

## Updating expected/ignored Citus errors

The `CitusBugs.java` file in the `src/sqlancer/citus/` directory and the `CitusCommon.java` file in the `src/sqlancer/citus/gen/` directory should be continuously updated to reflect the currently unsupported functionalities and active bugs. 

Not all SQL commands generated by SQLancer are supported by the DBMS - they might raise `SQLException`s. For instance, a command that involves an invalid casting may raise a `cannnot cast type` error. These errors do not indicate any bugs in the DBMS, which is why it is desirable to quietly ignore them if raised. The `PostgresCommon` and `CitusCommon` classes in SQLancer collect these expected errors and ensure that SQLancer does not explicitly raise an error if an expected error is thrown.

The `addCitusErrors()` method in `CitusCommon.java` adds Citus-specific errors to the pool of expected errors. It is important to note that it is enough for a string to be a substring of the error message for an error to be ignored. This method is populated with errors that are expected in Citus behavior either because the SQL command generated by SQLancer is currently not supported by Citus, or because a bug that has already been identified has not been fixed yet and is redundantly re-appearing. Both of these, especially the latter group, are dynamic and require updating. 

The `CitusBugs` class in `CitusBugs.java` is an interface between [issues](https://github.com/citusdata/citus/issues?q=is%3Aissue+label%3Asqlancer) opened in the Citus GitHub repository and the bugs listed in the `addCitusErrors()` method in `CitusCommon.java`. Each bug is assigned a corresponding boolean variable, which can be switched to `false` (uninitialized) when the error is fixed on the Citus master branch. 

### What to do: new bug found

If the bug found is a panic error, i.e. NOT a logic bug (mismatch in result sets identified by the TLP Oracle), this error should be added to the `CitusBugs` class and the `addCitusErrors()` method. 
1. Open an issue for the bug in the [Citus GitHub repository](https://github.com/citusdata/citus/issues?q=is%3Aissue+label%3Asqlancer+), and tag the issue with the `sqlancer` label.
2. Add a boolean variable associated with this issue to the `CitusBugs` class and set it to `true`.
3. Add the error message to the `addCitusErrors()` method wrapped inside an if-statement referring to the boolean created in the `CitusBugs` class.

If the bug found is a logic bug, i.e. a mismatch in result sets identified by the TLP Oracle, perform step 1 only.

### What to do: bug fixed

If the bug fixed was a panic error, i.e. NOT a logic bug (mismatch in result sets identified by the TLP Oracle), the boolean in the `CitusBugs` class corresponding to the issue resolved should be set to `false` (uninitialized) once the fix is merged to the Citus master branch.

If the bug found was a logic bug, i.e. a mismatch in result sets identified by the TLP Oracle, no actions are necessary.

### What to do: change in Citus support for PostgreSQL commands

An error that was previously raised by Citus due to unsupported PostgreSQL functionalities can be removed from the `addCitusErrors()` method if Citus begins supporting this functionality.

## Modifying the database environment setup

The `CitusProvider.java` file in the `src/sqlancer/citus/` directory includes the methods for connecting to an existing database and creating the distributed database environment, as well as for preparing the environment for testing (creation of local, distributed, and reference tables and modification of these tables).

## Modifying JOINs in the SELECT statements generated for testing

The `CitusTLPBase.java` file in the `src/sqlancer/citus/oracle/tlp/` directory includes the methods for generating JOIN clauses, which can be modified to alter the scope of the JOINs.