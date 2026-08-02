# 0024 — Stammdaten-Governance: kanonischer Kern, mandantenspezifische Peripherie

- **Status:** Proposed
- **Date:** 2026-08-02

## Context

[ADR-0004](0004-classification-and-extensible-attributes.md) gives tenant admins an EAV layer they
can extend without a code change or a schema migration. That is the feature. It is also the risk:
a system that lets operators define their own vocabulary will accumulate a bad one unless
something decides where the damage lands.

The failure is not primarily people entering wrong values. A wrong weight on one variant is local,
cheap and self-correcting — someone notices and fixes it. The expensive failures are **structural**:
forty-seven near-identical "colour" attributes, three overlapping `AttributeSet` rows bound to the
same `ClassificationNode`, an `AttributeDefinition` whose `data_type` no longer matches what
operators actually type into it. Those compound silently and are only repairable by a data
migration over live tenant data.

The obvious remedy — validate harder at entry — makes this worse, not better. The person entering
master data rarely bears the cost of entering it badly; the cost surfaces downstream in
procurement, reporting, the webshop, or a customer's ERP. Tightening entry validation pushes the
cost onto the one participant who gets no benefit from paying it, and they route around it
rationally: `"N/A"`, `"999"`, `"tbd"`, a duplicate product because search was slower than create, a
fresh attribute because the existing one had a required field they could not fill. **Strict
validation does not produce good data; it produces data that passes validation** — which is worse
than obviously bad data, because it is invisible to exactly the checks that were supposed to catch
it.

Two further constraints shape the decision:

- KoalixCRM is multi-tenant and sold to organisations that will not all staff a data-steward role.
  A governance design that only works with a dedicated steward will not hold for most workspaces.
- [ADR-0018](0018-canonical-product-attribute-vocabulary.md) already established a canonical
  `koalix.*` vocabulary read through `get_canonical_value()`, with `ProductAttributeMapping` as the
  data-driven adapter seam. The mechanism this ADR needs largely exists; what is missing is the
  *policy* that says what it is for.

## Decision

### 1. Canonical core, tenant periphery

**Business logic reads only the canonical vocabulary. Tenant-defined attributes are leaf data.**

Any code that branches, calculates, gates a workflow, or feeds a downstream contract resolves its
inputs through `canonical_attributes.get_canonical_value()` and the `koalix.*` keys, never through
a tenant's own `AttributeDefinition.key`. Tenant-defined attributes may be displayed, searched,
filtered, exported and imported — they may not be depended upon.

The consequence is the point: a tenant may create a messy vocabulary and it stays local. Nothing
load-bearing breaks, and restructuring it later is an edit rather than a migration event. The
system does not keep tenant data clean — it makes unclean tenant data **inert**.

This inverts the usual quality argument. The question is not "how do we stop operators making a
mess" but "how much of the system is allowed to care". Every consumer that reads a tenant key
directly converts a future rename from a local edit into a coordinated migration, so that coupling
is what governance restricts — not the operator's freedom to define attributes.

Adding a `koalix.*` key stays an ADR-0018 amendment, deliberately: the canonical set is small,
stable, and is the surface everything else is allowed to depend on.

### 2. Defining structure is a distinct privilege from entering values

Schema authorship and data entry are separated, because they carry different blast radii. The
models that define vocabulary are designated **schema models**:

`AttributeDefinition`, `AttributeGroup`, `AttributeSet`, `AttributeSetGroup`,
`AttributeSetDefault`, `AttributeValidationRule`, `Classification`, `ClassificationNode`,
`ProductAttributeMapping`.

Everything else in the products and stock domains is an **instance model**. Write access to schema
models is granted separately from, and far more sparingly than, write access to instance models.
Many people create products; very few define what a product *is*.

The registry lives in code (`koalixcrm.shared.governance`) rather than in prose, and a test asserts
that every products/stock model is classified as exactly one of the two. A newly added model
therefore cannot default into the permissive class by being forgotten.

### 3. Completeness is checked at state transitions, not at entry

A global `is_required` flag on `AttributeDefinition` forces one answer to several different
questions — what must be filled to save a draft, to publish to a webshop, to export to a customer's
ERP are not the same set — and operators resolve that conflict by typing placeholder values.

Required-ness is therefore evaluated against the transition being attempted, with
`Product.lifecycle_status` (`DRAFT` → `ACTIVE`) as the first such gate, per
[UC-0002](../06_runtime_view/use_case_0002.md) Alternativablauf B. Entry stays permissive;
publication is strict. This keeps the strictness at the point where someone has a reason to care
about it, which is the point where it will not be routed around.

## Notes

- **Provenance and reversibility outrank prevention.** Correct data cannot be guaranteed; knowing
  where a value came from and being able to undo a bad restructuring can be. `imported_at` and
  `source_mapping` on `ProductAttributeValueBase` are already this instinct (ADR-0018
  §Quellpriorität). The schema-migration facility that would complete it is not decided here.
- **The role plumbing is currently decorative.** `core.access.permissions_for_role()` and
  `effective_roles()` are consumed by nothing outside their own tests. Real enforcement today is
  Django's per-model permissions via `ModelPermissionsWithListView` (`DjangoModelPermissions`),
  plus `user_workspaces()` for workspace visibility. Decision 2 is therefore expressed in Django
  model permissions, which are wired, rather than by extending the `Role` enum, which is not.
  Anyone reading `permissions_for_role` as an enforcement point is reading it wrong.
- **`AttributeDefinition` deletion is currently destructive.**
  `ProductAttributeValueBase.attribute_definition` is `on_delete=CASCADE`, so deleting one
  definition removes every value recorded against it across the workspace, with no warning and no
  undo — and it is now reachable in two clicks from the Grappelli dashboard. This ADR does not fix
  it; it is recorded here because it is the single largest unguarded blast radius in the area the
  ADR governs.
- **`data_type` has no immutability guard.** Changing it without moving the value rows makes the
  ADR-0004 cascade read a different, empty typed table: the values still exist and become
  invisible. Also not fixed here, also recorded.
- **Governance workflow is deliberately not introduced.** Approval chains for attribute creation
  are the classic cure that is worse than the disease: the reliable outcome is that real product
  data moves to a spreadsheet and the system holds a stale subset. Cheap affordances — showing the
  nearest existing definitions before allowing a new one — are preferred over gates.

## Consequences

- A tenant can produce a poor vocabulary without endangering anything outside their own workspace's
  presentation and export paths. That is an accepted outcome, not a defect.
- Every new consumer of attribute data has to answer one question at review: does it read a
  canonical key or a tenant key? Reading a tenant key is not forbidden, but it is a decision with a
  migration cost attached, and it should be visible as such.
- Products sold to organisations that will not staff a data-steward role need opinionated shipped
  defaults — a strong canonical set and standard-derived `AttributeSet` rows most tenants never
  extend — rather than a flexible EAV they will fill with forty-seven colours. That is a product
  decision this ADR makes visible rather than settles.
- Decision 3 requires per-transition completeness profiles, which do not exist yet; today only the
  `DRAFT` → `ACTIVE` gate is in scope, and even that is currently specified in UC-0002 rather than
  implemented.

## References

- [ADR-0004](0004-classification-and-extensible-attributes.md) — classification and extensible attributes
- [ADR-0018](0018-canonical-product-attribute-vocabulary.md) — canonical `koalix.*` vocabulary and the mapping seam
- [ADR-0021](0021-produkt-variantengranularitaet-topologie-schluesselung-attributkaskade.md) — attribute inheritance cascade
- [UC-0002](../06_runtime_view/use_case_0002.md) — maintaining extensible attributes
