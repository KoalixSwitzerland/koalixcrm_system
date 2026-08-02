# 0023 — Pagination Requires a Total Ordering

- **Status:** Accepted
- **Date:** 2026-08-02

## Context

DRF's page-number pagination is `LIMIT`/`OFFSET` underneath. Each page is a separate query; the
database is free to return rows that compare equal under the `ORDER BY` clause in any order it
likes, and that order need not be the same from one query to the next.

If the ordering clause is not a **total order** — that is, if two distinct rows can compare equal
under it — then paging is not safe. A row can be returned on two consecutive pages, and another
row can be returned on none. A client that walks every page and concatenates the results then
counts one row twice and never sees the other. Nothing errors; the totals are simply wrong, and
they are wrong in a way that looks entirely plausible on screen.

The failure is easy to write and hard to see. `ordering = ["-created_at"]` looks complete and
behaves correctly in every test with fewer rows than a page, or with no two rows sharing a
timestamp. It starts misbehaving in production, on exactly the tables large enough to paginate and
busy enough to produce ties — and the symptom surfaces far from the cause, as a wrong number in a
summary.

Two properties of this codebase shape the decision:

- **KoalixCRM does not paginate today.** `REST_FRAMEWORK` sets no `DEFAULT_PAGINATION_CLASS` and
  no `PAGE_SIZE`, and no ViewSet sets `pagination_class`. List endpoints return every row. There is
  therefore no live defect — but also no protection the day pagination is switched on, which is a
  question of when rather than whether for the larger tables.
- **No ViewSet declares `ordering` at all.** Every list falls back to the model's `Meta.ordering`
  or to an unordered queryset. Several of those `Meta.ordering` declarations are non-unique
  (`["-created_at"]`, `["name"]`, `["account_number"]`, `["position_number"]`), so the property
  would be violated by default rather than by mistake.

The obvious remedy — "require every ViewSet to append the primary key to its `ordering`" — is a
rule a reviewer has to notice and an author has to remember. Nothing fails visibly when it is
forgotten, which makes it precisely the kind of rule that erodes. Anchoring it in
`BaseModelViewSet` is better but still assumes every paginated view inherits from that class, an
assumption that holds here today and quietly stops holding the first time someone writes a
`ListAPIView` or a bare `viewsets.ModelViewSet`.

## Decision

The primary key is appended as the final tiebreaker to the effective ordering of every paginated
queryset, and this is enforced **in the pagination class**, not in a ViewSet base class, not in a
filter backend, and not by convention.

`koalixcrm.shared.pagination.TotalOrderingPageNumberPagination` overrides `paginate_queryset()`.
It reads the effective ordering clause from `queryset.query.order_by`, falling back to
`model._meta.ordering` when the query carries none of its own, and re-orders with `pk` appended
unless the primary key is already part of that clause. Ordering semantics are otherwise untouched:
the existing sort keys keep their positions and directions, and only the tie-breaking behaviour
changes.

The pagination class is the enforcement point because it is the one component every paginated list
response passes through, whatever the view inherits from, whichever filter backends it configures,
and whether or not it declares an `ordering` attribute. An author cannot opt out of it by
accident — only by deliberately setting a different `pagination_class`.

**Pagination remains disabled for now.** This ADR installs the guarantee without changing any
response shape: no `DEFAULT_PAGINATION_CLASS` is registered, so list endpoints continue to return
bare arrays. The class binds the moment an endpoint sets `pagination_class` or the project-wide
default is set. Enabling pagination is a separate decision with its own consequences — it changes
every list response from a JSON array to a `{count, next, previous, results}` envelope, which is a
breaking change for API consumers outside this project — and is deliberately not taken here.

Any future custom pagination class in this codebase inherits from
`TotalOrderingPageNumberPagination` rather than from DRF's `PageNumberPagination` directly, so that
a class written for an unrelated reason (a different page size, a different query parameter) cannot
silently drop the guarantee.

## Notes

- **Client-side deduplication remains necessary and is not made redundant by this ADR.** A total
  ordering removes tie nondeterminism; it does not remove offset shift. If a row that sorts before
  the page boundary is inserted while a client is midway through walking pages, every later row
  shifts by one and the next `OFFSET` re-serves the previous page's last row. This is inherent to
  offset pagination and is not fixable server-side short of cursor pagination.
  `BaseAPIClient._get_object_list` therefore deduplicates by `id` when concatenating pages, and
  any other page-walking client must do the same.
- **Deduplication only covers half of it.** It removes the duplicate; nothing reconstructs a row
  that was shifted past the boundary by a delete and so appeared on no page at all. That is silent
  data loss, and for a caller that aggregates the list it produces a confidently wrong total.
  `BaseAPIClient._get_object_list` therefore also compares each page's `count` against the first
  page's: a difference proves the set changed mid-walk. The check is free — the server runs that
  COUNT per page regardless. On a mismatch the walk restarts once after a short randomised pause,
  and if it mismatches again it raises `ListWalkIncompleteError` rather than returning a short list.
  One retry, because this is contention rather than a transient fault: either writes are rare and
  the retry succeeds, or they are continuous and no retry count converges, while each attempt costs
  a full walk.
- `count` stability is **necessary but not sufficient** — one delete plus one insert between two
  pages leaves it unchanged while rows still shift. The check catches the common cases, not all of
  them. It is mitigation; cursor pagination is the fix.
- The guarantee applies to paginated responses only. Unpaginated consumers are unaffected, which is
  the correct scope: cross-page overlap is a property of pagination, not of ordering in general.
- Appending a sort key can in principle change a query plan. In practice the appended key is the
  primary key, which is indexed, and it is appended last, so it only orders rows that the preceding
  keys already left tied.
- Care is needed with querysets narrowed by `.values()` and grouped by an aggregate: adding a field
  to `ORDER BY` can widen a `GROUP BY` and change the result set rather than merely its order. No
  such paginated queryset exists in this codebase today; if one is introduced, it must be checked
  explicitly.

## Consequences

- A ViewSet author does not need to know this rule for their endpoint to be correct under
  pagination. That is the point: the property is a structural default rather than a review
  obligation.
- Existing non-unique `Meta.ordering` declarations can stay as they are. They express intended sort
  order; totality is supplied underneath them.
- A reusable `OrderingTotalityTestCaseMixin` provides the regression test — full page walk with no
  duplicates or gaps across a tie group, stable order across repeated identical requests, and the
  response envelope shape — so that verifying the property costs roughly one line of fixture setup
  per ViewSet instead of a hand-written test each time.
- ~~Turning pagination on project-wide remains an open decision, to be taken deliberately and with
  an API version note, not as a side effect of this one.~~ **Superseded by the amendment below —
  it was never an open decision.**

---

## Amendment 2026-08-02 — pagination enabled; the "open decision" framing was wrong

**Status:** Accepted

The original text above called enabling pagination project-wide an open decision. A cross-check
against the org standard shows it was not one. Org ADR-0001 §3.1 (`template-backend-dionysos`,
`docs/adr/0001-api-style-crud-rest.md`) states:

> Every list endpoint of the Django backend uses `PageNumberPagination`. The global DRF
> configuration (`DEFAULT_PAGINATION_CLASS`, `PAGE_SIZE`) sets this standard system-wide […]
> **No list endpoint returns an unpaginated response.**

[ADR-0022](0022-backend-architecture-org-binding.md) binds KoalixCRM to org ADRs 0001–0012
"as-written". The state this ADR described as a deliberate pause — no `DEFAULT_PAGINATION_CLASS`,
every list returning a bare array — was therefore a standing non-compliance with a binding this
project had already accepted, not a choice still available to it. Describing it as an open decision
made a gap look like a position.

### What is now configured

`REST_FRAMEWORK` in `projectsettings/settings/base_settings.py`:

- `DEFAULT_PAGINATION_CLASS = 'koalixcrm.shared.pagination.TotalOrderingPageNumberPagination'`
- `PAGE_SIZE = 50` — the org default in §3.1, matched exactly rather than chosen independently.

The envelope is DRF's `count`/`next`/`previous`/`results` with no additional top-level fields, as
§3.1 requires. The total-ordering guarantee installed by the original decision is what makes the
switch safe to make in one step: no page walk can double-serve or skip a tied row.

This *is* the breaking change the original text warned about — every list response changes from a
JSON array to an envelope. `BaseAPIClient._walk_object_pages` already handled both shapes, so no
in-repo consumer needed changing.

### Sibling-product precedent

The Workflow Support Webapp reached the same end state from the same starting point:
`TotalOrderingPageNumberPagination` registered as `DEFAULT_PAGINATION_CLASS`, page size 50,
envelope mandatory (WFS `architecture/backend_org_adr_binding.md`; WFS ADR-1). Two differences are
worth recording rather than copying:

- WFS additionally requires its two pre-existing custom pagination classes to inherit from
  `TotalOrderingPageNumberPagination`. KoalixCRM has no custom pagination classes today; the rule
  in the original decision above already covers the case if one is added.
- WFS ADR-32 moves two fully-walked aggregation endpoints to keyset pagination, which closes the
  offset-shift gap described in the Notes above. Org ADR-0001 §3.1 explicitly *rejects* cursor
  pagination as a global default, so WFS scoped it to two endpoints rather than changing
  `DEFAULT_PAGINATION_CLASS`. KoalixCRM has no equivalent endpoint yet; if one appears, that scoped
  shape is the precedent to follow.

### Compliance gaps this cross-check surfaced (not closed here)

Recording these so they are tracked rather than rediscovered:

1. **§3.2 fallback `ordering` attribute.** The org ADR requires a fixed `ordering` attribute on the
   ViewSet as the fallback when no `?ordering=` is passed. No KoalixCRM ViewSet declares one; every
   list falls back to `Meta.ordering`. Totality is satisfied structurally by the pagination class,
   so this is a literal-compliance gap rather than a correctness defect — but it is a gap, and the
   original decision's claim that "existing non-unique `Meta.ordering` declarations can stay as
   they are" addresses totality only, not §3.2's fallback requirement.
2. **Unpaginated `@action` list responses.** `SerialUnitViewSet.history`, `as_built`,
   `installed_components`, `holder_timeline` and `location_timeline` return bare arrays; DRF does
   not paginate `@action` responses unless the action paginates explicitly. §3.1's exception clause
   permits this only for a "fixed, small aggregation" with a commented justification. `history` is
   an unbounded movement log and does not qualify.
3. **No OpenAPI CI check.** §3.4 requires CI to assert that no list endpoint appears in the
   generated spec without the envelope schema. KoalixCRM has no such check.

### Filtering

Enabling pagination exposed that field filtering had never worked: `BaseModelViewSet.filter_backends`
*replaced* `DEFAULT_FILTER_BACKENDS` rather than extending it, so the configured
`DjangoFilterBackend` was active on no endpoint, and django-filter ignores unrecognised query
parameters rather than rejecting them — an unfiltered list looked like a filtered one.

Org ADR-0001 §3.3 is one sentence ("Filtering is done via query parameters […] validated via DRF /
django-filter") and prescribes no declaration style. KoalixCRM derives the FilterSet from the model
(`koalixcrm.shared.filters.AutoFilterBackend`), extending the reasoning of §3.4's structural-default
enforcement to filtering. **This diverges from WFS practice**, which declares `filterset_fields` or
a `FilterSet` per ViewSet and treats each filter as a documented API contract (see WFS
`architecture/api_contract_reassignment_signature_filters.md`). Both satisfy §3.3. The divergence is
deliberate and is flagged here for the org standard to settle, since a filter, once published in the
schema, is API surface that consumers depend on — the argument for WFS's explicitness — while
per-ViewSet declaration is exactly the review-dependent pattern §3.4 rejects for ordering — the
argument for derivation.
