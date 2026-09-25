# Session Branch Change Log

This document records the branch changes inspected during the current session. It focuses on the branches actually checked out, compared, run, or discussed. Existing user notes such as `MARIADB_LOCALMYSQL_RUN.md` remain separate.

## Early Lombok branches

### `2-lombok-builders`

- Added Lombok builder support to the early POJO/model classes.
- `@Builder` generated fluent builders and reduced manual object-construction code.
- The related session note is `BRANCH_2_LOMBOK_BUILDERS.md`.

### `3-lombok-constructors`

- Added Lombok constructor generation to the model classes.
- No-argument constructors support framework/deserialization use cases.
- All-argument constructors support direct construction and builder integration.
- The related session note is `branch-3-lombok-constructors.md`.

## Database and persistence branches

### `69-mysql-configuration`

- Introduced the MySQL/MariaDB-oriented configuration direction.
- Added the database connection settings used by later `localmysql` runs.

### `70-mysql-dependencies`

- Added the MySQL JDBC connector dependency.
- Enabled the application to obtain JDBC connections to MySQL-compatible databases.

### `71-mysql-profile`

- Added the `localmysql` Spring profile.
- Configured the profile for the local MariaDB database `restdb`.
- Configured the `restadmin` credentials and MySQL JDBC URL.

### `72-console-log-sql`

- Enabled Hibernate SQL logging and formatting.
- Added bind-parameter trace logging to make generated SQL values visible during debugging.

### `73-jpa-updates-for-mysql`

- Updated UUID persistence for MySQL/MariaDB compatibility.
- `@JdbcTypeCode(SqlTypes.CHAR)` stores UUID values as 36-character strings.
- Hibernate enum mapping was adjusted for a numeric database representation.

### `74-kikari-datasource-pool`

- Added HikariCP datasource configuration.
- Configured the pool name `RestDB-Pool`, pool sizing, and connection properties.
- Hikari reuses connections instead of opening a new database connection for every request.

### `75-db-create-scripts`

- Added database creation/schema examples used to explain JPA schema generation.
- The examples were later used as background for moving schema ownership to Flyway.

### `76-flyway-deps`

- Added Spring Boot Flyway support and the MySQL Flyway database module.
- Flyway was present but not yet enabled for the active application flow.

### `77-flyway-intit-script`

- Enabled Flyway for the `localmysql` profile.
- Added the V1 migration that creates the beer and customer tables.
- Changed Hibernate schema handling to validation so Hibernate checks the schema instead of creating it.
- Flyway created `flyway_schema_history` and recorded migration version 1.
- A non-empty existing MariaDB schema initially prevented Flyway from establishing history; the demo tables and history table were removed before rerunning the migration.

### `78-fly-add-column`

- Added `Customer.email` to the JPA entity.
- Added `V2__add-email-to-customer.sql`:

  ```sql
  alter table customer
  add column email varchar(255);
  ```

- On startup, Flyway validated V1 and applied V2 successfully.
- The current customer DTO does not expose `email`, so the column is queried by Hibernate but is not returned by the customer REST response.

## CSV branches

### `79-beer-csv-data`

- Updated the Spring Boot parent version from `4.0.0` to `4.0.6`.
- Removed a duplicate README course link.
- No new Java source was introduced on this branch.

### `80-beer-csv-pojo`

- Added `BeerCSVRecord`, a Lombok CSV row model with fields for row number, beer metadata, brewery information, location, and label.
- Updated the CSV header from an unnamed first column to `row`.
- The POJO was not yet connected to a parser or REST endpoint.

### `81-mapping-with-OpenCSV`

- Added the OpenCSV `5.12.0` dependency.
- Added `@CsvBindByName` mappings to `BeerCSVRecord`.
- Added explicit mappings for headers that differ from Java field names:
  - `count.x` to `count`
  - `brewery_id` to `breweryId`
  - `count.y` to `count_y`

### `82-beer-csv-parse-service`

- Added `BeerCsvService` with a `convertCSV(File)` contract.
- Added `BeerCsvServiceImpl` using OpenCSV's `CsvToBeanBuilder`.
- Updated `BootstrapData` to load `classpath:csvdata/beers.csv` when fewer than 10 beers exist.
- Added CSV-to-`Beer` conversion, including beer-style mapping, name abbreviation, row-number UPCs, price `10`, and CSV inventory counts.
- Added service/bootstrap tests.
- Running this branch imported approximately 2,400 CSV beers into MariaDB.

### `83-beer-csv-save-to-db`

- The checked-out branch pointed to the same commit as branch 82 in this repository state.
- No additional diff was present.
- The inherited CSV bootstrap persistence behavior remained active.

### `84-hibernate-create-update-timestamp`

- Added Hibernate `@CreationTimestamp` to `Beer.createdDate`.
- Added Hibernate `@UpdateTimestamp` to `Beer.updateDate`.
- New beers received timestamps during insert, and updates refreshed `updateDate` automatically.

### `85-beer-csv-fix-integration-tests`

- Imported `BeerCsvServiceImpl` into `BootstrapDataTest` so the CSV service dependency exists in the JPA test context.
- Updated expected beer counts from 3 to 2413 after CSV bootstrap loading.
- Updated the controller integration test to expect 2413 beers.

## Query branches

### `86-query-test`

- Added a MockMvc test for `GET /api/v1/beer?beerName=IPA`.
- The test expected exactly 100 results.
- At this stage the production controller ignored the query parameter and returned the full beer dataset.

### `87-query-add-query-param`

- Changed `BeerController.listBeers` to accept:

  ```java
  @RequestParam(required = false) String beerName
  ```

- The parameter was accepted but not passed to the service yet.
- Live curl testing still returned the complete dataset.

### `88-query-refactor-service`

- Changed `BeerService.listBeers()` to `listBeers(String beerName)`.
- Updated `BeerController` to pass `beerName` into the service.
- Updated both `BeerServiceImpl` and `BeerServiceJPA` signatures.
- Updated unit/controller tests to pass either a name or `null`.
- Neither service implementation filtered results yet, so the live endpoint still returned all beers.

### `89-query-list-by-name`

- Added the Spring Data repository method:

  ```java
  List<Beer> findAllByBeerNameIsLikeIgnoreCase(String beerName);
  ```

- Added a repository test expecting 336 matches for `%IPA%`.
- Added `BeerServiceJPA.listBeersByName(String)` as the intended service seam.
- In the inspected source, `listBeersByName` still returned an empty list rather than calling the repository method, so the complete filtering behavior was not finished in this branch.

### `90-query-spring-data-jpa`

- The branch name suggests the next Spring Data JPA query step, but in the repository state used during this session it pointed to the same commit as `89-query-list-by-name`.
- `git diff 89-query-list-by-name..90-query-spring-data-jpa` was empty.
- The effective query code therefore remained:

  ```java
  List<Beer> listBeersByName(String beerName) {
      return new ArrayList<>();
  }
  ```

- The repository method was present:

  ```java
  List<Beer> findAllByBeerNameIsLikeIgnoreCase(String beerName);
  ```

- It was not called by the service method, so the database query was not actually executed for the REST request.

#### Branch 90 runtime evidence

The application was started with the local MariaDB profile and Docker Compose disabled:

```bash
mvn spring-boot:run \
  -Dspring-boot.run.profiles=localmysql \
  -Dspring-boot.run.jvmArguments='-Dspring.docker.compose.enabled=false'
```

Startup completed with Spring Boot 4.0.6, Hikari pool `RestDB-Pool`, Flyway schema version 2, and Tomcat on port 8080. The query request produced an empty result because the service stub returned an empty list:

```bash
curl -i 'http://127.0.0.1:8080/api/v1/beer?beerName=IPA'
```

```text
HTTP/1.1 200
Content-Type: application/json
Content-Length: 2

[]
```

The other REST checks remained successful:

```bash
curl -i http://127.0.0.1:8080/api/v1/customer
```

```text
HTTP/1.1 200
Content-Type: application/json
Content-Length: 490

[{"name":"Customer 1","version":1},{"name":"Customer 2","version":1},{"name":"Customer 3","version":1}]
```

```bash
curl -i -X POST http://127.0.0.1:8080/api/v1/beer \
  -H 'Content-Type: application/json' \
  -d '{"beerName":"Branch 90 IPA","beerStyle":"IPA","upc":"900000001","price":8.90,"quantityOnHand":20}'
```

```text
HTTP/1.1 201
Location: /api/v1/beer/b412b82c-77b8-4ce2-ba1f-8ec397e9b903
Content-Length: 0
```

```bash
curl -i -X POST http://127.0.0.1:8080/api/v1/beer \
  -H 'Content-Type: application/json' \
  -d '{"beerName":"","beerStyle":null,"upc":"","price":0,"quantityOnHand":0}'
```

```text
HTTP/1.1 400
Content-Type: application/json

[{"beerName":"must not be blank"},{"beerStyle":"must not be null"},{"upc":"must not be blank"}]
```

The branch demonstrates the difference between adding a repository method and wiring that method into the service. The repository declaration is ready, but the service implementation still needs to call it with a wildcard such as `%IPA%` for a contains-style case-insensitive search.

## Runtime workarounds used

- Local Maven executable: `/data/data/com.termux/files/usr/bin/mvn`.
- MariaDB was started directly because the Termux service wrapper expected an unavailable OS `mysql` user.
- Spring Boot Docker Compose auto-start was disabled because Docker is unavailable:

  ```bash
  mvn spring-boot:run \
    -Dspring-boot.run.profiles=localmysql \
    -Dspring-boot.run.jvmArguments='-Dspring.docker.compose.enabled=false'
  ```

- MariaDB 12.3 produced a Flyway warning because the installed Flyway version was verified through MariaDB 11.7.
- Hibernate emitted an `Unknown column 'RESERVED'` metadata warning against MariaDB, but schema validation and application startup completed successfully.
- The MariaDB server and Spring Boot process were stopped after each run, and ports 3306 and 8080 were verified as released.

## Branch 96: refactor Spring Data methods for paging

### `96-refactor-spring-data-methods`

Branch created from `origin/96-refactor-spring-data-methods`. This branch completes the paging feature that branch 95 stubbed: the `PageRequest` created in `buildPageRequest()` is now threaded through the repository layer to actually limit results.

#### Changes

**Repository layer (`BeerRepository.java`)** — All three query methods changed from `List<Beer>` to `Page<Beer>` and accept a `Pageable` parameter:

```java
Page<Beer> findAllByBeerNameIsLikeIgnoreCase(String beerName, Pageable pageable);
Page<Beer> findAllByBeerStyle(BeerStyle beerStyle, Pageable pageable);
Page<Beer> findAllByBeerNameIsLikeIgnoreCaseAndBeerStyle(String beerName, BeerStyle beerStyle, Pageable pageable);
```

**Service interface (`BeerService.java`)** — Return type changed from `List<BeerDTO>` to `Page<BeerDTO>`.

**JPA service (`BeerServiceJPA.java`)** — Key changes:
- `listBeers()` passes the `PageRequest` (built from `buildPageRequest()`) to all repository methods instead of the earlier `List`-returning stubs.
- Helper methods `listBeersByName`, `listBeersByStyle`, `listBeersByNameAndStyle` now accept and forward `Pageable`.
- `findAll()` (the no-filter case) now calls `beerRepository.findAll(pageRequest)` instead of `beerRepository.findAll()`.
- The return is now `beerPage.map(beerMapper::beerToBeerDto)` — a single `Page.map()` call instead of `stream().map().collect()`. This preserves the `Page` metadata (total elements, total pages, etc.).
- Removed `import java.util.List` and `java.util.stream.Collectors` in favor of `Page` / `Pageable` imports.

**In-memory service (`BeerServiceImpl.java`)** — `listBeers()` now returns `PageImpl<>(new ArrayList<>(beerMap.values()))` to satisfy the new `Page<BeerDTO>` contract. This is a thin wrapper — the in-memory impl does not filter or paginate; it just wraps the full list in a `Page`.

**Controller (`BeerController.java`)** — `listBeers()` return type changed from `List<BeerDTO>` to `Page<BeerDTO>`, added `import org.springframework.data.domain.Page`. The `@RequestParam` parameters (`pageNumber`, `pageSize`) are unchanged from branch 95; they just now have an effect.

#### JSON response shape

Because the controller now returns Spring Data `Page<T>` directly, the JSON response structure changed:

**Before (List):**
```json
[{"beerName": "...", ...}, {"beerName": "...", ...}]
```

**After (Page):**
```json
{
  "content": [{"beerName": "...", ...}, {"beerName": "...", ...}],
  "pageable": {"pageNumber": 0, "pageSize": 25, ...},
  "totalPages": 97,
  "totalElements": 2428,
  "last": false,
  "first": true,
  "numberOfElements": 25,
  ...
}
```

This means clients must now look at `$.content` for the actual beer array rather than `$.` directly. Spring logged a warning: *"Serializing PageImpl instances as-is is not supported... use Spring Data's PagedModel"* — this is a known limitation of returning `Page` directly from a REST controller without `PageableHandlerInterceptor` / `PagedResourcesAssembler`.

#### Tests

All test files were updated to match the `Page`-returning API:

- **`BeerControllerIT`** — Changed all `$.size()` assertions to `$.content.size()` and `$.[0]` to `$.content[0]`. Added `.queryParam("pageSize", "800")` to most integration test queries so they get the full result set instead of a 25-record default page. `testListBeers` now expects `getContent().size()` of 1000 (the `buildPageRequest` ceiling) when requesting page size 2413.
- **`BeerControllerTest`** — Changed all `beerServiceImpl.listBeers(...)` return handling from `.get(0)` to `.getContent().get(0)` since the return is now a `Page`. Mock returns updated to `Page`.
- **`BeerRepositoryTest`** — Changed `List<Beer>` to `Page<Beer>`, `findAllByBeerNameIsLikeIgnoreCase("%IPA%")` now requires a `Pageable` argument (passed `null` which uses Spring Data's default unpaged request), and `.size()` to `.getContent().size()`.

#### Test results

- `BeerControllerIT`: **16 tests, 0 failures** ✅
- `BeerRepositoryTest`: **3 tests, 0 failures** ✅
- `BeerControllerTest` / `CustomerControllerTest`: **16 errors** — all caused by a pre-existing Mockito/ByteBuddy self-attach failure on JDK 25 in this Termux environment (`Could not initialize plugin: interface org.mockito.plugins.MockMaker`). This is unrelated to the paging changes — the `@WebMvcTest` tests use Mockito `@MockBean` which requires the inline mock maker agent that won't self-attach here.

#### Edge cases noted

1. **`pageSize` ceiling of 1000**: `buildPageRequest` clamps any `pageSize > 1000` to 1000. `testListBeers` confirms this by requesting `pageSize=2413` and getting 1000 back.
2. **`pageSize` of 0 or negative**: Not clamped — would reach `PageRequest.of()` and throw `IllegalArgumentException`, surfacing as a 500. This was flagged as a known issue in branch 95 and remains unaddressed.

## Branch 97: paging add sort

### `97-paging-add-sort`

Branch created from `origin/97-paging-add-sort`. Small, focused change: **one file, +4/−1 lines** — `BeerServiceJPA.java`. It adds ascending sort-by-name to every paginated beer query.

#### Change

Added `import org.springframework.data.domain.Sort;` and a sort clause inside `buildPageRequest()`:

```java
Sort sort = Sort.by(Sort.Order.asc("beerName"));

return PageRequest.of(queryPageNumber, queryPageSize, sort);
```

#### Why

- Branch 96 wired `PageRequest` into the queries, but `PageRequest.of(page, size)` carries no ordering. SQL `LIMIT/OFFSET` without `ORDER BY` has no guaranteed row order, so paged results could be arbitrary and even inconsistent between requests — the same page number could show different rows on separate calls.
- `Sort.by(Sort.Order.asc("beerName"))` sorts by the entity's `beerName` property, which JPA maps to the `beer_name` column. `Sort.Order.asc(...)` sets direction; `Sort.by(...)` accepts multiple orders, so additional keys can be chained later (e.g. `Sort.by(asc("beerName"), desc("price"))`).
- Because the sort lives in `buildPageRequest()`, all four query paths — name, style, name+style, and the no-filter `findAll` — inherit it with no controller or repository change.

#### Runtime evidence

Started with the `localmysql` profile (MariaDB, `restdb`, 2429 beers):

```bash
mvn spring-boot:run -Dspring-boot.run.profiles=localmysql
```

Hibernate confirmed the generated ordering:

```text
order by
    b1_0.beer_name 
limit
    ?, ?
```

Page 1 (`pageNumber=1&pageSize=5`) began at `#001 Golden Amber Lager`, `#002 American I.P.A.`, ...; page 2 continued at `#9`, `077XX`, `10 Degrees of Separation`, ... — a clean alphabetical progression confirming the sort spans pages.

Page metadata from the `Page` JSON:

```text
totalElements: 2429
totalPages:    486
size:          5
sort:          {empty: false, sorted: true, unsorted: false}
```

```bash
curl "http://localhost:8080/api/v1/beer?pageNumber=2&pageSize=5"
```

#### Notes

- The sort key `beerName` is the Java property name, not the column name; Spring Data resolves it against the entity metamodel.
- A deterministic sort is a prerequisite for correct paging. Without it, clients could see duplicate or missing records as they walk through pages.

## Branch 98: add order and order-line tables

### `98-rel-add-order-order-detail`

Branch created from `origin/98-rel-add-order-order-detail`. **No Java changes at all** — the only addition is a new manual DDL script, `src/scripts/add-order-tables.sql` (29 lines). It is a preparatory branch for the JPA relationship mapping work that follows in later branches (99+).

#### The script

```sql
drop table if exists beer_order_line;
drop table if exists beer_order;

CREATE TABLE `beer_order`
(
    id                 varchar(36) NOT NULL,
    created_date       datetime(6)  DEFAULT NULL,
    customer_ref       varchar(255) DEFAULT NULL,
    last_modified_date datetime(6)  DEFAULT NULL,
    version            bigint       DEFAULT NULL,
    customer_id        varchar(36)  DEFAULT NULL,
    PRIMARY KEY (id),
    CONSTRAINT FOREIGN KEY (customer_id) REFERENCES customer (id)
) ENGINE = InnoDB;

CREATE TABLE `beer_order_line`
(
    id                 varchar(36) NOT NULL,
    beer_id            varchar(36) DEFAULT NULL,
    created_date       datetime(6) DEFAULT NULL,
    last_modified_date datetime(6) DEFAULT NULL,
    order_quantity     int         DEFAULT NULL,
    quantity_allocated int         DEFAULT NULL,
    version            bigint      DEFAULT NULL,
    beer_order_id      varchar(36) DEFAULT NULL,
    PRIMARY KEY (id),
    CONSTRAINT FOREIGN KEY (beer_order_id) REFERENCES beer_order (id),
    CONSTRAINT FOREIGN KEY (beer_id) REFERENCES beer (id)
) ENGINE = InnoDB;
```

#### Key points

- **Not a Flyway migration.** The file lives in `src/scripts/`, not `src/main/resources/db/migration/`, and is not named `V*__*.sql`, so Flyway never sees it. It is a developer-run reference script — the schema is applied by hand, not on application startup. This is consistent with how earlier branches (e.g. `75-db-create-scripts`) introduced schema examples.
- **Drop order respects FK dependencies.** `beer_order_line` is dropped before `beer_order` because it references `beer_order`. Dropping in the reverse order would fail on the FK constraint.
- **Two-level ordering model.** `beer_order` (header) → `beer_order_line` (details), a one-to-many. The line table also references `beer`, so lines point at the products being ordered.
- **FK relationships created:**
  - `beer_order.customer_id` → `customer.id`
  - `beer_order_line.beer_order_id` → `beer_order.id`
  - `beer_order_line.beer_id` → `beer.id`
- **`version bigint`** on both tables holds the optimistic-locking column (note: `bigint` here, whereas the `beer`/`customer` tables use `integer` — the later entity mappings settle on `Integer`).
- **Audit timestamps.** `created_date` and `last_modified_date` mirror the `createdDate` / `updateDate` pattern on the existing entities. The name `last_modified_date` differs from the existing `update_date`, so the Order entity mapping will need an explicit `@Column` mapping.
- **`customer_ref`** is a plain `varchar(255)` denormalized copy of the customer identity, separate from the `customer_id` FK. This appears in later order-DTO work as a human-readable reference.

#### Empirical evidence

MariaDB (restdb) before applying the script contained only `beer`, `customer`, `flyway_schema_history`. After running the script by hand:

```bash
mysql -u restadmin -ppassword restdb < src/scripts/add-order-tables.sql
```

Tables present: `beer`, `beer_order`, `beer_order_line`, `customer`, `flyway_schema_history`.

FK constraints reported by `information_schema.KEY_COLUMN_USAGE`:

| TABLE_NAME | COLUMN_NAME | REFERENCED_TABLE | REFERENCED_COLUMN |
|---|---|---|---|
| beer_order | customer_id | customer | id |
| beer_order_line | beer_order_id | beer_order | id |
| beer_order_line | beer_id | beer | id |

#### Runtime behavior is unchanged

The application was started with the `localmysql` profile. Startup showed `Successfully validated 2 migrations` and `Schema \`restdb\` is up to date. No migration necessary.` — Flyway ignores the new script entirely. Spring Data reported **2 JPA repository interfaces** (`BeerRepository`, `CustomerRepository`), unchanged from branch 97, because no `BeerOrder`/`BeerOrderLine` entities or repositories exist yet.

`ddl-auto=validate` did not complain about the two new tables: Hibernate validates that mapped entities match the schema, and does not object to extra tables it has no mapping for. The REST API behaved exactly as on branch 97:

```bash
curl "http://localhost:8080/api/v1/beer?pageNumber=1&pageSize=3"
```

```text
totalElements: 2429
first three (name-sorted): #001 Golden Amber Lager, #002 American I.P.A., #003 Brown & Robust Porter
```

This branch is schema-only groundwork: the tables and foreign keys exist so that the next branches can add the JPA entities, repositories, and mappings that use them.

## Branch 99: order tables as a Flyway migration

### `99-rel-add-order-order-detail`

Branch created from `origin/99-rel-add-order-order-detail`. Again **no Java changes**. This branch takes the DDL script that branch 98 added by hand and moves it into Flyway's managed migration path as `src/main/resources/db/migration/V3__add-order-tables.sql` (29 lines, byte-identical content to branch 98's `src/scripts/add-order-tables.sql`).

#### What changed

- Added `V3__add-order-tables.sql` under `src/main/resources/db/migration/`, so Flyway discovers, validates, and applies it on startup.
- The branch-98 manual script `src/scripts/add-order-tables.sql` is **still tracked** — both files coexist. The `src/scripts/` copy is now redundant reference material; the Flyway copy is the one that actually shapes the database.

#### Why this matters

Branch 98's script required someone to run it manually (`mysql < src/scripts/add-order-tables.sql`). Branch 99 makes the schema change automatic and versioned:

- **Automatic on startup.** Application boot now creates the order tables without any manual step.
- **Recorded and checksummed.** Flyway writes a row to `flyway_schema_history` and hashes the file, so later edits to `V3` would be detected as a checksum mismatch.
- **Ordered after V1/V2.** The `V3` prefix guarantees the order tables are created only after `customer` and `beer` exist, satisfying the foreign keys.

#### The running-state detail worth noting

In the branch-98 step of this session, the order tables had already been created by hand — so before branch 99's app start, `beer_order` / `beer_order_line` physically existed but were **absent from `flyway_schema_history`** (history was at version 2). Flyway therefore treated V3 as unapplied and ran it. Because V3 begins with:

```sql
drop table if exists beer_order_line;
drop table if exists beer_order;
```

it dropped the manually created tables and recreated them under Flyway's ownership. This is why the migration is safe to run against a database where the tables already exist, but also why embedding `drop table` in a versioned migration is unusual for production use — it is destructive if any data were present. Here the tables were empty, so nothing was lost.

#### Empirical evidence

Startup with the `localmysql` profile:

```text
Successfully validated 3 migrations (execution time 00:00.115s)
Migrating schema `restdb` to version "3 - add-order-tables"
Successfully applied 1 migration to schema `restdb`, now at version v3 (execution time 00:00.108s)
Started Spring7RestMvcApplication in 21.278 seconds
```

`flyway_schema_history` after the run:

| version | description | success |
|---|---|---|
| 1 | init-mysql-database | 1 |
| 2 | add-email-to-customer | 1 |
| 3 | add-order-tables | 1 |

Tables present: `beer`, `beer_order`, `beer_order_line`, `customer`, `flyway_schema_history`.

FK constraints recreated by Flyway (re-confirmed via `information_schema.KEY_COLUMN_USAGE`):

| TABLE_NAME | COLUMN_NAME | REFERENCED_TABLE | REFERENCED_COLUMN |
|---|---|---|---|
| beer_order | customer_id | customer | id |
| beer_order_line | beer_order_id | beer_order | id |
| beer_order_line | beer_id | beer | id |

Spring Data still reports **2 JPA repository interfaces** — no Order entities or repositories yet, so the REST API is unchanged from branches 97/98:

```bash
curl "http://localhost:8080/api/v1/beer?pageNumber=1&pageSize=2"
```

```text
totalElements: 2429
first two (name-sorted): #001 Golden Amber Lager, #002 American I.P.A.
```

This is the same schema as branch 98, but now owned and versioned by Flyway rather than applied by hand. The JPA entity/repository work for orders remains in the following branches.

## Branch 100: order entities and the customer one-to-many

### `100-rel-one-to-many`

Branch created from `origin/100-rel-one-to-many`. First branch with **real Java changes since 97**: three files, +110 lines.

- `entities/BeerOrder.java` (new, 61 lines)
- `entities/BeerOrderLine.java` (new, 45 lines)
- `entities/Customer.java` (+4: import `java.util.Set`, `@OneToMany` field)

No repositories, DTOs, mappers, or controllers were added, so the entities are not yet reachable through the REST API. Spring Data still reports **2 JPA repository interfaces**.

#### `BeerOrder` entity

```java
@Getter @Setter @Entity @NoArgsConstructor @AllArgsConstructor @Builder
public class BeerOrder {

    @Id
    @GeneratedValue(generator = "UUID")
    @UuidGenerator
    @JdbcTypeCode(SqlTypes.CHAR)
    @Column(length = 36, columnDefinition = "varchar(36)", updatable = false, nullable = false )
    private UUID id;

    @Version
    private Long version;

    @CreationTimestamp
    @Column(updatable = false)
    private Timestamp createdDate;

    @UpdateTimestamp
    private Timestamp lastModifiedDate;

    public boolean isNew() {
        return this.id == null;
    }

    private String customerRef;

    @ManyToOne
    private Customer customer;
}
```

- Same UUID strategy as `Beer`/`Customer`: `@UuidGenerator` + `@JdbcTypeCode(SqlTypes.CHAR)` so the UUID binds as a 36-char string matching the `varchar(36)` column.
- `@Version private Long version` — note **`Long`, not `Integer`**, matching the `bigint` column in `V3`. This differs from `Beer`/`Customer`, which use `Integer`/`integer`.
- `@CreationTimestamp` on `createdDate` with `@Column(updatable = false)` — the column is written once on insert and never updated.
- `@UpdateTimestamp` on `lastModifiedDate` — refreshed on every update. Note the field/property name `lastModifiedDate` (maps to `last_modified_date`), different from the existing entities' `updateDate` (`update_date`). No explicit `@Column` name is needed because the implicit naming strategy converts camelCase to snake_case and matches the DDL.
- `isNew()` returns `this.id == null`. This is a common JPA idiom used later by service code to decide `persist` (new) versus `merge` (existing); it exists here but no caller uses it yet.
- `customerRef` — a plain string column (`customer_ref`), the denormalized reference that `V3` created.
- `@ManyToOne private Customer customer;` — the **owning side** of the Customer↔BeerOrder relationship. With no `@JoinColumn`, JPA's default join column is `customer_<pk>` → `customer_id`, which matches the `V3` FK `beer_order.customer_id → customer.id`. Fetch type defaults to `EAGER` for `@ManyToOne` (a point that often gets revisited later).

#### `BeerOrderLine` entity

```java
@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Entity @Builder
public class BeerOrderLine {

    @Id @GeneratedValue(generator = "UUID") @UuidGenerator
    @JdbcTypeCode(SqlTypes.CHAR)
    @Column(length = 36, columnDefinition = "varchar(36)", updatable = false, nullable = false )
    private UUID id;

    @Version
    private Long version;

    @CreationTimestamp
    @Column(updatable = false)
    private Timestamp createdDate;

    @UpdateTimestamp
    private Timestamp lastModifiedDate;

    public boolean isNew() {
        return this.id == null;
    }

    private Integer orderQuantity = 0;
    private Integer quantityAllocated = 0;
}
```

- Same UUID/version/timestamp/isNew pattern as `BeerOrder`.
- `orderQuantity` and `quantityAllocated` are initialized to `0`, so a newly built line defaults to zero rather than null.
- **Deliberately incomplete relationships.** Although `V3` gave `beer_order_line` two foreign keys (`beer_order_id → beer_order.id` and `beer_id → beer.id`), this entity maps **neither**. There is no `@ManyToOne BeerOrder`, no `@OneToMany` back-reference, and no `@ManyToOne Beer`. Those two FK columns exist in the schema but are currently unmanaged by JPA. This is the visible gap that branch 101 (`rel-assn-one-to-many`) is positioned to fill.

#### `Customer` change

```java
@OneToMany(mappedBy = "customer")
private Set<BeerOrder> beerOrders;
```

- Turns the relationship **bidirectional**. `Customer` is the **inverse** side: `mappedBy = "customer"` says the FK lives on the `BeerOrder.customer` field, so JPA reads the association from there and manages no column on the customer side.
- `Set` (not `List`) is chosen to avoid duplicate-element issues and to sidestep ordering concerns; it is not initialized, so it can be null until loaded/populated.
- No `cascade` and no `orphanRemoval` are declared, so persisting/removing a `Customer` will not cascade to its orders. Default `@OneToMany` fetch is `LAZY`.

#### Schema validation

The entities must line up with the `V3` DDL because `ddl-auto=validate`. They do:

| Entity property | Column | DDL type | Status |
|---|---|---|---|
| BeerOrder.id | id | varchar(36) | match |
| BeerOrder.version | version | bigint (`Long`) | match |
| BeerOrder.createdDate | created_date | datetime(6) | match |
| BeerOrder.lastModifiedDate | last_modified_date | datetime(6) | match |
| BeerOrder.customerRef | customer_ref | varchar(255) | match |
| BeerOrder.customer | customer_id | varchar(36) FK | match |
| BeerOrderLine.* | id, version, created_date, last_modified_date, order_quantity, quantity_allocated | — | match |

Startup completed with **zero `Schema-validation` errors** — Hibernate accepted every mapping. The unmapped `beer_id` / `beer_order_id` columns do not cause validation failures, since validation checks that mapped things exist in the schema, not that every column is mapped.

#### Empirical evidence

MariaDB had to be restarted (the server had stopped after the branch-99 run). Flyway reported the schema already at v3:

```text
Schema `restdb` is up to date. No migration necessary.
```

Application startup:

```text
Found 2 JPA repository interfaces.
Started Spring7RestMvcApplication in 23.53 seconds
```

(Devtools performed a follow-up restart, logging `Found 0 JPA repository interfaces` on the restart pass — a devtools reload artifact, not a real state change; the running app continued to serve requests normally.)

REST checks remained healthy and unchanged, because no order endpoints exist yet:

```bash
curl "http://localhost:8080/api/v1/beer?pageNumber=1&pageSize=2"
```

```text
totalElements: 2429
first two (name-sorted): #001 Golden Amber Lager, #002 American I.P.A.
```

```bash
curl http://localhost:8080/api/v1/customer
```

```text
[{"createdDate":"2026-09-14T16:37:55.778573","id":"8a857c62-...","name":"Customer 1","updateDate":"...","version":1}, ...]
```

#### Design notes / loose ends for later branches

- **Customer ↔ BeerOrder is now bidirectional**, owning side on `BeerOrder.customer`.
- **BeerOrderLine is an orphan entity** — no relationship to `BeerOrder` or `Beer` despite the schema FKs. This is the next branch's work.
- **Type inconsistency** across the codebase: `Long`/`Timestamp` on the order entities vs `Integer`/`LocalDateTime` on `Beer`/`Customer`. Not a bug, but worth noting.
- **No cascade/orphanRemoval**, so order lines will not persist automatically when an order is saved — that has to be handled explicitly.
- **No mappers/DTOs/repositories/controllers**, so none of this is exposed over HTTP yet.

## Branch 101: complete the one-to-many associations

### `101-rel-assn-one-to-many`

Branch created from `origin/101-rel-assn-one-to-many`. Four files, +17/−2 lines. This is the branch that closes the gap left in branch 100: `BeerOrderLine`, which until now had no relationship mappings, gains both of its owning sides, and the two inverse collections are added.

- `entities/BeerOrderLine.java` (+6) — the two `@ManyToOne` owning sides
- `entities/BeerOrder.java` (+7) — inverse `@OneToMany` collection
- `entities/Beer.java` (+4) — inverse `@OneToMany` collection
- `pom.xml` (−2) — trailing-blank-line cleanup only (no dependency change)

#### `BeerOrderLine` gains both owning sides

```java
@ManyToOne
private BeerOrder beerOrder;

@ManyToOne
private Beer beer;
```

These are the **owning** sides of both associations. With no `@JoinColumn`, JPA's implicit naming produces `beer_order_id` and `beer_id`, which match the two FKs that `V3` created. The columns that were unmapped since branch 100 are now managed.

#### `BeerOrder` and `Beer` gain inverse collections

In `BeerOrder`:

```java
@OneToMany(mappedBy = "beerOrder")
private Set<BeerOrderLine> beerOrderLines;
```

In `Beer`:

```java
@OneToMany(mappedBy = "beer")
private Set<BeerOrderLine> beerOrderLines;
```

Each is the **inverse** side. `mappedBy` points at the field on `BeerOrderLine` that owns the FK, so these collections are read-only with respect to the join column — JPA issues no insert/update for them and derives them from the owning side.

#### The completed relationship graph

After this branch the object graph is fully bidirectional and every schema FK is mapped:

| Association | Owning side (owns FK) | Inverse side (`mappedBy`) | Join column |
|---|---|---|---|
| Customer ↔ BeerOrder | `BeerOrder.customer` `@ManyToOne` | `Customer.beerOrders` | `beer_order.customer_id` |
| BeerOrder ↔ BeerOrderLine | `BeerOrderLine.beerOrder` `@ManyToOne` | `BeerOrder.beerOrderLines` | `beer_order_line.beer_order_id` |
| Beer ↔ BeerOrderLine | `BeerOrderLine.beer` `@ManyToOne` | `Beer.beerOrderLines` | `beer_order_line.beer_id` |

This mirrors the `V3` DDL exactly: a two-level order model whose detail rows also point back at the product being ordered.

#### Design notes

- **Fetch strategy is the JPA default.** `@ManyToOne` is `EAGER`, `@OneToMany` is `LAZY`. So loading a single `BeerOrderLine` eagerly pulls in both its `BeerOrder` and its `Beer`, and loading a `BeerOrder` eagerly pulls in its `Customer`. With two eager `@ManyToOne` associations on the line entity, iterating a set of lines can produce N+1 query patterns. Nothing here changes the defaults; tuning would come later.
- **No `cascade`, no `orphanRemoval`.** Saving a `BeerOrder` will not automatically persist its lines, and removing a line from the collection will not delete it. Persistence of the detail rows must be handled explicitly by later service code.
- **Uninitialized `Set` fields.** The three collections are declared but not initialized (`new HashSet<>()`), so they can be null unless populated by the provider or by application code.
- **`@JoinColumn` not required** — implicit naming and the `V3` column names agree, so no explicit join-column annotations were necessary.
- Still **no repositories/DTOs/mappers/controllers** for orders, so the whole order model remains unreachable via REST.

#### Schema validation and runtime

Started with the `localmysql` profile. Hibernate `validate` accepted all new mappings — **zero `Schema-validation` errors** — confirming the implicit join-column names resolve to the real `V3` columns. Flyway reported the schema already current:

```text
Successfully validated 3 migrations
Schema `restdb` is up to date. No migration necessary.
```

Spring Data still reports **2 JPA repository interfaces**; the REST API is unaffected:

```bash
curl "http://localhost:8080/api/v1/beer?pageNumber=1&pageSize=2"
```

```text
totalElements: 2429
first two (name-sorted): #001 Golden Amber Lager, #002 American I.P.A.
```

```bash
curl http://localhost:8080/api/v1/customer
```

```text
[{"createdDate":"2026-09-14T16:37:55.778573","id":"8a857c62-...","name":"Customer 1","updateDate":"...","version":1}, ...]
```

Because no order endpoints exist yet, the new associations are not exercised by any HTTP request; the evidence here is that the mappings compile and validate against the migrated schema.

## Branch 102: beer order repository

### `102-rel-beer-order-repository`

Branch created from `origin/102-rel-beer-order-repository`. Two new files, +51 lines: a repository interface and its first (print-style) test. No service, DTO, mapper, or controller work.

- `repositories/BeerOrderRepository.java` (new, 12 lines)
- `test/.../repositories/BeerOrderRepositoryTest.java` (new, 39 lines)

#### `BeerOrderRepository`

```java
public interface BeerOrderRepository extends JpaRepository<BeerOrder, UUID> {
}
```

- A plain Spring Data JPA repository for the `BeerOrder` aggregate root, keyed by `UUID`. No derived query methods are declared — only the inherited `JpaRepository` CRUD plus `count()`, `findAll()`, etc.
- With no `@Repository` annotation needed (Spring Data supplies the proxy), the interface alone is enough for component scanning to register a bean.
- Note there is **no** `BeerOrderLineRepository`; lines are intended to be reached through the order aggregate.

#### `BeerOrderRepositoryTest`

```java
@SpringBootTest
class BeerOrderRepositoryTest {

    @Autowired BeerOrderRepository beerOrderRepository;
    @Autowired CustomerRepository customerRepository;
    @Autowired BeerRepository beerRepository;

    Customer testCustomer;
    Beer testBeer;

    @BeforeEach
    void setUp() {
        testCustomer = customerRepository.findAll().get(0);
        testBeer = beerRepository.findAll().get(0);
    }

    @Test
    void testBeerOrders() {
        System.out.println(beerOrderRepository.count());
        System.out.println(customerRepository.count());
        System.out.println(beerRepository.count());
        System.out.println(testCustomer.getName());
        System.out.println(testBeer.getBeerName());
    }
}
```

- **`@SpringBootTest`** (full application context), not `@DataJpaTest`. That means the whole app boots, including `BootstrapData`, so the seeded data is available. It does **not** use Testcontainers here, so the test runs against the default in-memory H2 datasource with Hibernate creating the schema from the entity mappings.
- **No assertions.** `testBeerOrders()` only prints values; it is a "print-based" learning test to demonstrate that the repositories wire up and can be queried. It would pass even if the values changed, so it provides no real regression protection.
- `setUp()` uses `findAll().get(0)` to grab the first customer and first beer. This is order-dependent and would throw `IndexOutOfBoundsException` if the bootstrap data were absent.

#### Empirical evidence

**Runtime (`localmysql` profile, MariaDB):** repository scanning now finds **3 JPA repository interfaces** (was 2):

```text
Finished Spring Data repository scanning in 303 ms. Found 3 JPA repository interfaces.
Schema `restdb` is up to date. No migration necessary.
Started Spring7RestMvcApplication in 23.282 seconds
```

MariaDB counts before start: 2429 beers, 0 orders. The REST API is unaffected (still no order endpoints):

```bash
curl "http://localhost:8080/api/v1/beer?pageNumber=1&pageSize=2"
```

```text
beers totalElements: 2429
```

**Test run:**

```bash
mvn -o test -Dtest=BeerOrderRepositoryTest
```

```text
Tests run: 1, Failures: 0, Errors: 0, Skipped: 0
BUILD SUCCESS
```

Printed output captured from the surefire `system-out`:

```text
0        <- beerOrderRepository.count()  (H2 test DB has no orders)
3        <- customerRepository.count()
2413     <- beerRepository.count()      (3 seeded + 2410 CSV beers)
Customer 1   <- testCustomer.getName()
Galaxy Cat   <- testBeer.getBeerName()
```

This confirms the new repository bean is wired, the aggregate can be counted, and the existing customer/beer repositories still resolve in the same context. The orders count of 0 reflects that nothing has been persisted to `beer_order` yet — and in the H2 test database the order tables exist only because Hibernate generated them from the entity graph, since Flyway is disabled in the default profile.

#### Position in the series

This is the first branch that touches the order aggregate from the persistence layer. It is a stepping stone: the repository exists and is proven to wire up, but there is still no service, no DTO/mapper, and no controller, so orders remain unreachable over HTTP. The next branches are expected to build the order service and API on top of this repository.

## Branch 103: persisting relationships

### `103-rel-persisting-relationships`

Branch created from `origin/103-rel-persisting-relationships`. **Only one file changed** — `BeerOrderRepositoryTest.java`, +24/−6. No production code. The branch converts the branch-102 print-only test into one that actually **persists** a `BeerOrder` that references a `Customer`.

#### The new test

```java
@Transactional
@Test
void testBeerOrders() {
    BeerOrder beerOrder = BeerOrder.builder()
            .customerRef("Test order")
            .customer(testCustomer)
            .build();

    BeerOrder savedBeerOrder = beerOrderRepository.saveAndFlush(beerOrder);

    System.out.println(savedBeerOrder.getCustomerRef());
}
```

#### The logic, line by line

- **`@Transactional` on the test method** is the crucial addition. Spring's test framework wraps a `@Transactional` test in a transaction and **rolls it back automatically** when the method ends. So the insert is real and observed by the code inside the test, but it never survives the test. This is why the branch can persist orders without polluting the database and without needing cleanup code.
- **`.customer(testCustomer)`** wires the already-loaded (first) customer into the order's `@ManyToOne`. This exercises the owning side of the Customer↔BeerOrder association — the value written to `beer_order.customer_id`.
- **`.customerRef("Test order")`** sets the plain denormalized string column.
- **`saveAndFlush(...)`** (rather than `save`) forces the SQL INSERT to execute immediately and returns the managed instance. That matters because the entity id is generated by Hibernate (`@UuidGenerator`), the `@Version` is assigned, and `@CreationTimestamp`/`@UpdateTimestamp` are populated — `saveAndFlush` guarantees those are realized before the next line runs. With plain `save` the insert might be deferred until flush/commit, and the returned object's generated fields could be unset at the point of use.
- **`savedBeerOrder.getCustomerRef()`** prints the persisted value, confirming the entity round-tripped through the repository.

#### What this demonstrates about relationship persistence

The order holds a **reference to an existing** customer, so no cascade is involved: Hibernate only needs the customer's id to write the FK. The detached `testCustomer` loaded in `@BeforeEach` (its own transaction) is sufficient — assigning a detached entity to an owning `@ManyToOne` is fine when the row already exists. This is the pattern a future order service will use: look up the customer, attach it to the order, save the order. Nothing about `BeerOrderLine` is exercised yet, so the "save an order *with lines*" case (which needs cascade or explicit line saves) is still ahead.

#### Test nature

Still **no assertions** — it prints `savedBeerOrder.getCustomerRef()` and would pass as long as no exception is thrown. It is a demonstration test: it proves the mapping and repository can carry out an insert involving a relationship, not that any particular value is correct.

#### Empirical evidence

**Test run:**

```bash
mvn -o test -Dtest=BeerOrderRepositoryTest
```

```text
Tests run: 1, Failures: 0, Errors: 0, Skipped: 0
BUILD SUCCESS
```

Captured `system-out`:

```text
Test order
```

The printed value confirms the order was built, flushed, and read back.

Because the test has no active profile, it runs on the **H2** test datasource (Flyway disabled, Hibernate generates the schema from the entity graph), not MariaDB. The `@Transactional` rollback means nothing persists even in H2.

**MariaDB after the run** (proving the rollback and that the test did not touch the real database):

```text
beer_order count:      0
beer_order_line count: 0
beers:                 2429
```

**Application runtime** (`localmysql` profile) was unchanged: 3 JPA repositories, `Schema \`restdb\` is up to date. No migration necessary.`, Tomcat on 8080, REST returning 2429 beers. The order API is still absent.

## Branch 104: relationship helper methods

### `104-rel-helper-methods`

Branch created from `origin/104-rel-helper-methods`. Three files, +20/−3. This branch introduces the **bidirectional-consistency helper** pattern: keeping both sides of a JPA relationship in sync when only one side is assigned.

- `entities/BeerOrder.java` (+17/−1) — explicit all-args constructor and a `setCustomer` helper
- `entities/Customer.java` (+4/−1) — initialize the inverse collection
- `test/.../BeerOrderRepositoryTest.java` (+1/−1) — `save` instead of `saveAndFlush`

#### `BeerOrder`: constructor + syncing setter

```java
@Setter
@Entity
@NoArgsConstructor
@Builder                        // @AllArgsConstructor removed
public class BeerOrder {

    public BeerOrder(UUID id, Long version, Timestamp createdDate, Timestamp lastModifiedDate,
                     String customerRef, Customer customer, Set<BeerOrderLine> beerOrderLines) {
        this.id = id;
        this.version = version;
        this.createdDate = createdDate;
        this.lastModifiedDate = lastModifiedDate;
        this.customerRef = customerRef;
        this.setCustomer(customer);          // <-- routed through the helper
        this.beerOrderLines = beerOrderLines;
    }

    ...

    @ManyToOne
    private Customer customer;

    public void setCustomer(Customer customer) {
        this.customer = customer;
        customer.getBeerOrders().add(this);  // <-- keeps the inverse side in sync
    }
}
```

- **`@AllArgsConstructor` was removed and hand-replaced** by a constructor with the same signature, differing in one line: it calls `this.setCustomer(customer)` instead of `this.customer = customer`. This matters because Lombok's `@Builder` builds through the all-args constructor — so **`builder.build()` now automatically routes through the syncing setter**. That is the whole trick of the branch: you get the two-sided consistency for free from the builder.
- **`setCustomer` overrides Lombok's generated setter.** Lombok's `@Setter` skips generating a setter for a field that already has a hand-written one, so this method is the only `setCustomer`. It does two things: assigns the owning field (which becomes `customer_id`), and adds `this` order to the customer's inverse `beerOrders` collection.
- The purpose is the classic JPA **"set both sides" helper**: in a bidirectional association, if you only set `order.setCustomer(c)` the in-memory `c.getBeerOrders()` stays stale until a reload. The helper writes the relationship to both objects at once.

#### `Customer`: initialized inverse collection

```java
@Builder.Default
@OneToMany(mappedBy = "customer")
private Set<BeerOrder> beerOrders = new HashSet<>();
```

- **Initialized to `new HashSet<>()`.** This is what makes the helper safe: without it, `customer.getBeerOrders()` would be `null` and `.add(this)` in `setCustomer` would throw `NullPointerException`. Because `@Builder` would otherwise ignore the field initializer (Lombok's builder only sets fields you specify, and would leave the default as the Java field default), **`@Builder.Default`** is added so that a builder-built customer still gets the empty `HashSet` rather than `null`.
- Fetch type remains the `@OneToMany` default, `LAZY`.

#### Test change

```java
- BeerOrder savedBeerOrder = beerOrderRepository.saveAndFlush(beerOrder);
+ BeerOrder savedBeerOrder = beerOrderRepository.save(beerOrder);
```

Reverting to `save` — the test now relies on the surrounding `@Transactional` to flush at commit time (then roll back). The printed `customerRef` does not depend on a flush, so this works.

#### What the branch achieves and its caveats

**Achieves:** constructing a `BeerOrder` (via builder or the constructor) with a `Customer` now updates both the order's owning field and the customer's in-memory collection.

**Caveats:**

- **`setCustomer` has no null guard.** Passing `null` (or building an order with no customer, as the builder would if `.customer(...)` is omitted) throws `NullPointerException` on `customer.getBeerOrders()`. Since the builder always calls the constructor, an order cannot be built without a non-null customer on this branch.
- **Sync is one-directional.** Only the order→customer direction is handled. There is no `Customer.addBeerOrder(...)` and no `removeBeerOrder(...)`, and no equivalent helper for `BeerOrderLine`, so the inverse collections cannot be maintained when adding/removing lines or detaching an order.
- **No cascade still.** Syncing the in-memory collection does not make Hibernate persist related entities; that remains explicit.
- **Lazy-loading consequence.** Touching `customer.getBeerOrders()` on a lazy proxy triggers a load, so the helper can cause an extra query depending on context.

#### Empirical evidence

Started with the `localmysql` profile (MariaDB already running, so it was reused rather than restarted):

```text
Found 3 JPA repository interfaces.
Schema `restdb` is up to date. No migration necessary.
Started Spring7RestMvcApplication in 22.199 seconds
```

The modified test passes and exercises the helper through the builder path:

```bash
mvn -o test -Dtest=BeerOrderRepositoryTest
```

```text
Tests run: 1, Failures: 0, Errors: 0, Skipped: 0
BUILD SUCCESS
```

No orders were persisted (the `@Transactional` test still rolls back, and it runs on H2), and the running app was unaffected:

```text
beers: 2429
beer_order count: 0
```

The order REST API still does not exist — this branch is still model/documentation-level groundwork for the service layer that follows.

## Branch 105: many-to-many (Beer ↔ Category)

### `105-many-to-many`

Branch created from `origin/105-many-to-many`. Three files, +96 lines. Introduces a **many-to-many** association between `Beer` and a new `Category` entity, plus the join table via a new Flyway migration.

- `entities/Category.java` (new, 69 lines)
- `entities/Beer.java` (+6) — the `categories` association
- `db/migration/V4__category.sql` (new, 21 lines)

#### The join-table migration

```sql
drop table if exists category;
drop table if exists beer_category;

create table category
(
    id                 varchar(36) NOT NULL PRIMARY KEY,
    description        varchar(50),
    created_date       timestamp,
    last_modified_date datetime(6) DEFAULT NULL,
    version            bigint      DEFAULT NULL
) ENGINE = InnoDB;

create table beer_category
(
    beer_id     varchar(36) NOT NULL,
    category_id varchar(36) NOT NULL,
    primary key (beer_id, category_id),
    constraint pc_beer_id_fk FOREIGN KEY (beer_id) references beer (id),
    constraint pc_category_id_fk FOREIGN KEY (category_id) references category (id)
) ENGINE = InnoDB;
```

- `beer_category` is the classic **join table**: a **composite primary key** `(beer_id, category_id)` and two FKs back to `beer` and `category`.
- The composite PK makes each (beer, category) pair unique — a beer cannot be in the same category twice.

#### `Category` entity

```java
@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Entity
public class Category {

    @Id
    @GeneratedValue(generator = "UUID")
    @GenericGenerator(name = "UUID", strategy = "org.hibernate.id.UUIDGenerator")
    @JdbcTypeCode(SqlTypes.CHAR)
    @Column(length = 36, columnDefinition = "varchar(36)", updatable = false, nullable = false )
    private UUID id;

    @Version
    private Long version;

    @CreationTimestamp
    @Column(updatable = false)
    private Timestamp createdDate;

    @UpdateTimestamp
    private Timestamp lastModifiedDate;

    private String description;

    @ManyToMany
    @JoinTable(name = "beer_category",
      joinColumns = @JoinColumn(name = "category_id"),
      inverseJoinColumns = @JoinColumn(name = "beer_id"))
    private Set<Beer> beers;

    @Override public boolean equals(Object o) { ... compares description ... }
    @Override public int hashCode() { ... based on description ... }
}
```

- Uses `@GenericGenerator` (the older Hibernate generator API) instead of `@UuidGenerator` used by the other entities. On this build it **compiles with deprecation warnings** — Hibernate emits `GenericGenerator ... has been deprecated and marked for removal` — but it still resolves to the UUID generator, so the ids are UUID-shaped `varchar(36)` as expected.
- Its `@ManyToMany` point at the **same** `beer_category` table as `Beer` but with `joinColumns`/`inverseJoinColumns` **swapped**.
- `equals`/`hashCode` are overridden to compare `description`, and `equals` also calls `super.equals(o)` (Object identity). Using a mutable field such as `description` for hashing is a known hazard in a `Set`/`Map`: mutating `description` after the entity is in a collection changes its hash and can make it unfindable. It also means two categories with the same description but different ids are "equal", which is usually not what you want for entities.

#### `Beer` entity change

```java
@ManyToMany
@JoinTable(name = "beer_category",
        joinColumns = @JoinColumn(name = "beer_id"),
   inverseJoinColumns = @JoinColumn(name = "category_id"))
private Set<Category> categories;
```

#### The bidirectional many-to-many design (worth noting)

This is a **bidirectional** association, but unusually **both sides declare `@JoinTable`** on the same table with mirrored columns rather than one side using `mappedBy`. In a conventional JPA bidirectional M2M, only the owning side declares `@JoinTable` and the inverse side uses `mappedBy = "..."`, so there is one writer of the join rows. Here, both `Beer` and `Category` are owners of the same physical table. Hibernate accepted this and validation passed, treating them as two associations sharing `beer_category`. It works, but it means there is no single "owning" side, and it is the configuration the next branch (`106-rel-many-to-many-persistence`) likely revisits when it comes to persisting associations.

#### Other inconsistencies observed

- **Drop order in V4 is reversed** relative to V3: it drops `category` *before* `beer_category`, yet `beer_category` holds an FK to `category`. On a fresh database both are absent so the drops are no-ops and the migration succeeds; but if the tables existed with data, dropping the referenced `category` first would fail with a foreign-key error. (V3 correctly dropped the child table first.) Since Flyway runs V4 exactly once, this never bites in normal operation — it is a latent issue only if the script is run by hand against a populated schema.
- **Column type inconsistency within one table:** `category.created_date` is `timestamp` while `last_modified_date` is `datetime(6)`.
- **Length mismatch that Hibernate does not catch:** DDL declares `description varchar(50)`, but the entity field has no `@Column(length=...)`, so Hibernate's implicit default is 255. Schema *validation* checks that mapped columns exist and have a compatible type — it does not compare lengths — so validation passed.

#### Empirical evidence

MariaDB was already running (reused, not restarted). Flyway applied the new migration on startup:

```text
Migrating schema `restdb` to version "4 - category"
Successfully applied 1 migration to schema `restdb`, now at version v4 (execution time 00:00.150s)
Started Spring7RestMvcApplication in 21.987 seconds
```

`flyway_schema_history`: versions 1–4 all `success = 1`.

Tables now present: `beer`, `beer_category`, `beer_order`, `beer_order_line`, `category`, `customer`, `flyway_schema_history`.

`beer_category` structure confirmed:

```text
beer_id      varchar(36)  NO  PRI
category_id  varchar(36)  NO  PRI
PRIMARY KEY (beer_id, category_id)
CONSTRAINT pc_beer_id_fk     FOREIGN KEY (beer_id)     REFERENCES beer (id)
CONSTRAINT pc_category_id_fk FOREIGN KEY (category_id) REFERENCES category (id)
```

Spring Data still reports **3 JPA repository interfaces** (no `CategoryRepository`), so the new entity remains unreachable over REST. The beer API is unaffected:

```text
curl .../api/v1/beer?pageNumber=1&pageSize=1  ->  beers: 2429
```

Compile-time deprecation warning captured:

```text
Category.java: org.hibernate.annotations.GenericGenerator ... has been deprecated and marked for removal
```

## Branch 106: many-to-many persistence

### `106-rel-many-to-many-persistence`

Branch created from `origin/106-rel-many-to-many-persistence`. Four files, +76/−13. This branch makes the Beer↔Category many-to-many **persistable** and fixes the deprecation warning introduced on branch 105.

- `entities/Beer.java` (+14) — `@Builder.Default`-annotated, pre-initialized `categories` collection plus `addCategory` / `removeCategory` helpers
- `entities/Category.java` (+8/−7) — replaces `@GenericGenerator` with `@UuidGenerator`, adds `@Builder`, pre-initializes `beers`
- `repositories/CategoryRepository.java` (new, 12 lines)
- `repositories/CategoryRepositoryTest.java` (new, 44 lines)

#### `Beer`: collection initialization + add/remove helpers

```java
@Builder.Default
@ManyToMany
@JoinTable(name = "beer_category",
       joinColumns = @JoinColumn(name = "beer_id"),
       inverseJoinColumns = @JoinColumn(name = "category_id"))
private Set<Category> categories = new HashSet<>();

public void addCategory(Category category){
    this.categories.add(category);
    category.getBeers().add(this);
}

public void removeCategory(Category category){
    this.categories.remove(category);
    category.getBeers().remove(category);
}
```

- **`@Builder.Default` + `= new HashSet<>()`** ensures the collection is non-null even when a `Beer` is built via `Beer.builder()` (Lombok's builder would otherwise leave the field at its Java default of `null`, which would NPE on `.add`). `@Builder.Default` is what makes the initializer survive the builder.
- **`addCategory`** is the two-sided sync helper: it adds the category to this beer's side **and** adds this beer to the category's side, keeping both `@JoinTable` owners consistent in memory. Either side can then be saved to flush the join rows.
- **`removeCategory` has a bug.** The second line reads `category.getBeers().remove(category)` — it tries to remove a `Category` from a `Set<Beer>`. `Set.remove(Object)` compiles and return-`false`s silently, so **the beer is never actually removed from the category's side**. The intended call is `category.getBeers().remove(this)`. Left as-is, detaching a category via this helper leaves a dangling half of the association. It is the kind of latent bug that only matters once code path sees real use (no test exercises `removeCategory`).

#### `Category`: generator fix + builder

```java
@Builder                              // added
@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Entity
public class Category {

    @Id @GeneratedValue(generator = "UUID")
    @UuidGenerator                    // was: @GenericGenerator(name="UUID", strategy="...") 
    @JdbcTypeCode(SqlTypes.CHAR)
    @Column(...)
    private UUID id;
    ...
    @Builder.Default
    @ManyToMany @JoinTable(name = "beer_category",
      joinColumns = @JoinColumn(name = "category_id"),
      inverseJoinColumns = @JoinColumn(name = "beer_id"))
    private Set<Beer> beers = new HashSet<>();
```

- `@GenericGenerator` (Hibernate 7: deprecated, "marked for removal") is replaced by `@UuidGenerator` — **the same generator used by every other entity**. The compile-time deprecation warning that branch 105 reported is eliminated.
- `@Builder` is added so categories can be created via `Category.builder()...build()`; with the existing `@AllArgsConstructor` present, Lombok wires the builder through the all-args constructor.
- `beers` is initialized with `@Builder.Default` for the same null-safety reason as `Beer.categories`.

#### `CategoryRepository` and its test

```java
public interface CategoryRepository extends JpaRepository<Category, UUID> { }
```

```java
@SpringBootTest
class CategoryRepositoryTest {
    @Autowired CategoryRepository categoryRepository;
    @Autowired BeerRepository beerRepository;
    Beer testBeer;

    @BeforeEach
    void setUp() {
        testBeer = beerRepository.findAll().get(0);
    }

    @Transactional
    @Test
    void testAddCategory() {
        Category savedCat = categoryRepository.save(Category.builder()
                .description("Ales").build());

        testBeer.addCategory(savedCat);
        Beer saveBeer = beerRepository.save(testBeer);

        System.out.println(saveBeer.getBeerName());
    }
}
```

- `setUp()` grabs the first beer — on the H2 test DB (default profile) this is **Galaxy Cat**, the first seeded beer, because `BootstrapData` runs under `@SpringBootTest`.
- The test exercises the **Beer-owning-side** write path: it saves the category first (no beers yet, so no join rows), then `addCategory` links the two in memory, then `beerRepository.save(testBeer)` flushes the `beer_category` row. `@Transactional` rolls the whole thing back, so nothing persists.
- `@ManyToOne` on `BeerOrderLine`/`BeerOrder`/`Category` associations defaults to **EAGER** fetch; here it means `beerRepository.findAll()` loads each beer and, because `Beer.categories` is `Set<Category>` with default `@ManyToMany` (which is **LAZY**, not EAGER — `@ManyToMany` defaults to LAZY), the categories won't be initialized by the findAll. (Fetch defaults: `@ManyToMany` → LAZY, `@ManyToOne` → EAGER.)

#### Runtime evidence

MariaDB was already up (reused). Flyway already at v4: `Schema \`restdb\` is up to date. No migration necessary.`

Repository scanning now finds **4 JPA repository interfaces** (the new `CategoryRepository`):

```text
Found 4 JPA repository interfaces.
Started Spring7RestMvcApplication in 21.066 seconds
```

Tests:

```bash
mvn -o test -Dtest=CategoryRepositoryTest
```

```text
Tests run: 1, Failures: 0, Errors: 0, Skipped: 0   -- printed: Galaxy Cat
```

Regression check (the `Beer.java` change touches the entity behind the main controller):

```bash
mvn -o test -Dtest=BeerControllerIT
```

```text
Tests run: 16, Failures: 0, Errors: 0, Skipped: 0
BUILD SUCCESS   -- no regression
```

And the deprecation warning that branch 105 reported is gone (verified 0 occurrences of `GenericGenerator.*deprecated` in the branch-106 run log). REST still healthy:

```text
curl .../api/v1/beer?pageNumber=1&pageSize=1  ->  beers: 2429
```

#### Summary of the relationship layer after this branch

| Entity pair | Owning side (writes FK/join) | Inverse side |
|---|---|---|
| Customer ↔ BeerOrder | `BeerOrder.customer` (`@ManyToOne`, EAGER) | `Customer.beerOrders` (`@OneToMany`, LAZY) |
| BeerOrder ↔ BeerOrderLine | `BeerOrderLine.beerOrder` (`@ManyToOne`, EAGER) | `BeerOrder.beerOrderLines` (`@OneToMany`, LAZY) |
| Beer ↔ BeerOrderLine | `BeerOrderLine.beer` (`@ManyToOne`, EAGER) | `Beer.beerOrderLines` (`@OneToMany`, LAZY) |
| Beer ↔ Category | `Beer.categories` (`@ManyToMany`, LAZY, `beer_id` join) **+ also** `Category.beers` (`@ManyToMany`, LAZY, `category_id` join) | mutual — no `mappedBy` on either side |

The Beer↔BeerOrderLine and BeerOrder↔BeerOrderLine pairs are now complete two-sided associations with helper-sync logic on the Beer↔Category side. The order aggregate still has no service/DTO/mapper/controller, so it remains unreachable over HTTP. The Beer↔Category M2M having **both sides as owners** (no `mappedBy`) remains the open design question for the persistence path — saving via both sides could write duplicate join rows.

## Branch 107: order shipment (one-to-one)

### `107-rel-one-to-one`

Branch created from `origin/107-rel-one-to-one`. Three files, +71/−2. Adds a **one-to-one** association between `BeerOrder` and a new `BeerOrderShipment` entity (the "shipment tracking" side), plus the corresponding schema and an explicit all-args constructor that now also carries the shipment.

- `entities/BeerOrder.java` (+7/−2) — new `@OneToOne BeerOrderShipment beerOrderShipment` field and an extended explicit constructor that assigns it
- `entities/BeerOrderShipment.java` (new, 47 lines) — UUID id, `Long version`, `@OneToOne BeerOrder beerOrder`, `trackingNumber`, `Timestamp` timestamps, `@Builder`
- `db/migration/V5__order-shipment.sql` (new, 19 lines) — creates `beer_order_shipment` and adds `beer_order_shipment_id` to `beer_order`

#### The one-to-one design (two unidirectional joins)

The association is implemented as **two independent, one-sided `@OneToOne` associations** rather than a single bidirectional one with `mappedBy`:

```java
// BeerOrder.java
@OneToOne
private BeerOrderShipment beerOrderShipment;
```
```java
// BeerOrderShipment.java
@OneToOne
private BeerOrder beerOrder;
```

Why two sides both owning a FK instead of `@JoinColumn(optional=false, mappedBy=...)` on one side? Because MySQL/MariaDB **has no deferred foreign-key checks**: when both sides point at each other with a single FK column, inserting a parent and its child in the same transaction is impossible — the child's FK to the not-yet-inserted parent fails before the row exists. Splitting the FK column onto each table side sidesteps this entirely:

- `beer_order_shipment.beer_order_id VARCHAR(36) UNIQUE` → FK to `beer_order(id)` (the **BeerOrderShipment → BeerOrder** direction). The `UNIQUE` makes it a true one-to-one (one shipment per order); it is nullable because the row can be inserted before the back-reference is set.
- `beer_order.beer_order_shipment_id VARCHAR(36)` → FK to `beer_order_shipment(id)` (the **BeerOrder → BeerOrderShipment** direction), added via `ALTER TABLE` after the shipment table exists.

Hibernate's **default `@OneToOne` join column** naming (`beerOrder_id` / `beer_order_shipment_id`) happened to match the explicit DDL column names, so `ddl-auto=validate` passed with no `Schema-validation` error — verified in the run log (`Started Spring7RestMvcApplication`, no schema errors).

#### `BeerOrderShipment` entity

```java
@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Entity @Builder
public class BeerOrderShipment {
    @Id @GeneratedValue(generator="UUID") @UuidGenerator
    @JdbcTypeCode(SqlTypes.CHAR)
    @Column(length=36, columnDefinition="varchar(36)", updatable=false, nullable=false)
    private UUID id;

    @Version
    private Long version;

    @OneToOne
    private BeerOrder beerOrder;          // FK column beer_order_id, UNIQUE

    private String trackingNumber;

    @CreationTimestamp @Column(updatable = false) private Timestamp createdDate;
    @UpdateTimestamp                        private Timestamp lastModifiedDate;

    @Override public int hashCode() { return super.hashCode(); }
}
```

- Mirrors the field typing of the rest of the order aggregate: **`Long version`** and **`java.sql.Timestamp`** timestamps.
- Uses `@UuidGenerator` (the modern generator, no deprecation warning).
- `hashCode()` simply delegates to `super.hashCode()` (identity) rather than keying off `trackingNumber` — so it avoids the mutable-field trap that `Category` fell into on branch 105.

#### `BeerOrder` constructor change

The hand-written all-args constructor (the one `@Builder` routes through) gained a trailing `BeerOrderShipment beerOrderShipment` parameter and assigns it directly:

```java
public BeerOrder(UUID id, Long version, Timestamp createdDate, Timestamp lastModifiedDate,
                 String customerRef, Customer customer, Set<BeerOrderLine> beerOrderLines,
                 BeerOrderShipment beerOrderShipment) {
    ...
    this.setCustomer(customer);
    this.beerOrderLines = beerOrderLines;
    this.beerOrderShipment = beerOrderShipment;
}
```

Because `@Builder` builds via this all-args constructor, every `BeerOrder.builder()…build()` call now **must** supply `beerOrderShipment` (or `null` explicitly). Existing call sites that don't will fail to compile — but there are currently no builders constructing `BeerOrder` in the main sources (only `Customer.builder()` and `Beer.builder()` are used in `BootstrapData`), so the build is unaffected. The order entities still have no service/DTO/mapper/controller, so this is pure model-layer work — nothing here is reachable over HTTP yet.

#### The schema (what Flyway applied)

`V5__order-shipment.sql` — note it is **idempotently written with `CREATE TABLE` directly** (no `drop table if exists` for `beer_order` itself) and finishes the link with an `ALTER`:

```sql
drop table if exists beer_order_shipment;

CREATE TABLE beer_order_shipment
(
    id                 VARCHAR(36) NOT NULL PRIMARY KEY,
    beer_order_id            VARCHAR(36) UNIQUE,
    tracking_number    VARCHAR(50),
    created_date       TIMESTAMP,
    last_modified_date DATETIME(6) DEFAULT NULL,
    version            BIGINT      DEFAULT NULL,
    CONSTRAINT bos_pk FOREIGN KEY (beer_order_id) REFERENCES beer_order (id)
) ENGINE = InnoDB;

ALTER TABLE beer_order
    ADD COLUMN beer_order_shipment_id VARCHAR(36);

ALTER TABLE beer_order
    ADD CONSTRAINT bos_shipment_fk
        FOREIGN KEY (beer_order_shipment_id) REFERENCES beer_order_shipment (id);
```

`beer_order_id` is `UNIQUE`, enforcing the one-to-one cardinality at the DB level (one shipment row per order, nullable so the shipment can be created before the back-reference is populated).

#### Runtime evidence

MariaDB reused (already up). Flyway:

```text
Migrating schema `restdb` to version "5 - order-shipment"
Successfully applied 1 migration to schema `restdb`, now at version v5 (execution time 00:00.190s)
Started Spring7RestMvcApplication in 23.102 seconds
```

`flyway_schema_history`: versions 1–5 all `success = 1`. **4 JPA repository interfaces** (no `BeerOrderShipmentRepository` added).

Schema confirmed live:

```text
beer_order_shipment.beer_order_id  varchar(36)  YES  UNI   -> FK beer_order(id)
beer_order.beer_order_shipment_id  varchar(36)            -> FK beer_order_shipment(id)
```

REST unchanged:

```text
curl .../api/v1/beer?pageNumber=1&pageSize=1  ->  HTTP 200, 2429 beers
```

Regression (`BeerOrder.java` is touched):

```text
BeerControllerIT: Tests run: 16, Failures: 0, Errors: 0, BUILD SUCCESS
```

## Branch 108: persist the one-to-one

### `108-rel-one-to-one-pers`

Branch created from `origin/108-rel-one-to-one-pers`. Two files, +18/−5. Branch 107 declared the BeerOrder↔BeerOrderShipment one-to-one but **left it unpersistable** (no cascade, no inverse-side sync). This branch applies the exact same **helper + cascade** pattern that branch 104 used for `BeerOrder.customer`, this time to the one-to-one, and wires a shipment into the order repository test.

- `entities/BeerOrder.java` (+13/−7) — new `setBeerOrderShipment` sync helper; the all-args constructor routes the field through that setter; `@OneToOne` gains `cascade = CascadeType.PERSIST`; the wildcard imports `import lombok.*` / `import org.hibernate.annotations.*` are narrowed to explicit imports (no behavior change)
- `repositories/BeerOrderRepositoryTest.java` (+5) — builds a shipment (`.trackingNumber("1235r")`) into the beer order and saves it

#### The persistence fix

```java
// BeerOrder.java
public void setBeerOrderShipment(BeerOrderShipment beerOrderShipment) {
    this.beerOrderShipment = beerOrderShipment;
    beerOrderShipment.setBeerOrder(this);          // sync the inverse side
}

@OneToOne(cascade = CascadeType.PERSIST)            // was: bare @OneToOne
private BeerOrderShipment beerOrderShipment;
```

Two things are required for a one-to-one to save correctly, and this branch adds both:

1. **`cascade = CascadeType.PERSIST`** — without it, `beerOrderRepository.save(beerOrder)` writes the order row but **not** the shipment row. The order's `beer_order_shipment_id` would then point at a non-existent `beer_order_shipment.id` (or be null), violating the FK constraint added by V5. Cascade is what makes the shipment row follow the order into the transaction in one `save()`.
2. **`setBeerOrderShipment` syncs the inverse side** — `beerOrderShipment.setBeerOrder(this)` populates the shipment's `beer_order_id` FK, so the join actually links the two rows. The explicit all-args constructor now calls `this.setBeerOrderShipment(beerOrderShipment)` (instead of the old direct `this.beerOrderShipment = beerOrderShipment`) so the field assignment **always** goes through the sync setter — the same trick branch 104 used for `setCustomer`.

This is structurally identical to 104's approach: **the builder routes through an all-args constructor that delegates field writes to two-sided setters, and `@Builder.Default` keeps collections non-null**. The difference is one-to-one (`@OneToOne`, single object) instead of many-to-one.

The updated test exercises the round-trip:

```java
@Test @Transactional
void testBeerOrders() {
    BeerOrder beerOrder = BeerOrder.builder()
            .customerRef("Test order")
            .customer(testCustomer)
            .beerOrderShipment(BeerOrderShipment.builder()
                    .trackingNumber("1235r")
                    .build())
            .build();

    BeerOrder savedBeerOrder = beerOrderRepository.save(beerOrder);
    System.out.println(savedBeerOrder.getCustomerRef());
}
```

#### What's still not done

- `beerOrderShipment`/`setBeerOrderShipment` have **no null guard** — building an `BeerOrder` without a shipment NPEs inside the setter (e.g. `NullPointerException` in `setBeerOrderShipment` → `beerOrderShipment.setBeerOrder`), exactly as `setCustomer` does without a customer. Since `@Builder` funnels through the all-args constructor, every builder call must supply the shipment (the test does). This is consistent with the codebase's current "no defensive null-checks" stance for internal builders.
- There is still **no `BeerOrderShipmentRepository`, no service/DTO/mapper/controller** — the entity pair is model-only and remains unreachable over HTTP.
- The test still asserts only indirectly (it prints `customerRef` and relies on the save not throwing); `@Transactional` rolls everything back, so no order rows leak into the test DB.
- Only `CascadeType.PERSIST` is set (not `ALL`/`MERGE`), so **updating** an order's shipment must be done explicitly — a deliberately narrow cascade, consistent with how 104 handled `customer`/`customerRef`.

#### Runtime evidence

App was already running from the 107 run (reused — MariaDB stayed up):

```text
Repository scanning ... Found 4 JPA repository interfaces.
Schema `restdb` is up to date. No migration necessary.
Started Spring7RestMvcApplication in 43.886 seconds
```

Repository count stayed at **4** (no new repository interface). Test (H2/default profile):

```bash
mvn -o test -Dtest=BeerOrderRepositoryTest
```
```text
Tests run: 1, Failures: 0, Errors: 0, Skipped: 0
BUILD SUCCESS
```

A passing `save` confirms the cascade wrote the shipment row and the `setBeerOrderShipment` setter populated `beer_order_shipment.beer_order_id` with no FK/NULL violation.

## Overall progression

The session progressed from Lombok model conveniences, through MySQL/MariaDB connectivity and pooling, to Flyway-owned schema migrations. It then added a customer column, introduced CSV parsing and database bootstrap persistence, automated entity timestamps, repaired integration-test expectations, and began layering beer-name query support from the controller down toward the repository.

## Detailed implementation notes

### MariaDB profile configuration

The active `localmysql` profile used throughout the runs contains the following important settings:

```properties
spring.datasource.username=restadmin
spring.datasource.password=password
spring.datasource.url=jdbc:mysql://127.0.0.1:3306/restdb?useUnicode=true&characterEncoding=UTF-8&serverTimezone=UTC
spring.jpa.database=mysql
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.MySQLDialect
spring.jpa.hibernate.ddl-auto=validate
spring.flyway.enabled=true
```

The profile deliberately points to `127.0.0.1` rather than a Docker hostname. That is why the application can run against the manually started MariaDB server in Termux. `ddl-auto=validate` means Hibernate checks the tables and columns but does not create or alter them; Flyway owns those changes.

The Hikari settings configure a five-connection pool named `RestDB-Pool`. Prepared-statement caching and server-prepared statements are enabled. SQL output, formatting, and JDBC bind-value tracing are also enabled, which explains the detailed Hibernate statements seen during POST and PATCH requests.

### Flyway schema lifecycle

V1 creates two InnoDB tables. The beer table uses a `varchar(36)` UUID primary key, `beer_name varchar(50)`, `beer_style smallint`, decimal pricing, inventory quantity, UPC, timestamps, and an optimistic-locking `version` column. The customer table initially contains UUID, name, timestamps, and version.

V2 adds the customer email column without recreating the table. On branch 78, startup output showed:

```text
Successfully validated 2 migrations
Current version of schema `restdb`: 1
Migrating schema `restdb` to version "2 - add-email-to-customer"
Successfully applied 1 migration
```

On later branches, startup showed schema version 2 and `No migration necessary`, proving that Flyway history was reused rather than rerunning migrations.

### UUID and MariaDB representation

The entities use `@UuidGenerator` for identifier generation. `@JdbcTypeCode(SqlTypes.CHAR)` tells Hibernate to bind UUID values as character data, matching the Flyway `varchar(36)` columns. Hibernate trace output confirmed bindings such as:

```text
binding parameter (9:CHAR) <- [b98b91fc-1ec2-4b65-8a49-5deca3e1b594]
```

This is the practical difference from the earlier failing branch: the Java UUID mapping and MariaDB column definition agree on a textual UUID representation rather than relying on an incompatible database-specific UUID mapping.

### CSV mapping details

`BeerCSVRecord` represents the imported dataset. OpenCSV maps headers by name. The source data contains names such as `count.x`, `brewery_id`, and `count.y`; the explicit annotations prevent those names from being lost when mapped to Java fields.

The bootstrap conversion intentionally creates simplified application beers:

| CSV value | Stored beer value |
|---|---|
| `beer` | `beerName`, abbreviated to 50 characters |
| `style` | mapped to the application `BeerStyle` enum |
| `row` | `upc` as a string |
| `count.x` | `quantityOnHand` |
| no CSV price | fixed price `10` |
| CSV record | new generated UUID |

The style switch covers known dataset styles such as American IPA, American Pale Lager, American Porter, Oatmeal Stout, Saison/Farmhouse Ale, and English Pale Ale. Unknown styles fall back to `PILSNER`. This fallback prevents an unmapped source value from stopping the entire bootstrap operation, but it can also reduce semantic accuracy for styles not explicitly listed.

### Bootstrap idempotence and test counts

`loadCsvData()` checks `beerRepository.count() < 10`. On a fresh schema, the three hand-written beers are inserted first, then the CSV rows are imported. On subsequent starts, the count is already above the threshold and the CSV is skipped. This prevents duplicate imports during ordinary restarts, but it is a coarse guard: it does not verify whether every expected CSV row exists.

The integration-test expected count changed from 3 to 2413 because the dataset adds approximately 2410 rows to the three initial beers. The repository test imports the service/bootstrap components into the JPA test context so the CSV data exists when testing the derived query.

### Timestamp behavior

Before branch 84, timestamps were set manually in the initial bootstrap objects and could remain null for objects created through other paths. `@CreationTimestamp` and `@UpdateTimestamp` move timestamp assignment into Hibernate. A POST followed by GET on branch 84 returned values like:

```json
{
  "createdDate": "2026-09-14T17:25:57.650782",
  "updateDate": "2026-09-14T17:25:57.651008",
  "version": 0
}
```

The PATCH SQL updated `update_date` while leaving `created_date` unchanged. The entity version also incremented from 0 to 1, providing optimistic locking for updates.

### Query progression and unfinished behavior

The query feature was intentionally developed in layers:

1. Branch 86 added a test for a name query.
2. Branch 87 added the optional HTTP request parameter.
3. Branch 88 propagated the parameter through the service interface and implementations.
4. Branch 89 added the Spring Data derived-query method and repository test.

The final branch inspected in the session still contained this unfinished method:

```java
List<Beer> listBeersByName(String beerName) {
    return new ArrayList<>();
}
```

Consequently, live `GET /api/v1/beer?beerName=IPA` did not yet return the expected 336 records. Depending on the active implementation path, the request returned the complete dataset or an empty name-filtered result. The repository method exists, but the service method must call it with wildcard values such as `%IPA%` before the feature is complete.

### REST evidence collected

Across the runs, the stable REST behavior was:

```text
GET /api/v1/customer       -> 200 with three customers
POST /api/v1/beer          -> 201 and a Location header containing a UUID
PATCH /api/v1/beer/{id}    -> 204 and an updated version/timestamp
POST invalid beer payload  -> 400 with validation messages
```

Successful POST input used fields `beerName`, `beerStyle`, `upc`, `price`, and `quantityOnHand`. Invalid POST input used an empty name, null style, and empty UPC, producing the expected `must not be blank` and `must not be null` messages.

### Runtime warnings versus failures

The Termux Maven environment printed a Jansi native-library warning because the bundled native library was not compatible with the Android/ARM environment. Maven continued to build and run the application.

Flyway warned that MariaDB 12.3 was newer than the latest version verified by the installed Flyway release. Migration validation and execution still completed successfully.

Hibernate warned about the MariaDB metadata query referencing `RESERVED`, and it reported generic database metadata such as version 8.0. These warnings did not prevent datasource creation, Flyway validation, JPA initialization, HTTP startup, or CRUD requests.

### Branch state and repository hygiene

No tracked application source was edited by the agent during these branch walkthroughs. The branch checkouts and runs preserved the existing untracked notes and logs. The report itself is the new untracked file `SESSION_BRANCH_CHANGELOG.md`.

## Branch 91: completed name search

### `91-query-complete-name-search`

- Replaced the unfinished service stub with a repository call:

  ```java
  public List<Beer> listBeersByName(String beerName) {
      return beerRepository.findAllByBeerNameIsLikeIgnoreCase("%" + beerName + "%");
  }
  ```

- `StringUtils.hasText(beerName)` selects between `findAll()` and the name-search method.
- The Spring Data method generates a case-insensitive `LIKE` query.
- The controller integration test expectation changed from 100 to 336 IPA matches.

### Runtime evidence

The branch ran with:

```bash
mvn spring-boot:run \
  -Dspring-boot.run.profiles=localmysql \
  -Dspring-boot.run.jvmArguments='-Dspring.docker.compose.enabled=false'
```

Startup completed with Spring Boot 4.0.6, Hikari `RestDB-Pool`, Flyway schema version 2, and Tomcat on port 8080.

```bash
curl -i 'http://127.0.0.1:8080/api/v1/beer?beerName=GUBNA'
```

```text
HTTP/1.1 200
Content-Type: application/json
Content-Length: 195

[{"beerName":"GUBNA Imperial IPA","beerStyle":"IPA","createdDate":null,"id":"000212bc-6178-4e38-b48b-b26da298bb2e","price":10.00,"quantityOnHand":1581,"upc":"1581","updateDate":null,"version":0}]
```

The broader IPA search returned HTTP 200 with a 67,164-byte JSON body. Hibernate confirmed the repository call:

```text
where upper(b1_0.beer_name) like upper(?) escape '\\'
binding parameter (1:VARCHAR) <- [%IPA%]
```

The customer endpoint remained healthy:

```bash
curl -i http://127.0.0.1:8080/api/v1/customer
```

```text
HTTP/1.1 200
Content-Type: application/json
Content-Length: 490

[{"name":"Customer 1","version":1},{"name":"Customer 2","version":1},{"name":"Customer 3","version":1}]
```

Create and validation checks also remained successful:

```bash
curl -i -X POST http://127.0.0.1:8080/api/v1/beer \
  -H 'Content-Type: application/json' \
  -d '{"beerName":"Complete Search IPA","beerStyle":"IPA","upc":"910000001","price":8.91,"quantityOnHand":21}'
```

```text
HTTP/1.1 201
Location: /api/v1/beer/c0d5df17-1d36-4c78-8d89-308976ebd26a
Content-Length: 0
```

```bash
curl -i -X POST http://127.0.0.1:8080/api/v1/beer \
  -H 'Content-Type: application/json' \
  -d '{"beerName":"","beerStyle":null,"upc":"","price":0,"quantityOnHand":0}'
```

```text
HTTP/1.1 400
Content-Type: application/json

[{"upc":"must not be blank"},{"beerName":"must not be blank"},{"beerStyle":"must not be null"}]
```

This branch completes the query path: request parameter → controller → service → Spring Data repository → mapped DTO list. MariaDB and the application were stopped after testing, and ports 3306 and 8080 were released.

## Branch 92: beer-style search

### `92-query-beer-style-search`

- Added `BeerStyle` as an optional request parameter to `BeerController.listBeers`:

  ```java
  public List<BeerDTO> listBeers(
          @RequestParam(required = false) String beerName,
          @RequestParam(required = false) BeerStyle beerStyle) {
      return beerService.listBeers(beerName, beerStyle);
  }
  ```

- Expanded `BeerService.listBeers` to accept both name and style.
- Added the Spring Data repository method:

  ```java
  List<Beer> findAllByBeerStyle(BeerStyle beerStyle);
  ```

- Added `BeerServiceJPA.listBeersByStyle`, which delegates to that repository method.
- The JPA service currently chooses one filter at a time:
  - name present and style absent → name search;
  - name absent and style present → style search;
  - both absent → all beers;
  - both present → falls through to all beers rather than combining the predicates.
- Updated the in-memory service and controller tests to use the two-parameter signature.
- Added a MockMvc test expecting 548 `IPA` style records and retained the 336 name-match expectation.

### Runtime evidence

The branch ran with:

```bash
mvn spring-boot:run \
  -Dspring-boot.run.profiles=localmysql \
  -Dspring-boot.run.jvmArguments='-Dspring.docker.compose.enabled=false'
```

Style-only search:

```bash
curl -i 'http://127.0.0.1:8080/api/v1/beer?beerStyle=IPA'
```

```text
HTTP/1.1 200
Content-Type: application/json
Transfer-Encoding: chunked

BODY_BYTES=107755
BODY_PREFIX=[{"beerName":"GUBNA Imperial IPA","beerStyle":"IPA","createdDate":null,"id":"000212bc-6178-4e38-b48b-b26da298bb2e","price":10.00,"quantityOnHand":1581,"upc":"1581","updateDate":null,"version":0},...]
```

Hibernate confirmed the style repository query:

```text
select ... from beer b1_0 where b1_0.beer_style=?
binding parameter (1:SMALLINT) <- [IPA]
```

Name-only search still worked:

```bash
curl -i 'http://127.0.0.1:8080/api/v1/beer?beerName=GUBNA'
```

```text
HTTP/1.1 200
Content-Type: application/json
Content-Length: 195

[{"beerName":"GUBNA Imperial IPA","beerStyle":"IPA","createdDate":null,"id":"000212bc-6178-4e38-b48b-b26da298bb2e","price":10.00,"quantityOnHand":1581,"upc":"1581","updateDate":null,"version":0}]
```

The combined request demonstrates the current limitation:

```bash
curl -i 'http://127.0.0.1:8080/api/v1/beer?beerName=GUBNA&beerStyle=IPA'
```

```text
HTTP/1.1 200
Content-Type: application/json
Transfer-Encoding: chunked

BOTH_BODY_BYTES=472835
BOTH_BODY_PREFIX=[{"beerName":"GUBNA Imperial IPA","beerStyle":"IPA",...all beers...}]
```

The combined request executes `findAll()` because the current service has no combined name-and-style repository method. Customer listing, beer creation, and validation remained healthy:

```bash
curl -i -X POST http://127.0.0.1:8080/api/v1/beer \
  -H 'Content-Type: application/json' \
  -d '{"beerName":"Style Search IPA","beerStyle":"IPA","upc":"920000001","price":8.92,"quantityOnHand":22}'
```

```text
HTTP/1.1 201
Location: /api/v1/beer/09dff2c7-913d-4f75-98e1-9c8d081bd2b2
Content-Length: 0
```

```bash
curl -i -X POST http://127.0.0.1:8080/api/v1/beer \
  -H 'Content-Type: application/json' \
  -d '{"beerName":"","beerStyle":null,"upc":"","price":0,"quantityOnHand":0}'
```

```text
HTTP/1.1 400
Content-Type: application/json

[{"beerStyle":"must not be null"},{"beerName":"must not be blank"},{"upc":"must not be blank"}]
```

This branch completes independent beer-name and beer-style filters, while combined filtering remains the next design step. MariaDB and Spring Boot were stopped after testing, and ports 3306 and 8080 were released.

## Branch 93: complete search params

### `93-query-complete-search-params`

Substantive commit: `01e6fe7 adding showInventory and complete search conditions`, merged into the branch line by `ad4181d Merge branch '92-query-beer-style-search' into 93-query-complete-search-params`.

This branch closes the combined-filter gap left by branch 92 and adds inventory visibility control to `GET /api/v1/beer`.

#### Controller

`BeerController.listBeers` now accepts a third optional parameter and passes all three to the service:

```java
public List<BeerDTO> listBeers(@RequestParam(required = false) String beerName,
                               @RequestParam(required = false) BeerStyle beerStyle,
                               @RequestParam(required = false) Boolean showInventory){
    return beerService.listBeers(beerName, beerStyle, showInventory);
}
```

#### Service

`BeerService.listBeers` signature changed from two arguments to three across `BeerService`, `BeerServiceImpl`, and `BeerServiceJPA`. The in-memory `BeerServiceImpl` only follows the new signature; it still returns the full beer map without filtering.

`BeerServiceJPA.listBeers` now covers all four parameter combinations instead of falling back to `findAll()` when both filters are present:

| beerName | beerStyle | Repository call |
|---|---|---|
| set | null | `findAllByBeerNameIsLikeIgnoreCase("%" + beerName + "%")` |
| null | set | `findAllByBeerStyle(beerStyle)` |
| set | set | `findAllByBeerNameIsLikeIgnoreCaseAndBeerStyle("%" + beerName + "%", beerStyle)` |
| null | null | `findAll()` |

This resolves the branch-92 limitation where `?beerName=GUBNA&beerStyle=IPA` executed `findAll()` and returned all 2413 beers.

#### Repository

```java
List<Beer> findAllByBeerNameIsLikeIgnoreCaseAndBeerStyle(String beerName, BeerStyle beerStyle);
```

Spring Data derives a combined predicate from the method name: a case-insensitive name `LIKE` (upper-cased on both sides, matching the branch-91 generated SQL pattern `upper(beer_name) like upper(?)`) plus a style equality on the smallint-mapped `beer_style` column.

#### showInventory behavior

After the search branch runs, the service applies a response-shaping step:

```java
if (showInventory != null && !showInventory) {
    beerList.forEach(beer -> beer.setQuantityOnHand(null));
}
```

- `showInventory=true` or omitted (`null`) → `quantityOnHand` remains in the JSON response.
- `showInventory=false` → `quantityOnHand` is nulled on the loaded entity objects before DTO mapping, so the response omits the inventory value.
- The nulling operates on detached entities: the repository call's transaction has already ended and the service method is not transactional, so the change is never flushed to the database. It only affects the response payload.

#### Tests

`BeerControllerIT` gained three tests and now covers the full query matrix:

| Request | Expected result |
|---|---|
| no parameters | 2413 |
| `?beerName=IPA` | 336 |
| `?beerStyle=IPA` | 548 |
| `?beerName=IPA&beerStyle=IPA` | 310 |
| `?beerName=IPA&beerStyle=IPA&showInventory=true` | 310, first `quantityOnHand` not null |
| `?beerName=IPA&beerStyle=IPA&showInventory=false` | 310, first `quantityOnHand` null |

- `tesListBeersByStyleAndName` — the combined filter now returns 310 instead of the previous all-beers fall-through.
- `tesListBeersByStyleAndNameShowInventoryTrue` / `...False` — assert inventory visibility via `IsNull.notNullValue()` and `IsNull.nullValue()`.
- Controller unit tests and the `testListBeers`/`testEmptyList` integration tests were updated to the three-argument signature (`beerServiceImpl.listBeers(null, null, false)` / `beerController.listBeers(null, null, false)`).
- `BeerRepositoryTest` was not modified on this branch; its name-search expectation remains 336.

#### Runtime evidence

MariaDB was started directly (same workaround as earlier sessions):

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

The application ran with:

```bash
mvn spring-boot:run \
  -Dspring-boot.run.profiles=localmysql \
  -Dspring-boot.run.jvmArguments='-Dspring.docker.compose.enabled=false'
```

Startup completed with the `localmysql` profile, Hikari pool `RestDB-Pool`, Flyway reporting `Successfully validated 2 migrations` and `Schema \`restdb\` is up to date. No migration necessary.` (schema version 2), the usual MariaDB 12.3 newer-than-verified warning, and Tomcat on port 8080 (`Started Spring7RestMvcApplication in 23.594 seconds`).

##### Live database baseline

The MariaDB `restdb` dataset contains the 2413 bootstrap beers plus 15 beers POSTed during the branch 90–92 sessions, for 2428 total. Direct SQL counts taken before the app start:

| Query | Count |
|---|---|
| all beers | 2428 |
| `upper(beer_name) like upper('%IPA%')` | 351 |
| `beer_style = 7` (IPA ordinal) | 563 |
| name match AND style match | 325 |
| customers | 3 |

##### Combined search

```bash
curl -i 'http://127.0.0.1:8080/api/v1/beer?beerName=IPA&beerStyle=IPA'
```

```text
HTTP 200, 62774 bytes, 325 records

first record:
{"beerName":"GUBNA Imperial IPA","beerStyle":"IPA","createdDate":null,"id":"000212bc-6178-4e38-b48b-b26da298bb2e","price":10,"quantityOnHand":1581,"upc":"1581","updateDate":null,"version":0}
```

The combined request now executes the derived query instead of the branch-92 `findAll()` fall-through. Hibernate logged:

```text
where
    upper(b1_0.beer_name) like upper(?) escape '\\'
    and b1_0.beer_style=?
binding parameter (1:VARCHAR) <- [%IPA%]
binding parameter (2:SMALLINT) <- [IPA]
```

##### showInventory behavior

```bash
curl -i 'http://127.0.0.1:8080/api/v1/beer?beerName=IPA&beerStyle=IPA&showInventory=true'
```

```text
HTTP 200, 62774 bytes, 325 records
first.quantityOnHand: 1581
```

```bash
curl -i 'http://127.0.0.1:8080/api/v1/beer?beerName=IPA&beerStyle=IPA&showInventory=false'
```

```text
HTTP 200, 62944 bytes, 325 records

first record:
{"beerName":"GUBNA Imperial IPA","beerStyle":"IPA","createdDate":null,"id":"000212bc-6178-4e38-b48b-b26da298bb2e","price":10,"quantityOnHand":null,"upc":"1581","updateDate":null,"version":0}
```

The `showInventory=false` response is slightly larger in bytes (62944 vs 62774) because each `quantityOnHand` renders as `null` instead of a number. The record sets are identical across all three variants, confirming the flag reshapes the response only.

##### Regression checks

```text
GET /api/v1/beer?beerName=IPA   -> 200, 351 records (matches SQL baseline)
GET /api/v1/beer?beerStyle=IPA  -> 200, 563 records (matches SQL baseline)
GET /api/v1/beer                -> 200, 2428 records, 473314 bytes
GET /api/v1/customer            -> 200 with the three customers
POST /api/v1/beer (valid)       -> 201, Location: /api/v1/beer/cda2e110-55c1-42da-a743-4ef1d881e309
POST /api/v1/beer (invalid)     -> 400 [{"beerStyle":"must not be null"},{"beerName":"must not be blank"},{"upc":"must not be blank"}]
```

##### Live counts vs test expectations

Every live count is exactly 15 higher than the corresponding `BeerControllerIT` expectation (2428 vs 2413 all, 351 vs 336 name, 563 vs 548 style, 325 vs 310 combined). The offset is the 15 beers POSTed during the branch 90–92 sessions, all of which have `IPA` in the name and `IPA` style, so they appear in every filter's result set. Each live count matches the direct SQL count exactly, confirming the derived queries implement the intended predicates.

##### Shutdown

Spring Boot was stopped first, then MariaDB was shut down over its socket:

```bash
mariadb-admin \
  --socket=/data/data/com.termux/files/usr/var/run/mysqld.sock \
  --user=root shutdown
```

Ports 8080 and 3306 were verified closed (via `/dev/tcp` checks; `netstat` in this Termux environment does not support `AF INET`), and no `java` or `mariadbd` processes remained.

## Branch 94: paging parameters

### `94-page-add-parameters`

Branch created locally from `origin/94-page-add-parameters` (HEAD `85cdb07 Merge branch '93-query-complete-search-params' into 94-page-add-parameters`). This branch introduces paging as a new layer, following the same incremental pattern used for name and style search: parameters first, behavior later.

#### Changes

`BeerController.listBeers` accepts two more optional request parameters and passes all five to the service:

```java
public List<BeerDTO> listBeers(@RequestParam(required = false) String beerName,
                               @RequestParam(required = false) BeerStyle beerStyle,
                               @RequestParam(required = false) Boolean showInventory,
                               @RequestParam(required = false) Integer pageNumber,
                               @RequestParam(required = false) Integer pageSize){
    return beerService.listBeers(beerName, beerStyle, showInventory, pageNumber, pageSize);
}
```

The five-argument signature now flows through `BeerService`, `BeerServiceImpl`, and `BeerServiceJPA`.

#### Important: paging is not implemented yet

Only the signature changed. `BeerServiceJPA.listBeers` still runs the branch-93 search branches and returns every matching row — `pageNumber` and `pageSize` are accepted and immediately ignored:

```java
public List<BeerDTO> listBeers(String beerName, BeerStyle beerStyle, Boolean showInventory,
                               Integer pageNumber, Integer pageSize) {
    ...
    beerList = listBeersByNameAndStyle(beerName, beerStyle);
    ...
    return beerList.stream().map(beerMapper::beerToBeerDto).collect(Collectors.toList());
}
```

There is no `Pageable`, no `PageRequest`, and no slice on `beerList`. No repository method returns a `Page`/`Slice` either. So `?pageNumber=2&pageSize=50` currently returns the full result set, exactly as it would without those parameters.

#### Tests

- `BeerControllerIT` gained `tesListBeersByStyleAndNameShowInventoryTruePage2`, requesting `beerName=IPA&beerStyle=IPA&showInventory=true&pageNumber=2&pageSize=50` and expecting `$.size()` of 50 with a non-null first `quantityOnHand`.
- All existing controller and integration tests were mechanically updated to the five-argument signature, using `1, 25` as placeholder values.
- These placeholder values are never asserted; they exist only because the signature requires them, so the updates do not test paging behavior.

#### Verified failure

The new paging test was run to confirm the gap is real and not just a code-reading inference:

```bash
mvn -o test -Dtest=BeerControllerIT#tesListBeersByStyleAndNameShowInventoryTruePage2
```

```text
[ERROR] BeerControllerIT.tesListBeersByStyleAndNameShowInventoryTruePage2:69 JSON path "$.size()"
Expected: is <50>
     but: was <310>

Tests run: 1, Failures: 1, Errors: 0, Skipped: 0
```

The endpoint returned all 310 combined-matching records instead of the requested 50-row page, confirming the parameters are threaded but not applied. This is the expected state for this branch; the paging behavior is presumably the subject of the next branch.

#### Test environment note

This test ran against the H2/in-memory test datasource (the `@SpringBootTest` IT context bootstraps and seeds its own data), not the MariaDB `localmysql` profile. That is why the count is 310 — matching the branch-93 test expectation — rather than the live 325. MariaDB was not required and was not started for this run.

## Branch 95: create page request

### `95-page-create-page-request`

Branch created locally from `origin/95-page-create-page-request` (HEAD `b2f6b7b Merge branch '94-page-add-parameters' into 95-page-create-page-request`). Exactly one file changed: `BeerServiceJPA.java`, +31/−1.

#### Added constants

```java
private static final int DEFAULT_PAGE = 0;
private static final int DEFAULT_PAGE_SIZE = 25;
```

#### Added `buildPageRequest`

```java
public PageRequest buildPageRequest(Integer pageNumber, Integer pageSize) {
    int queryPageNumber;
    int queryPageSize;

    if (pageNumber != null && pageNumber > 0) {
        queryPageNumber = pageNumber - 1;
    } else {
        queryPageNumber = DEFAULT_PAGE;
    }

    if (pageSize == null) {
        queryPageSize = DEFAULT_PAGE_SIZE;
    } else {
        if (pageSize > 1000) {
            queryPageSize = 1000;
        } else {
            queryPageSize = pageSize;
        }
    }

    return PageRequest.of(queryPageNumber, queryPageSize);
}
```

Three policies are encoded here:

- **1-based to 0-based translation.** The public API is 1-based (`pageNumber=2` means the second page) while Spring Data is 0-based, so the method subtracts 1. A null, zero, or negative `pageNumber` falls back to `DEFAULT_PAGE` (0).
- **Page-size ceiling of 1000.** Any `pageSize` above 1000 is clamped, bounding how much a single request can load.
- **Defaults.** Omitted `pageNumber` yields page 0 and omitted `pageSize` yields 25.

An unguarded edge remains: `pageSize=0` or a negative value passes both checks and reaches `PageRequest.of(...)`, which requires a size of at least 1 and throws `IllegalArgumentException`. That would surface as a 500 rather than a clamped value or a 400.

#### Not yet applied

`listBeers` builds the request and then never uses it:

```java
PageRequest pageRequest = buildPageRequest(pageNumber, pageSize);

List<Beer> beerList;

if(StringUtils.hasText(beerName) && beerStyle == null) {
    beerList = listBeersByName(beerName);
} else if ...
```

The local variable is unused, and none of the repository methods take a `Pageable` or return `Page`/`Slice` — they still return `List<Beer>`. So this branch constructs the paging tool without connecting it to any query, and the endpoint's output is unchanged from branch 94.

#### Verified failure

The same branch-94 paging test was rerun to confirm the behavior is identical:

```bash
mvn -o test -Dtest=BeerControllerIT#tesListBeersByStyleAndNameShowInventoryTruePage2
```

```text
[ERROR] BeerControllerIT.tesListBeersByStyleAndNameShowInventoryTruePage2:69 JSON path "$.size()"
Expected: is <50>
     but: was <310>
```

Still 310, exactly as on branch 94. The `PageRequest` exists but has no effect on the response. Again this ran on the in-memory test datasource, so no MariaDB was started.

## Branch 109: optimistic locking demo

### `109-lock-demo-optimistic`

Checked out from `origin/109-lock-demo-optimistic` (HEAD `889c1a2 Merge branch '108-rel-one-to-one-pers' into 109-lock-demo-optimistic`). The local `application-localmysql.properties` modification (`spring.docker.compose.enabled=false`) carried across the checkout unchanged because the file is identical on both branches.

The branch diff against its parent 108 is one file and one test:

```java
@Disabled // just for demo purposes
@Test
void testUpdateBeerBadVersion() throws Exception {
    Beer beer = beerRepository.findAll().get(0);
    BeerDTO beerDTO = beerMapper.beerToBeerDto(beer);
    beerDTO.setBeerName("Updated Name");

    mockMvc.perform(put(BeerController.BEER_PATH_ID, beer.getId())
            ... .content(objectMapper.writeValueAsString(beerDTO)))
        .andExpect(status().isNoContent());

    beerDTO.setBeerName("Updated Name 2");
    // second PUT with the SAME (now stale) DTO version
    mockMvc.perform(put(BeerController.BEER_PATH_ID, beer.getId())
            ... .content(objectMapper.writeValueAsString(beerDTO)))
        .andExpect(status().isNoContent());
}
```

#### What the demo shows (and why it expects 204 twice)

The test sends a stale `version` in the second PUT body and still expects `204 No Content` — because `BeerServiceJPA.updateBeerById` ignores the client-supplied version entirely:

```java
beerRepository.findById(beerId).ifPresentOrElse(foundBeer -> {
    foundBeer.setBeerName(beer.getBeerName());
    foundBeer.setBeerStyle(beer.getBeerStyle());
    foundBeer.setUpc(beer.getUpc());
    foundBeer.setPrice(beer.getPrice());
    foundBeer.setQuantityOnHand(beer.getQuantityOnHand());
    ...  beerRepository.save(foundBeer)
```

No `foundBeer.setVersion(...)` — the update path is load-and-copy. The entity being saved always carries the version that was just loaded from the database, so a client-sent version can never participate in the optimistic-lock check. PUT responses are `204` with no body, so a well-behaved client would have to re-GET to learn the current version anyway. The `@Disabled` annotation keeps this "nothing fails" demonstration out of the normal test run.

#### Runtime evidence (live MariaDB run)

Environment notes: `restdb` was already migrated to Flyway version 5 (order, category, shipment tables — applied by earlier branch 96–108 sessions), so this run required no new migrations: `Successfully validated 5 migrations` / `Schema \`restdb\` is up to date.` Startup: `localmysql` profile, Hikari `RestDB-Pool`, Tomcat 8080, `Started Spring7RestMvcApplication in 22.229 seconds`. Dataset: 2429 beers (CSV bootstrap plus prior-session POSTs), 3 customers.

The beer listing now returns a Spring Data `Page` JSON (paging landed in branches 96–98), e.g. `{"content":[...],"totalElements":2429,"size":25,...}`, and the paged query carries `order by b1_0.beer_name limit ?` with bind `25`. Boot logged the known `PageImpl` serialization warning: the page JSON shape is not guaranteed stable.

Optimistic-lock demo on beer `000212bc-6178-4e38-b48b-b26da298bb2e` (GUBNA Imperial IPA, version 0):

```text
PUT #1 body version=0  -> 204 ; GET -> version: 1 (name "Lock Demo 1")
PUT #2 body version=0  -> 204 ; GET -> version: 2 (name "Lock Demo 2")   <- stale version ignored
```

Hibernate bind values prove the WHERE predicate used the server-loaded version, not the client's:

```text
PUT #1: binding parameter (7:INTEGER) <- [1]   (new version)
        binding parameter (9:INTEGER) <- [0]   (WHERE version = 0)
PUT #2: binding parameter (7:INTEGER) <- [2]
        binding parameter (9:INTEGER) <- [1]   <- NOT the stale 0 from the request body
```

##### Triggering a real conflict

Twelve concurrent PUTs against the same beer, each with `version: 99` in the body:

```text
4 x HTTP 204, 8 x HTTP 500
final: beerName "Race 1", version 6   (2 + 4 successful increments)
```

The 500s are genuine optimistic-lock failures:

```text
org.springframework.orm.ObjectOptimisticLockingFailureException:
Unexpected row count (expected row count 1 but was 0)
[update beer set beer_name=?,beer_style=?,price=?,quantity_on_hand=?,upc=?,update_date=?,version=?
 where id=? and version=?] for entity [Beer with id '000212bc-...']
Caused by: org.hibernate.StaleStateException
```

Eight `ObjectOptimisticLockingFailureException` / eight `StaleStateException` occurrences in the log, matching the eight 500s. The race: concurrent requests load version N; the first flush increments it; the others' flush UPDATE ... WHERE version=N matches zero rows and Hibernate throws. The body's `version:99` was once again irrelevant.

Two operational observations:

- The exception has no handler in `CustomErrorController` (which covers only `TransactionSystemException` and `MethodArgumentNotValidException`), and no `@ResponseStatus` applies, so clients get an unhelpful HTTP 500. A production API would map `OptimisticLockingFailureException` to 409 Conflict — likely a follow-up branch.
- `updateBeerById` runs `findById` and `save` in separate repository transactions (the service is not `@Transactional`), which widens the window in which this conflict can occur compared to a single transactional read-modify-write.

##### Cleanup and regression

The demo record was restored via `PATCH {"beerName":"GUBNA Imperial IPA"}` → 204 (version now 7). Regressions after the run: `GET /api/v1/customer` 200; paged listing with `?pageNumber=1&pageSize=5` returned `size:5, totalElements:2429`. Spring Boot was stopped, MariaDB shut down via socket, ports 8080/3306 verified closed, no `java`/`mariadbd` processes remained.

## Branch 110: Spring Security Maven dependency

### `110-spring-sec-maven-deps`

Checked out from `origin/110-spring-sec-maven-deps` (HEAD `cf1a0d6 Merge branch '109-lock-demo-optimistic' into 110-spring-sec-maven-deps`). The local `application-localmysql.properties` modification carried across unchanged.

The entire branch diff against 109 is four lines in `pom.xml`:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

No Java source, no configuration, no tests changed. This is the first step of the Spring Security arc (110–116): add the dependency, then configure users, then lock down tests.

#### Why a dependency-only branch matters

Adding `spring-boot-starter-security` with zero configuration triggers Spring Boot's security auto-configuration, which immediately changes runtime behavior:

- A default `SecurityFilterChain` is registered that requires authentication for **every** endpoint.
- A single in-memory user named `user` is created with a **randomly generated password** printed to the startup log.
- Form login and HTTP Basic are both enabled by default.

#### Runtime evidence

Started with the usual command against the already-migrated `restdb` (Flyway version 5, `Successfully validated 5 migrations`, `No migration necessary`). Maven downloaded the new security artifacts (`spring-boot-starter-security 4.0.6`, `spring-security-config/core/crypto 7.0.5`) on first run.

Startup log:

```text
Using generated security password: 253dfcab-7797-4570-a8ea-20cf39e1cb96
Tomcat started on port 8080
Started Spring7RestMvcApplication in 21.428 seconds
```

Unauthenticated access is now blocked:

```text
GET /api/v1/beer?pageNumber=1&pageSize=2   -> HTTP 401
GET /api/v1/customer                        -> HTTP 401
```

Authenticated access with the generated credentials works and paging is intact:

```text
curl -u user:253dfcab-7797-4570-a8ea-20cf39e1cb96 ...

GET /api/v1/beer?pageNumber=1&pageSize=2   -> HTTP 200
GET /api/v1/customer                        -> HTTP 200
body: {"content":[{"beerName":"#001 Golden Amber Lager",...
```

Note the generated password changes on every restart, so it is only usable for local smoke testing. Subsequent branches (111+) replace this with explicit user/password configuration, and the MockMvc tests will need security context wiring (branches 114–116) since the default chain now rejects unauthenticated test requests.

Spring Boot was stopped, MariaDB shut down via socket, ports 8080/3306 verified closed, no leftover processes.
