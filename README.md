# cq-9048-reproducer

Standalone reproducer for [apache/camel-quarkus#9048](https://github.com/apache/camel-quarkus/issues/9048) —
the DB2 part of `camel-quarkus-integration-test-jdbc-grouped` failing in the Quarkus Platform build with:

```
java.lang.IllegalStateException: The image icr.io/db2_community/db2:12.1.5.0 requires you to accept a license agreement.
Please place a file at the root of the classpath named container-license-acceptance.txt, ...
```

## What this module does

It mimics how the Quarkus Platform executes the Camel Quarkus integration tests
(see the generated `quarkus-camel/integration-tests/camel-quarkus-integration-test-jdbc-grouped` platform module):

* the test classes come from the **published test-jar** `camel-quarkus-integration-test-jdbc-grouped-3.39.0-tests.jar`,
  discovered via surefire `dependenciesToScan` — this module has **no test sources and no test resources** of its own;
* the `oracle`, `mysql`, `mariadb` and `postgresql` datasources are disabled through fake JDBC URLs in surefire
  `systemPropertyVariables` (the same trick the platform uses to disable datasources), so only the `db2`, `mssql`
  and `h2` dev services start.

The DB2 license acceptance file `container-license-acceptance.txt` is present **inside the test-jar** (with the correct
image name), but in the platform CI the Testcontainers lookup of that classpath resource fails, killing the DB2 dev
service during augmentation — and with it all 77 tests of the grouped module.

## Prerequisites

* Java 17+, Maven 3.9+
* a Docker-API compatible container runtime (Docker, or a Podman socket at `unix:///var/run/docker.sock`)
* network access to pull `icr.io/db2_community/db2:12.1.5.0` (~2 GB) on the first run

## Plain run

```
mvn test -Dtest=CamelDb2JdbcTest
```

Starts the real DB2 and MS SQL Server dev services and runs the 11 DB2 tests. On a healthy local environment the
license lookup finds the file inside the test-jar and this run **passes** (first run takes a few minutes for the
image pulls and DB2 startup). In the Quarkus Platform CI the same setup fails with the license error above.

## Run A — reproduce the platform failure deterministically

The failure mode is "the license lookup does not find/match the entry for the image". You can simulate it on any
machine by using an image name that is not listed in the license file (tag the same image differently):

```
docker pull icr.io/db2_community/db2:12.1.5.0
docker tag icr.io/db2_community/db2:12.1.5.0 icr.io/db2_community/db2:12.1.5.0-repro

mvn test -Dtest=CamelDb2JdbcTest \
  '-Dquarkus.datasource.db2.devservices.image-name=icr.io/db2_community/db2:12.1.5.0-repro' \
  '-Dquarkus.datasource.mssql.jdbc.url=jdbc:sqlserver://localhost:1433;databaseName=disabled'
```

Result — the exact failure signature seen in the platform CI (`Tests run: 11, Errors: 1, Skipped: 10`):

```
Caused by: java.lang.IllegalStateException: The image icr.io/db2_community/db2:12.1.5.0-repro requires you to accept a license agreement. ...
        at org.testcontainers.utility.LicenseAcceptance.assertLicenseAccepted(LicenseAcceptance.java:30)
        at org.testcontainers.db2.Db2Container.configure(Db2Container.java:64)
```

(The MS SQL Server datasource is disabled via its fake URL here so that only DB2 is exercised.)

## Run B — verify the fix

Accepting the license through the container environment makes `Db2Container.configure()` skip the classpath
lookup entirely (Testcontainers only checks the file when the `LICENSE` env variable is absent):

```
mvn test -Dtest=CamelDb2JdbcTest \
  '-Dquarkus.datasource.db2.devservices.image-name=icr.io/db2_community/db2:12.1.5.0-repro' \
  '-Dquarkus.datasource.db2.devservices.container-env.LICENSE=accept' \
  '-Dquarkus.datasource.mssql.jdbc.url=jdbc:sqlserver://localhost:1433;databaseName=disabled'
```

Result: `Tests run: 11, Failures: 0, Errors: 0` against the real DB2 container — with the same "unlicensed"
image name that failed in run A.

This is exactly what the fix for apache/camel-quarkus#9048 does in
`integration-test-groups/jdbc/db2/src/main/resources/application.properties`:

```
quarkus.datasource.db2.devservices.container-env."LICENSE"=accept
```

(plus `quarkus.datasource.mssql.devservices.container-env."ACCEPT_EULA"=Y` for MS SQL Server, which relies on the
same classpath-file mechanism), removing the dependency on `container-license-acceptance.txt` altogether.
