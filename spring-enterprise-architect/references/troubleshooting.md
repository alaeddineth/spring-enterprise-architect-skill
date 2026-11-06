# Known-failure catalogue (Spring Boot + Oracle + Angular)

Real production/UAT issues and their root causes. Check here before guessing.

## 1. JDBC connection leak / pool exhausted (HikariCP)
**Symptom:** `Connection is not available, request timed out after 30000ms`; active connections climb and never drop.
**Root cause:** `NamedParameterJdbcTemplate` used inside `@Transactional` managed by `JpaTransactionManager` that doesn't bind the same `DataSource` the way you assume, or code that calls `dataSource.getConnection()` manually without closing.
**Fix:** declare a dedicated `DataSourceTransactionManager` for JDBC work and reference it explicitly (`@Transactional(transactionManager = "jdbcTransactionManager")`); never open connections manually; enable `leak-detection-threshold` to get the stack trace of the leaker.

## 2. Native query returns empty although SQL works in SQL Developer
**Checklist:** wrong schema (missing synonym) · parameter type mismatch (String vs NUMBER, DATE vs TIMESTAMP with time part) · `TRUNC` missing on dates · null parameter not handled · filter on a regional/user scope that the working query applies and the broken one doesn't. Diff against a working sibling repository line by line.

## 3. Oracle "invalid column type" / null binding
**Cause:** binding `null` without a SQL type. **Fix:** `new MapSqlParameterSource().addValue("p", value, Types.VARCHAR)`.

## 4. JasperReports "Unknown property pCode"
**Cause:** Lombok generates `getPCode()`; JavaBeans introspection for a field `pCode` expects `getpCode()`. **Fix:** explicit getters, or rename fields (`productCode`).

## 5. `anonymousUser` in a secured endpoint
**Symptoms:** `SecurityContextHolder` empty; "invalid user: anonymousUser".
**Causes found in the wild:**
- JWT filter not registered in the *active* `SecurityFilterChain` bean (`addFilterBefore` missing).
- Filter annotated `@Component` **and** added via `addFilterBefore` → runs twice (Spring Boot auto-registers servlet filters). Fix: remove `@Component`, or add a `FilterRegistrationBean` with `setEnabled(false)`.
- Validation failing silently: log the exact reason (expired, signature, role mismatch). Never compare role `Set`s strictly against a freshly fetched list — trust signed claims, or refresh deliberately.

## 6. PostgreSQL syntax on Oracle
`LIMIT`, `ILIKE`, `::date`, `TRUE/FALSE` literals, `NOW()` → use `FETCH FIRST`, `UPPER() LIKE`, `CAST`/`TO_DATE`, `1/0`, `SYSDATE`/`SYSTIMESTAMP`.

## 7. N+1 queries on search screens
**Fix:** summary projection (`interface` or record DTO), `@EntityGraph` for drill-down only, `hibernate.default_batch_fetch_size=50`.

## 8. Angular screen shows nothing although API returns 200
Usually the DTO shape assumed in the front differs from the real JSON (nested hierarchy vs flat list). Log the raw response once, generate the TypeScript interface from it (or from OpenAPI), then build the template.

## Debug method (always)
1. Reproduce with the smallest input. 2. Read the full stack trace (count how many times your filter/service appears). 3. Turn on SQL + bind logging (`logging.level.org.springframework.jdbc.core=TRACE`). 4. Compare with a working sibling. 5. Fix root cause + add a log or test.
