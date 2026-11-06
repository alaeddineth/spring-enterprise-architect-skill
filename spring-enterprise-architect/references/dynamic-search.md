# Dynamic multi-criteria search (JPA Specifications + PrimeNG lazy table)

Replaces the classic "15-field Oracle Forms query screen".

## Backend: composable specifications

```java
public final class ConditionSpecs {
    private ConditionSpecs() {}

    public static Specification<Condition> hasStatus(String status) {
        return (root, q, cb) -> status == null ? null : cb.equal(root.get("status"), status);
    }

    public static Specification<Condition> createdBetween(LocalDate from, LocalDate to) {
        return (root, q, cb) -> {
            if (from == null && to == null) return null;
            Path<LocalDate> d = root.get("createdAt");
            if (from == null) return cb.lessThanOrEqualTo(d, to);
            if (to == null)   return cb.greaterThanOrEqualTo(d, from);
            return cb.between(d, from, to);
        };
    }

    public static Specification<Condition> inProductFamily(String familyCode) {
        return (root, q, cb) -> {
            if (familyCode == null) return null;
            Join<Condition, Product> p = root.join("product", JoinType.INNER);
            return cb.equal(p.get("family").get("code"), familyCode);
        };
    }
}
```

```java
@Transactional(readOnly = true)
public Page<ConditionSummary> search(ConditionSearchRequest r, Pageable pageable) {
    Specification<Condition> spec = Specification.allOf(
        ConditionSpecs.hasStatus(r.status()),
        ConditionSpecs.createdBetween(r.from(), r.to()),
        ConditionSpecs.inProductFamily(r.familyCode()));
    return repository.findAll(spec, pageable).map(mapper::toSummary);
}
```

- Endpoint: `POST /api/conditions/search` (body = criteria, query = page/size/sort). POST avoids URL length limits.
- Return a **summary DTO** for the grid; load details with `GET /api/conditions/{id}/details` on drill-down.
- Watch for duplicates with joins: `q.distinct(true)` and a count query that matches.
- Index the columns that users actually filter on (check with business which 3–4 criteria are used 90% of the time).

## Frontend: Angular + PrimeNG

- Reactive form, `debounceTime(300)` only for free-text fields; explicit "Search" button for the rest.
- Cascading dropdowns (Family → Sub-family → Product group): `valueChanges` + `switchMap` to lookup endpoints, reset children on parent change.
- `<p-table [lazy]="true" (onLazyLoad)="load($event)" [totalRecords]="total">`.
- Persist criteria in `sessionStorage` so "back from details" restores the search.
- Lock/disable fields that conflict (e.g. account search disables client fields) with `control.disable({emitEvent:false})`.
