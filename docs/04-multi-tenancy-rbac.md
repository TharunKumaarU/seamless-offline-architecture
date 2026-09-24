# 04 — Multi-Tenancy & RBAC

*One deployment, many operating companies, and permissions that stay explainable.*

## Tenancy model

A field-service group typically runs country/regional operating units — same processes, separate data, occasionally different rules. The pragmatic model is **row-level tenancy in one database**:

- Every business table carries `tenant_id NOT NULL` (indexed, usually as the leading column of composite indexes).
- Users hold **memberships**: a user ↔ tenant mapping with a role per tenant. Engineers usually live in one tenant; managers and admins often span several.
- Every request resolves to an **active tenant context** (from the JWT claims plus a selected-tenant header where multi-tenant users choose). All queries filter by it — by construction, not by convention.

"By construction" means the filter is applied in one chokepoint, not sprinkled through handlers:

```python
# repository layer — the ONLY way handlers get a query
def tenant_query(session, model, ctx):
    q = select(model).where(model.tenant_id == ctx.tenant_id)
    if hasattr(model, "is_deleted"):
        q = q.where(model.is_deleted.is_(False))
    return q
```

Two failure classes to test for explicitly, because each is one forgotten filter away:

- **Cross-tenant read** — list endpoints and *aggregate/dashboard* endpoints (the classic leak: a count query someone wrote directly).
- **Cross-tenant write** — the subtler one: the payload references a foreign key (work order, customer) belonging to another tenant. Validate that *every referenced entity* is in the caller's tenant, not just the new row itself. An approval endpoint that takes `entity_id` from the request body and forgets the tenant check is a textbook IDOR.

## Per-tenant configuration

Tenants differ in mundane, high-leverage ways. Give configuration its own tables instead of `if tenant == X` code:

| Config | Why per-tenant |
|---|---|
| Email/SMTP identity | Each unit mails from its own address/relay |
| Document-number formats | Different prefixes/sequences per unit (doc 05) |
| Feature toggles | Units adopt modules at different speeds |
| Approval workflow settings | Who must approve what varies by unit |

Config is **pull-only master data** to devices (doc 02): cached locally, server-authoritative, never edited in the field.

## RBAC with layered inheritance

Flat role→permission tables die quickly in real organizations: "all engineers can create inspection reports, except in tenant B where only seniors can, except Ali who's in a pilot." Model permissions as **layered overrides**:

```
effective(user, tenant, permission) =
    user-level override in tenant      if set
 else role-level override in tenant    if set
 else global role default              if set
 else deny
```

- Layers are sparse — most cells are unset and fall through. Storage: one `(scope_type, scope_id, tenant_id, permission, allow)` table.
- **Most-specific-wins, and deny-by-default at the bottom.** Both halves matter; the second is what auditors ask about.
- Resolve once per request (or on login for devices) into a flat effective-permission set; cache it. Devices receive the *resolved* set as pull-only data — mobile should never re-implement the inheritance algorithm.

### Separate capability axes

Some flags are not business permissions but **access axes** and deserve first-class columns, resolved per tenant: `web_access`, `mobile_access`, `approver_access`. The predictable bugs when these are conflated with roles: dropdowns listing every user instead of field-capable ones (filter user pickers by capability, not by role name), and approval rights leaking across tenant boundaries when a manager spans tenants (approver capability must be granted *per tenant membership*, not on the user row).

### Read scope ≠ action permission

"Can see the time-entries list" and "sees whose time entries" are different questions. Model read scope explicitly per role: **own / team / tenant**. Every list endpoint applies both the tenant filter *and* the scope filter from the same resolved context. The failure smell: HR-ish lists (attendance, expenses) where any authenticated user can widen the filter by editing query params.

## Fail closed, especially in approval paths

Approval gates read state that might be missing or malformed (a stub row, a null config, an unparseable legacy field). The rule is boring and absolute: **any error or ambiguity in an authorization check resolves to "no"**. A gate that fails open under malformed data is a vulnerability that looks like a convenience — it will approve exactly the records that are already broken.

```python
def can_approve(user_ctx, report) -> bool:
    try:
        cfg = approval_config(report.tenant_id, report.type)
        if cfg is None:
            return False                    # unconfigured ≠ unrestricted
        return evaluate(cfg, user_ctx, report)
    except Exception:
        log.exception("approval check failed — denying")
        return False                        # fail CLOSED
```

## Auditing

One audit interceptor at the API layer: who, what entity, which tenant, before/after for sensitive fields, when. Approvals, permission changes, and deletions are non-negotiable audit events — in field service, month-end disputes are settled by this table.

## Checklist

- [ ] `tenant_id` on every business table, enforced NOT NULL, in composite indexes
- [ ] Single query chokepoint applies tenant filter; handlers cannot bypass it
- [ ] Write paths validate tenant of **referenced** entities, not just the new row
- [ ] Aggregates/dashboards/counts covered by the same scoping tests
- [ ] Permission resolution: user > role > global, deny-by-default, resolved once and cached
- [ ] Devices receive resolved permissions; never re-derive them client-side
- [ ] Capability axes (web/mobile/approver) separate from roles, granted per membership
- [ ] List endpoints enforce read scope (own/team/tenant) server-side
- [ ] All authorization gates fail closed on error, with an alert on the error path
- [ ] Audit log on approvals, permission changes, deletions
