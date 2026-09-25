# MariaDB `localmysql` Run

Date: 2026-09-14  
Branch: `72-console-log-sql`  
Project: `spring-7-rest-mvc`

## Objective

Run the Spring Boot application with the `localmysql` profile against the locally installed MariaDB server, exercise the REST API with `curl`, capture the database-related issues, apply temporary runtime/database workarounds, and stop both Spring Boot and MariaDB afterward.

## Branch Configuration

The active profile is defined in `src/main/resources/application-localmysql.properties`:

```properties
spring.datasource.username=restadmin
spring.datasource.password=password
spring.datasource.url=jdbc:mysql://127.0.0.1:3306/restdb?useUnicode=true&characterEncoding=UTF-8&serverTimezone=UTC
spring.jpa.database=mysql
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.MySQLDialect
spring.jpa.hibernate.ddl-auto=update
spring.jpa.properties.hibernate.show_sql=true
spring.jpa.properties.hibernate.format_sql=true
```

The branch also contains `src/scripts/mysql-init.sql`, which creates `restdb` and the `restadmin` user.

## Environment

```text
mariadbd: /data/data/com.termux/files/usr/bin/mariadbd
mariadb:  /data/data/com.termux/files/usr/bin/mariadb
Maven:    Apache Maven 3.9.16
Java:     OpenJDK 25.0.4
MariaDB:  12.1.2-MariaDB
Spring:   Spring Boot 4.0.6 / Spring Framework 7.0.7
```

MariaDB was initially stopped. Its existing data directory was:

```text
/data/data/com.termux/files/usr/var/lib/mysql
```

## Issues and Workarounds

### 1. MariaDB service wrapper could not start

The packaged service wrapper was attempted:

```text
/data/data/com.termux/files/usr/etc/init.d/mysql start
```

It failed because the installation has no operating-system `mysql` user:

```text
mariadbd-safe-helper: Can't change to run as user 'mysql'. Please check that the user exists!
ERROR!
```

Workaround: start `mariadbd` directly as the current user, using the existing data directory and packaged socket:

```bash
mariadbd \
  --user=root \
  --datadir=/data/data/com.termux/files/usr/var/lib/mysql \
  --socket=/data/data/com.termux/files/usr/var/run/mysqld.sock \
  --port=3306 \
  --bind-address=127.0.0.1 \
  --pid-file=/tmp/spring-rest-mariadb.pid \
  --log-error=/data/data/com.termux/files/usr/var/lib/mysql/localhost.err
```

Verification:

```text
$ mariadb-admin --protocol=tcp --host=127.0.0.1 --port=3306 --user=root ping
mysqld is alive
```

### 2. MySQL user-creation syntax was rejected by MariaDB

The repository script contains:

```sql
CREATE USER IF NOT EXISTS `restadmin`@`%`
IDENTIFIED WITH mysql_native_password BY 'password';
```

MariaDB returned:

```text
ERROR 1064 (42000): You have an error in your SQL syntax
```

Workaround: create the user using MariaDB-compatible syntax and grant the required privileges:

```sql
DROP USER IF EXISTS 'restadmin'@'%';
CREATE USER 'restadmin'@'%' IDENTIFIED BY 'password';
GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, DROP, REFERENCES, INDEX,
ALTER, EXECUTE, CREATE VIEW, SHOW VIEW, CREATE ROUTINE, ALTER ROUTINE,
EVENT, TRIGGER ON restdb.* TO 'restadmin'@'%';
FLUSH PRIVILEGES;
```

Spring connected as `restadmin@localhost`, so an explicit localhost account was also required:

```sql
DROP USER IF EXISTS 'restadmin'@'localhost';
CREATE USER 'restadmin'@'localhost' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON restdb.* TO 'restadmin'@'localhost';
FLUSH PRIVILEGES;
```

Without this account Spring failed with:

```text
Access denied for user 'restadmin'@'localhost' (using password: YES)
```

### 3. The configured `MySQLDialect` generated incompatible MariaDB DDL

The first profile run connected successfully but generated columns such as:

```sql
create table beer (
    id varchar not null,
    ...
)
```

MariaDB rejected the DDL because `varchar` had no length. The application then failed because `restdb.beer` did not exist.

Workaround: override the dialect for this run without changing repository files:

```bash
mvn spring-boot:run \
  -Dspring-boot.run.profiles=localmysql \
  -Dspring-boot.run.jvmArguments='-Dspring.jpa.properties.hibernate.dialect=org.hibernate.dialect.MariaDBDialect'
```

The branch still declares `MySQLDialect`; the override is a runtime compatibility workaround for this MariaDB version.

### 4. UUID columns required binary storage

After switching to `MariaDBDialect`, Hibernate connected and generated valid DDL, but the existing manually created `varchar(36)` UUID columns rejected Hibernate's binary UUID values:

```text
Data truncation: Incorrect string value ... for column `restdb`.`beer`.`id`
```

Workaround: change the temporary schema columns to `BINARY(16)`:

```sql
ALTER TABLE beer MODIFY id BINARY(16) NOT NULL;
ALTER TABLE customer MODIFY id BINARY(16) NOT NULL;
```

The tables used for the successful run were created as follows:

```sql
CREATE TABLE beer (
  id BINARY(16) NOT NULL,
  beer_name VARCHAR(50) NOT NULL,
  beer_style TINYINT NOT NULL,
  created_date DATETIME(6),
  price DECIMAL(38,2) NOT NULL,
  quantity_on_hand INTEGER,
  upc VARCHAR(255) NOT NULL,
  update_date DATETIME(6),
  version INTEGER,
  PRIMARY KEY (id)
) ENGINE=InnoDB;

CREATE TABLE customer (
  id BINARY(16) NOT NULL,
  created_date DATETIME(6),
  name VARCHAR(255),
  update_date DATETIME(6),
  version INTEGER,
  PRIMARY KEY (id)
) ENGINE=InnoDB;
```

## Successful Application Run

The final successful command was:

```text
$ mvn spring-boot:run -Dspring-boot.run.profiles=localmysql -Dspring-boot.run.jvmArguments='-Dspring.jpa.properties.hibernate.dialect=org.hibernate.dialect.MariaDBDialect'
...
The following 1 profile is active: "localmysql"
HikariPool-1 - Added connection com.mysql.cj.jdbc.ConnectionImpl
Tomcat started on port 8080
Started Spring7RestMvcApplication
```

Hibernate SQL logging was active. Representative output included:

```sql
select
    count(*)
from
    beer b1_0

insert
into
    beer
    (beer_name, beer_style, created_date, price, quantity_on_hand, upc, update_date, version, id)
values
    (?, ?, ?, ?, ?, ?, ?, ?, ?)
```

## Complete API Requests and Responses

### List beers

```text
$ curl -i http://127.0.0.1:8080/api/v1/beer
HTTP/1.1 200
Content-Type: application/json

[{"beerName":"Sunshine City","beerStyle":"IPA","createdDate":"2026-09-14T14:28:05.91117","id":"16bac1ed-22ad-42e0-94a9-7ea1ea3b61f5","price":13.99,"quantityOnHand":144,"upc":"12356","updateDate":"2026-09-14T14:28:05.911173","version":0},{"beerName":"Crank","beerStyle":"PALE_ALE","createdDate":"2026-09-14T14:28:05.911154","id":"6a2faa64-dfb9-46f1-bfd7-26dc9b393800","price":11.99,"quantityOnHand":392,"upc":"12356222","updateDate":"2026-09-14T14:28:05.911159","version":0},{"beerName":"Galaxy Cat","beerStyle":"PALE_ALE","createdDate":"2026-09-14T14:28:05.911076","id":"c8b6c7b8-4855-4a99-8336-bc43b947058c","price":12.99,"quantityOnHand":122,"upc":"12356","updateDate":"2026-09-14T14:28:05.911114","version":0}]
```

### Create a beer

```text
$ curl -i -X POST http://127.0.0.1:8080/api/v1/beer -H "Content-Type: application/json" --data '{"beerName":"MariaDB IPA","beerStyle":"IPA","upc":"987654321","price":9.99,"quantityOnHand":24}'
HTTP/1.1 201
Location: /api/v1/beer/a6cd55a1-ce22-4d85-b310-b293fde4d3e3
Content-Length: 0
```

### Patch the persisted beer

```text
$ curl -i -X PATCH http://127.0.0.1:8080/api/v1/beer/a6cd55a1-ce22-4d85-b310-b293fde4d3e3 -H "Content-Type: application/json" --data '{"price":11.49,"quantityOnHand":19}'
HTTP/1.1 204
```

### Read the patched beer

```text
$ curl -i http://127.0.0.1:8080/api/v1/beer/a6cd55a1-ce22-4d85-b310-b293fde4d3e3
HTTP/1.1 200
Content-Type: application/json

{"beerName":"MariaDB IPA","beerStyle":"IPA","createdDate":null,"id":"a6cd55a1-ce22-4d85-b310-b293fde4d3e3","price":11.49,"quantityOnHand":19,"upc":"987654321","updateDate":null,"version":2}
```

### Invalid request

```text
$ curl -i -X POST http://127.0.0.1:8080/api/v1/beer -H "Content-Type: application/json" --data '{"beerName":"","beerStyle":null,"upc":"","price":null}'
HTTP/1.1 400
Content-Type: application/json

[{"beerName":"must not be blank"},{"beerStyle":"must not be null"},{"price":"must not be null"},{"upc":"must not be blank"}]
```

## Database Verification

After the API calls:

```text
beer_count
4

customer_count
3
```

The four beers were the three bootstrap records plus `MariaDB IPA`. The three customers were inserted by `BootstrapData`.

## Shutdown

Spring Boot was stopped first. MariaDB was then shut down cleanly using its socket:

```bash
mariadb-admin \
  --socket=/data/data/com.termux/files/usr/var/run/mysqld.sock \
  --user=root shutdown
```

Final verification showed no `Spring7RestMvcApplication`, Maven `spring-boot:run`, or `mariadbd` processes, and ports `8080` and `3306` were closed.

## Repository Changes

No application source files were modified during this run. The only new file is this report. The MariaDB schema, user accounts, and runtime dialect override were operational workarounds performed outside the repository.

## Why UUID Works on Branch 73

The previous branch did not explicitly define how Java `UUID` values should be bound to JDBC. Hibernate 7 therefore treated UUID values as binary data by default:

```text
Java UUID
    -> default Hibernate UUID mapping
    -> BINARY(16)
```

The entity column, however, was declared as text-oriented:

```java
@Column(length = 36, columnDefinition = "varchar", updatable = false, nullable = false)
private UUID id;
```

When the previous run used a `VARCHAR(36)` column, MariaDB received binary UUID bytes and rejected the insert:

```text
Incorrect string value ... for column `restdb`.`beer`.`id`
```

The earlier workaround was to change the MariaDB IDs to `BINARY(16)` and use `MariaDBDialect`. That allowed the binary mapping to work, but it did not match the entity's intended textual UUID representation.

Branch 73 adds an explicit Hibernate JDBC mapping to both `Beer.id` and `Customer.id`:

```java
import org.hibernate.annotations.JdbcTypeCode;
import org.hibernate.type.SqlTypes;

@JdbcTypeCode(SqlTypes.CHAR)
private UUID id;
```

The resulting mapping is now consistent:

```text
Java UUID
    -> Hibernate CHAR mapping
    -> 36-character UUID string
    -> VARCHAR(36)
```

The branch 73 bind-value log confirms the fix:

```text
binding parameter (9:CHAR) <- [281ef7d5-40a8-49aa-b65e-f74eda75df3d]
```

The same mapping is required for customer UUIDs so both entity IDs use the same text representation. The `@JdbcTypeCode(SqlTypes.CHAR)` change is the specific UUID fix; the separate MariaDB dialect and schema adjustments addressed database-specific DDL compatibility.
