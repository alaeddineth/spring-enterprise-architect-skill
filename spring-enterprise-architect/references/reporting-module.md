# Reporting module pattern (Spring Boot + JasperReports + Angular)

**One report = one vertical slice.** Predictable, reviewable, easy to hand to a junior.

```
report/
  condbydate/
    CondByDateController.java   // GET /api/reports/cond-by-date?format=pdf|xlsx
    CondByDateService.java      // params validation, fill, export
    CondByDateRepository.java   // NamedParameterJdbcTemplate native SQL
    CondByDateRow.java          // DTO (explicit getters!)
resources/reports/cond_by_date.jrxml
frontend: features/reports/cond-by-date/ (component + service)
```

## Controller

```java
@RestController
@RequestMapping("/api/reports/cond-by-date")
@RequiredArgsConstructor
public class CondByDateController {
    private final CondByDateService service;

    @GetMapping
    public ResponseEntity<byte[]> export(@Valid CondByDateParams params,
                                         @RequestParam(defaultValue = "pdf") String format) {
        ReportFile file = service.generate(params, ReportFormat.from(format));
        return ResponseEntity.ok()
            .contentType(file.mediaType())
            .header(HttpHeaders.CONTENT_DISPOSITION,
                    ContentDisposition.attachment().filename(file.name()).build().toString())
            .body(file.content());
    }
}
```

## Service

```java
@Service
@RequiredArgsConstructor
public class CondByDateService {
    private final CondByDateRepository repository;
    private final JasperTemplateCache templates; // compiles .jrxml once at startup

    @Transactional(readOnly = true, transactionManager = "jdbcTransactionManager")
    public ReportFile generate(CondByDateParams p, ReportFormat format) {
        List<CondByDateRow> rows = repository.find(p);
        JasperPrint print = JasperFillManager.fillReport(
            templates.get("cond_by_date"),
            Map.of("P_START", p.start(), "P_END", p.end()),
            new JRBeanCollectionDataSource(rows));
        return ReportExporter.export(print, format, "conditions_" + p.start());
    }
}
```

## Rules

- **Compile JRXML once** (cache `JasperReport`), never per request.
- Use a `DataSourceTransactionManager` for `NamedParameterJdbcTemplate` work if JPA is also present (see troubleshooting #1).
- DTOs consumed by Jasper: **explicit getters** for fields like `pCode`/`fCode` (Lombok generates `getPCode()`, Jasper's introspection expects `getpCode()`).
- Conditional styling in JRXML must use the **real DB values** — verify them with a `SELECT DISTINCT` before writing expressions.
- Large exports: stream XLSX, cap PDF row count, or move to async job + download link.
- Angular side: call with `responseType: 'blob'`, then `URL.createObjectURL` → download; show a spinner and disable the button during generation.
