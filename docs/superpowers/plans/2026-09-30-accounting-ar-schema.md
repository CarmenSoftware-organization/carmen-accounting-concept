# AR Schema + Migration (schema-only) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add the AR / customer tables (7 enums, 10 models) to the backend-v2 tenant Prisma schema with a reviewed, deployable migration — no services.

**Architecture:** Schema text is copied verbatim from the spec's Appendix A into `schema.prisma` after the AP block, four back-relations are added to existing models, and the migration SQL is produced by `prisma migrate diff` against a throwaway local database that has every existing migration applied. The new tables join the audit-extension `excludeModels` list so the generic write-interceptor ignores them until AR services exist.

**Tech Stack:** Prisma 7.10 (`prisma-client` generator, `prisma.config.ts` + dotenvx), PostgreSQL 16 local (`samutpra@localhost:5432`, passwordless, `CREATEDB` role), Bun 1.4, Turbo (`check-types`), Vitest (prisma package).

**Spec:** `docs/superpowers/specs/2026-09-30-accounting-ar-schema-design.md` (this repo, `carmen-accounting-concept`). The code lives in `carmen-turborepo-backend-v2` (sibling directory `../carmen-turborepo-backend-v2`).

## Global Constraints

- Work in `carmen-turborepo-backend-v2` on branch `feature/accounting-ar-schema` cut from `origin/main`; never commit to `main` or touch `prod`.
- Only these files may change: `packages/prisma-shared-schema-tenant/prisma/schema.prisma`, `packages/prisma-shared-schema-tenant/prisma/migrations/<ts>_accounting_ar_tables/migration.sql`, `packages/prisma-shared-schema-tenant/src/client.ts`, `apps/micro-business/CLAUDE.md`.
- Do NOT run `prisma migrate dev`, `prisma migrate reset`, or `db:deploy` without an explicit `DATABASE_URL=<scratch>` prefix (spec §5; `apps/micro-business/CLAUDE.md` §Tenant migrations). The package `.env` points at the real `CARMEN_TENANT` schema. Verified 2026-09-30: an explicit `DATABASE_URL=` on the command line wins over `.env` (dotenvx does not overload).
- Scratch database URL is exactly `postgresql://samutpra@localhost:5432/ar_scratch?schema=public` — typed in full, never derived from `.env`. Confirm the target with `SELECT current_database(), current_schema()` before any DDL.
- Prisma 7 flags: `migrate diff --from-config-datasource --to-schema prisma/schema.prisma [--script | --exit-code]` (there is no `--from-url` / `--to-schema-datamodel` in this version).
- Migration folder name: `<YYYYMMDDHHmmss>_accounting_ar_tables` with a UTC timestamp greater than `20260928130000`.
- The migration must be ADD-only: 7 `CREATE TYPE`, 10 `CREATE TABLE`, indexes, unique constraints, foreign keys. Any `ALTER TABLE ... DROP`, `DROP`, or changes to tables outside the ten new ones means the scratch DB or schema is wrong — stop and investigate, do not hand-edit around it.
- No automated tests are written in this plan (user preference). Static checks are mandatory: `prisma validate`, `db:generate`, `check-types`, and the existing prisma-package Vitest suite must pass.
- Commit messages in English, imperative, prefixed `feat(prisma):` / `chore:` / `docs:`.

## Review Focus

Inputs and conditions the spec implies but no step exercises — checked by the reviewer, not by tests (no tests in this plan):

1. **`tb_ar_tax_invoice` rows with both or neither FK set** — the schema cannot express XOR; the reviewer confirms the two `@unique` columns are nullable and the comment says the AR service enforces XOR (spec §4.2 #7).
2. **Migration applied to a schema that already has `enum_customer_address_type` or a `tb_customer` table** (e.g. a BU where someone hand-created one) — `CREATE TYPE`/`CREATE TABLE` will fail; the deploy note in the PR body must tell the operator to check `\dt tb_customer*` per BU before firing the endpoint.
3. **`tb_customer.code` uniqueness** — unique on `(code, deleted_at)` like `tb_ap_invoice.doc_no`, so a soft-deleted customer frees its code; reviewer confirms this matches `tb_vendor` (`vendor_code_name_u` is `(code, name, deleted_at)`) closely enough and that the spec chose `(code, deleted_at)` deliberately.
4. **Decimal precision drift** — every money column must be `Decimal(20,5)` and every rate `Decimal(15,5)`; reviewer greps the appendix block for `Decimal(18` or `Decimal(16` (FRD precision) and expects zero hits.
5. **`excludeModels` string typos** — a misspelled table name silently leaves that table audited; reviewer diffs the ten strings in `client.ts` against `grep '^model tb_ar_\|^model tb_customer' schema.prisma`.

---

### Task 1: Branch and scratch database baseline

**Files:**
- None modified. Creates local database `ar_scratch`.

**Interfaces:**
- Produces: branch `feature/accounting-ar-schema` at `origin/main`; database `ar_scratch` with every migration under `prisma/migrations/` applied and `migrate diff --exit-code` = 0 against the unmodified schema. Later tasks rely on the shell variable `SCRATCH_URL`.

- [ ] **Step 1: Create the branch**

```bash
cd /Users/samutpra/GitHub/carmensoftware-organize/carmen-turborepo-backend-v2
git fetch origin
git switch -c feature/accounting-ar-schema origin/main
git log --oneline -1
```
Expected: branch created; HEAD is the current `origin/main` commit.

- [ ] **Step 2: Create the scratch database and verify the target**

```bash
createdb ar_scratch
export SCRATCH_URL='postgresql://samutpra@localhost:5432/ar_scratch?schema=public'
psql "$SCRATCH_URL" -Atc 'select current_database(), current_schema()'
```
Expected output exactly: `ar_scratch|public`. If it prints anything else, stop.

- [ ] **Step 3: Apply every existing migration to the scratch database**

```bash
cd packages/prisma-shared-schema-tenant
DATABASE_URL="$SCRATCH_URL" bunx prisma migrate deploy
```
Expected: ends with `All migrations have been successfully applied.` and the count of applied migrations equals `ls prisma/migrations | grep -vc migration_lock.toml`.

- [ ] **Step 4: Confirm the unmodified schema matches the scratch database**

```bash
DATABASE_URL="$SCRATCH_URL" bunx prisma migrate diff --from-config-datasource --to-schema prisma/schema.prisma --exit-code; echo "exit=$?"
```
Expected: `exit=0` (empty diff). If `exit=2`, the committed schema and migrations already disagree on `main`; stop and report the diff (`--script`) before touching anything.

- [ ] **Step 5: Nothing to commit** — record `SCRATCH_URL` for the next tasks.

---

### Task 2: Add AR enums, models and back-relations to `schema.prisma`

**Files:**
- Modify: `packages/prisma-shared-schema-tenant/prisma/schema.prisma` — enums after line 344 (`enum enum_ap_payment_method` closes there), models after line 7424 (`model tb_ap_payment_expense` closes there), plus one line each in `tb_tax_profile` (before line 1592), `tb_gl_dimension` (before line 6941), `tb_gl_dimension_value` (before line 6970), `tb_bank_account` (before line 7042). Line numbers are as of `origin/main` `b80024588`; re-locate by the anchor text if they moved.

**Interfaces:**
- Consumes: spec Appendix A (`docs/superpowers/specs/2026-09-30-accounting-ar-schema-design.md`, the single ```prisma fenced block).
- Produces: 7 new enums `enum_ar_invoice_doc_type`, `enum_ar_invoice_status`, `enum_ar_invoice_source`, `enum_ar_tax_invoice_status`, `enum_ar_receipt_status`, `enum_ar_receipt_method`, `enum_customer_address_type`; 10 new models `tb_customer`, `tb_customer_address`, `tb_ar_invoice`, `tb_ar_invoice_detail`, `tb_ar_invoice_detail_dimension`, `tb_ar_invoice_reference`, `tb_ar_tax_invoice`, `tb_ar_receipt`, `tb_ar_receipt_detail`, `tb_ar_receipt_wht`; generated client types under `generated/client`.

- [ ] **Step 1: Extract the appendix block from the spec into two scratch files**

```bash
SPEC=/Users/samutpra/GitHub/carmensoftware-organize/carmen-accounting-concept/docs/superpowers/specs/2026-09-30-accounting-ar-schema-design.md
awk '/^```prisma$/{p=1; next} /^```$/{p=0} p' "$SPEC" > /tmp/ar_block.prisma
awk '/^model /{m=1} !m' /tmp/ar_block.prisma > /tmp/ar_enums.prisma
awk '/^model /{m=1} m' /tmp/ar_block.prisma > /tmp/ar_models.prisma
grep -c '^enum ' /tmp/ar_enums.prisma; grep -c '^model ' /tmp/ar_models.prisma
```
Expected: `7` then `10`. (Using `/tmp` here is fine — these are throwaway inputs, not project files.)

- [ ] **Step 2: Insert the enums after `enum_ap_payment_method`**

```bash
cd /Users/samutpra/GitHub/carmensoftware-organize/carmen-turborepo-backend-v2/packages/prisma-shared-schema-tenant
python3 - <<'PY'
p='prisma/schema.prisma'; s=open(p,encoding='utf-8').read()
anchor='enum enum_ap_payment_method {\n  bank_transfer\n  cheque\n  cash\n  credit_card\n  promptpay\n}\n'
assert s.count(anchor)==1, 'anchor for enum insertion not found exactly once'
block=open('/tmp/ar_enums.prisma',encoding='utf-8').read().strip('\n')+'\n'
s=s.replace(anchor, anchor+'\n'+block)
open(p,'w',encoding='utf-8').write(s); print('enums inserted')
PY
grep -n '^enum enum_ar_\|^enum enum_customer_address_type' prisma/schema.prisma
```
Expected: 7 lines, all between line 345 and ~400.

- [ ] **Step 3: Insert the models after `tb_ap_payment_expense`**

```bash
python3 - <<'PY'
import re
p='prisma/schema.prisma'; s=open(p,encoding='utf-8').read()
m=re.search(r'^model tb_ap_payment_expense \{\n.*?^\}\n', s, re.S|re.M)
assert m, 'tb_ap_payment_expense not found'
block=open('/tmp/ar_models.prisma',encoding='utf-8').read().strip('\n')+'\n'
s=s[:m.end()]+'\n'+block+s[m.end():]
open(p,'w',encoding='utf-8').write(s); print('models inserted')
PY
grep -n '^model tb_customer\|^model tb_ar_' prisma/schema.prisma
```
Expected: 10 lines, all after the `tb_ap_payment_expense` line.

- [ ] **Step 4: Add the four back-relations to existing models**

```bash
python3 - <<'PY'
p='prisma/schema.prisma'; s=open(p,encoding='utf-8').read()
edits=[
 ('  tb_tax_profile_comment              tb_tax_profile_comment[]\n\n  @@unique([name, deleted_at], map: "taxprofile_name_deletedat_u")',
  '  tb_tax_profile_comment              tb_tax_profile_comment[]\n  tb_customer                         tb_customer[]\n\n  @@unique([name, deleted_at], map: "taxprofile_name_deletedat_u")'),
 ('  tb_ap_invoice_detail_dimension tb_ap_invoice_detail_dimension[]\n\n  @@unique([code, deleted_at], map: "gldimension_code_u")',
  '  tb_ap_invoice_detail_dimension tb_ap_invoice_detail_dimension[]\n  tb_ar_invoice_detail_dimension tb_ar_invoice_detail_dimension[]\n\n  @@unique([code, deleted_at], map: "gldimension_code_u")'),
 ('  tb_ap_invoice_detail_dimension tb_ap_invoice_detail_dimension[]\n\n  @@unique([gl_dimension_id, code, deleted_at], map: "gldimensionvalue_code_u")',
  '  tb_ap_invoice_detail_dimension tb_ap_invoice_detail_dimension[]\n  tb_ar_invoice_detail_dimension tb_ar_invoice_detail_dimension[]\n\n  @@unique([gl_dimension_id, code, deleted_at], map: "gldimensionvalue_code_u")'),
 ('  tb_ap_payment        tb_ap_payment[]\n\n  @@unique([code, deleted_at], map: "bankaccount_code_u")',
  '  tb_ap_payment        tb_ap_payment[]\n  tb_ar_receipt        tb_ar_receipt[]\n\n  @@unique([code, deleted_at], map: "bankaccount_code_u")'),
]
for old,new in edits:
    assert s.count(old)==1, old.splitlines()[0]
    s=s.replace(old,new)
open(p,'w',encoding='utf-8').write(s); print('4 back-relations added')
PY
grep -n 'tb_customer\[\]\|tb_ar_invoice_detail_dimension\[\]\|tb_ar_receipt\[\]' prisma/schema.prisma
```
Expected: 4 lines — in `tb_tax_profile`, `tb_gl_dimension`, `tb_gl_dimension_value`, `tb_bank_account` (the `tb_ar_invoice_detail_dimension[]` in `tb_ar_invoice_detail` itself is a 5th hit and is fine).

- [ ] **Step 5: Validate the schema and regenerate the client**

```bash
bunx prisma validate
bun run db:generate
```
Expected: `The schema at prisma/schema.prisma is valid 🚀` and `Generated Prisma Client` with no relation errors. Typical failure: `Error validating field ... in model ...: The relation field ... is missing an opposite relation field` → a back-relation from Step 4 did not land; re-run Step 4's grep.

- [ ] **Step 6: Check precision and naming rules from the spec**

```bash
awk '/^model tb_customer \{/{p=1} p' prisma/schema.prisma | grep -c 'Decimal(18\|Decimal(16'
grep -c '@@map' /tmp/ar_models.prisma
grep -c 'OI-' /tmp/ar_block.prisma
```
Expected: `0`, `0`, `0`.

- [ ] **Step 7: Commit**

```bash
cd /Users/samutpra/GitHub/carmensoftware-organize/carmen-turborepo-backend-v2
git add packages/prisma-shared-schema-tenant/prisma/schema.prisma
git commit -m "feat(prisma): add AR and customer models to tenant schema

7 enums + 10 models mirroring the AP tables (spec: carmen-accounting-concept
docs/superpowers/specs/2026-09-30-accounting-ar-schema-design.md). Schema only;
no services yet."
```

---

### Task 3: Generate and verify the migration

**Files:**
- Create: `packages/prisma-shared-schema-tenant/prisma/migrations/<ts>_accounting_ar_tables/migration.sql`

**Interfaces:**
- Consumes: Task 1 `SCRATCH_URL` (scratch DB at pre-AR state), Task 2 schema.
- Produces: a tracked migration folder whose SQL brings a database from the last migration (`20260928130000_drop_unused_calculation_method_enum`) to the Task 2 schema.

- [ ] **Step 1: Create the migration folder with a UTC timestamp**

```bash
cd /Users/samutpra/GitHub/carmensoftware-organize/carmen-turborepo-backend-v2/packages/prisma-shared-schema-tenant
TS=$(date -u +%Y%m%d%H%M%S); echo "$TS"
[ "$TS" \> "20260928130000" ] && mkdir -p "prisma/migrations/${TS}_accounting_ar_tables" && echo "created prisma/migrations/${TS}_accounting_ar_tables"
```
Expected: folder created; timestamp printed is later than `20260928130000`.

- [ ] **Step 2: Diff the scratch database against the new schema into the migration file**

```bash
DATABASE_URL="$SCRATCH_URL" bunx prisma migrate diff --from-config-datasource --to-schema prisma/schema.prisma --script > "prisma/migrations/${TS}_accounting_ar_tables/migration.sql"
wc -l "prisma/migrations/${TS}_accounting_ar_tables/migration.sql"
```
Expected: a few hundred lines (the AP migration with 15 tables was 644 lines; expect roughly 400–550).

- [ ] **Step 3: Review the SQL is ADD-only and complete**

```bash
M="prisma/migrations/${TS}_accounting_ar_tables/migration.sql"
echo "types: $(grep -c '^CREATE TYPE' $M)  tables: $(grep -c '^CREATE TABLE' $M)"
grep -n 'DROP\|ALTER TABLE "tb_\(tax_profile\|gl_dimension\|gl_dimension_value\|bank_account\)"' $M
grep -o 'CREATE TABLE "[a-z_]*"' $M
grep -c 'FOREIGN KEY' $M
```
Expected: `types: 7  tables: 10`; the `grep -n 'DROP...'` line prints **nothing** (back-relations are Prisma-side only and produce no SQL on the parent tables); the ten table names are exactly `tb_customer`, `tb_customer_address`, `tb_ar_invoice`, `tb_ar_invoice_detail`, `tb_ar_invoice_detail_dimension`, `tb_ar_invoice_reference`, `tb_ar_tax_invoice`, `tb_ar_receipt`, `tb_ar_receipt_detail`, `tb_ar_receipt_wht`; FOREIGN KEY count is 17 (customer→tax_profile 1, address→customer 1, invoice→customer 1, detail→invoice 1, detail_dimension→detail/dimension/value 3, reference→invoice ×2, tax_invoice→invoice/receipt 2, receipt→customer/bank 2, receipt_detail→receipt/invoice/detail 3, receipt_wht→receipt 1). If any other table appears, or any `DROP`, stop — the scratch DB is not at the `main` state; recreate it (Task 1 Steps 2–4) and redo this task.

- [ ] **Step 4: Apply the new migration to the scratch database and confirm zero drift**

```bash
DATABASE_URL="$SCRATCH_URL" bunx prisma migrate deploy
DATABASE_URL="$SCRATCH_URL" bunx prisma migrate diff --from-config-datasource --to-schema prisma/schema.prisma --exit-code; echo "exit=$?"
DATABASE_URL="$SCRATCH_URL" bunx prisma migrate status | tail -3
psql "$SCRATCH_URL" -Atc "select count(*) from information_schema.tables where table_schema='public' and table_name in ('tb_customer','tb_customer_address','tb_ar_invoice','tb_ar_invoice_detail','tb_ar_invoice_detail_dimension','tb_ar_invoice_reference','tb_ar_tax_invoice','tb_ar_receipt','tb_ar_receipt_detail','tb_ar_receipt_wht')"
```
Expected: `1 migration found ... applied`, `exit=0`, `Database schema is up to date!`, and `10`.

- [ ] **Step 5: Commit**

```bash
cd /Users/samutpra/GitHub/carmensoftware-organize/carmen-turborepo-backend-v2
git add "packages/prisma-shared-schema-tenant/prisma/migrations/${TS}_accounting_ar_tables/migration.sql"
git commit -m "feat(prisma): add accounting_ar_tables migration

Generated with prisma migrate diff against a scratch database at the
20260928130000 state, applied and re-diffed to an empty diff (ADD-only:
7 enums, 10 tables)."
```

---

### Task 4: Exclude the new tables from the audit extension and note the dormant tables

**Files:**
- Modify: `packages/prisma-shared-schema-tenant/src/client.ts:135-144` (`excludeModels`, AP group)
- Modify: `apps/micro-business/CLAUDE.md:28-30` (§Tenant migrations)

**Interfaces:**
- Consumes: the ten model names from Task 2.
- Produces: audit interceptor skips `tb_customer*` / `tb_ar_*`; a CLAUDE.md bullet that future AR service work must read.

- [ ] **Step 1: Add the ten names after the AP group in `excludeModels`**

```bash
cd /Users/samutpra/GitHub/carmensoftware-organize/carmen-turborepo-backend-v2
python3 - <<'PY'
p='packages/prisma-shared-schema-tenant/src/client.ts'; s=open(p,encoding='utf-8').read()
old="    'tb_ap_payment_expense',\n"
new=old+"""    // accounting — receivable (schema only as of 2026-09-30; no handlers yet, so no activity registry entries)
    'tb_customer',
    'tb_customer_address',
    'tb_ar_invoice',
    'tb_ar_invoice_detail',
    'tb_ar_invoice_detail_dimension',
    'tb_ar_invoice_reference',
    'tb_ar_tax_invoice',
    'tb_ar_receipt',
    'tb_ar_receipt_detail',
    'tb_ar_receipt_wht',
"""
assert s.count(old)==1; s=s.replace(old,new)
open(p,'w',encoding='utf-8').write(s); print('excludeModels updated')
PY
diff <(grep -o "'tb_customer[a-z_]*'\|'tb_ar_[a-z_]*'" packages/prisma-shared-schema-tenant/src/client.ts | tr -d "'" | sort) <(grep -o '^model tb_customer[a-z_]*\|^model tb_ar_[a-z_]*' packages/prisma-shared-schema-tenant/prisma/schema.prisma | sed 's/^model //' | sort) && echo "excludeModels matches schema"
```
Expected: `excludeModels updated` then `excludeModels matches schema` (empty diff).

- [ ] **Step 2: Add the CLAUDE.md bullet**

```bash
python3 - <<'PY'
p='apps/micro-business/CLAUDE.md'; s=open(p,encoding='utf-8').read()
anchor="rather than trusting `migrate dev` to produce a clean, feature-scoped diff.\n"
assert s.count(anchor)==1
bullet=("- **`tb_customer*` and `tb_ar_*` (migration `accounting_ar_tables`, 2026-09-30) are schema-only.** "
 "The 10 tables exist in every BU schema but no service, RPC contract, gateway route, running-code type, `gl_setting` key or activity-registry entry uses them yet — they were created ahead of the AR sub-project so the schema decisions in "
 "`carmen-accounting-concept/docs/superpowers/specs/2026-09-30-accounting-ar-schema-design.md` §1.1 are locked. "
 "They are already in `excludeModels` (same treatment as AP), so the first AR handler must add its activity-registry entries or it will leave no trail. "
 "`tb_ar_tax_invoice` has nullable `ar_invoice_id` / `ar_receipt_id`; exactly one must be set, enforced by the future service, not the schema.\n")
s=s.replace(anchor, anchor+bullet)
open(p,'w',encoding='utf-8').write(s); print('CLAUDE.md updated')
PY
grep -n 'schema-only' apps/micro-business/CLAUDE.md
```
Expected: `CLAUDE.md updated` and one matching line inside §Tenant migrations.

- [ ] **Step 3: Static checks — prisma package tests, types across the monorepo**

```bash
cd packages/prisma-shared-schema-tenant && bun run test; cd ../..
bun run check-types
git status --short
```
Expected: Vitest reports all tests passed (`client.test.ts` skips the DB-integration cases unless `VITEST_INCLUDE_DB_INTEGRATION=true`); Turbo `check-types` succeeds for every package; `git status --short` shows exactly `client.ts` and `CLAUDE.md` modified.

- [ ] **Step 4: Commit**

```bash
git add packages/prisma-shared-schema-tenant/src/client.ts apps/micro-business/CLAUDE.md
git commit -m "chore: exclude AR tables from audit extension, note schema-only status"
```

---

### Task 5: Push and open the PR (merge is the user's call)

**Files:**
- None.

**Interfaces:**
- Consumes: three commits from Tasks 2–4.
- Produces: PR against `main` in `carmen-turborepo-backend-v2` with the deploy checklist from spec §7.

- [ ] **Step 1: Final diff audit**

```bash
cd /Users/samutpra/GitHub/carmensoftware-organize/carmen-turborepo-backend-v2
git diff --stat origin/main..HEAD
```
Expected: exactly 4 files: `schema.prisma`, `migrations/<ts>_accounting_ar_tables/migration.sql`, `src/client.ts`, `apps/micro-business/CLAUDE.md`.

- [ ] **Step 2: Push and create the PR**

```bash
git push -u origin feature/accounting-ar-schema
gh pr create --base main --head feature/accounting-ar-schema --title "feat(prisma): AR and customer tables (schema only)" --body "$(cat <<'BODY'
## Summary
- Add 7 enums + 10 tables for AR (customer master, AR invoice/DN/CN/deposit, tax-invoice register, receipt) to the tenant schema, mirroring the AP tables field-for-field
- Migration `accounting_ar_tables` generated with `prisma migrate diff` against a scratch DB at the `20260928130000` state, applied and re-diffed to an empty diff — ADD-only
- Tables added to the audit-extension `excludeModels`; no services, RPC, gateway, seeds or `gl_setting` keys in this PR
- Design: `carmen-accounting-concept/docs/superpowers/specs/2026-09-30-accounting-ar-schema-design.md` (§1.1 lists the AR FRD open issues decided ahead of BA sign-off; they are cheap ALTERs later because the tables are empty)

## Deploy (after merge + Deploy Dev)
Migration is ADD-only, so old code keeps running on the new schema. Fire `POST /api-system/tenant/migrations/:bu_id/deploy` for every BU (14 as of 2026-09-07) and confirm each with `prisma migrate status`. Before firing, check no BU already has a hand-made `tb_customer` (`\dt tb_customer*`) — `CREATE TABLE` would fail on it.

## Verification
- `prisma validate`, `db:generate`, `bun run check-types`, prisma-package `vitest run`: pass
- migration.sql: 7 `CREATE TYPE`, 10 `CREATE TABLE`, 17 `FOREIGN KEY`, no `DROP`/`ALTER` on existing tables
BODY
)"
```
Expected: PR URL printed. Do **not** merge; report the URL and stop. Merging and the per-BU deploy are decided by the user.

- [ ] **Step 3: Drop the scratch database**

```bash
psql postgres -Atc "select datname from pg_database where datname='ar_scratch'"
dropdb ar_scratch
```
Expected: `ar_scratch` listed, then dropped without error.

---

### Task 6 (after the PR is merged): Sync the concept repo

**Files:**
- Modify: `carmen-accounting-concept/prisma/schema.prisma` (move AR from DRAFT §3/§4 to IMPLEMENTED §1/§2)
- Modify: `carmen-accounting-concept/README.md:42` (module table row 3, AR status)

**Interfaces:**
- Consumes: merged `main` of backend-v2 (record its short SHA as `MERGED_SHA`).
- Produces: concept schema whose IMPLEMENTED sections again equal the tenant schema for accounting tables.

- [ ] **Step 1: Regenerate the implemented sections from the merged tenant schema**

```bash
cd /Users/samutpra/GitHub/carmensoftware-organize/carmen-turborepo-backend-v2 && git switch main && git pull --ff-only && MERGED_SHA=$(git rev-parse --short HEAD) && echo "$MERGED_SHA"
cd /Users/samutpra/GitHub/carmensoftware-organize/carmen-accounting-concept
python3 - "$MERGED_SHA" <<'PY'
import re, sys
sha=sys.argv[1]
T='/Users/samutpra/GitHub/carmensoftware-organize/carmen-turborepo-backend-v2/packages/prisma-shared-schema-tenant/prisma/schema.prisma'
C='prisma/schema.prisma'
t=open(T,encoding='utf-8').read(); c=open(C,encoding='utf-8').read()
def block(src, kind, name):
    m=re.search(rf'^{kind} {name} \{{\n.*?^\}}\n', src, re.S|re.M); assert m, (kind,name)
    start=src[:m.start()].count('\n')+1; end=src[:m.end()].count('\n')
    return m.group(0), start, end
AR_MODELS=['tb_customer','tb_customer_address','tb_ar_invoice','tb_ar_invoice_detail','tb_ar_invoice_detail_dimension','tb_ar_invoice_reference','tb_ar_tax_invoice','tb_ar_receipt','tb_ar_receipt_detail','tb_ar_receipt_wht']
AR_ENUMS=['enum_ar_invoice_doc_type','enum_ar_invoice_status','enum_ar_invoice_source','enum_ar_tax_invoice_status','enum_ar_receipt_status','enum_ar_receipt_method','enum_customer_address_type']
# 1) remove the draft copies
for n in AR_ENUMS: c=re.sub(rf'^enum {n} \{{\n.*?^\}}\n\n?', '', c, flags=re.S|re.M)
for n in AR_MODELS: c=re.sub(rf'^model {n} \{{\n.*?^\}}\n\n?', '', c, flags=re.S|re.M)
c=re.sub(r'^// ---- Accounts Receivable \(DRAFT\) -+\n\n', '', c, flags=re.M)
# 2) append implemented enums before section 2 header, models before section 3 header
sec2=re.search(r'^// =+\n// 2\. IMPLEMENTED — models', c, re.M)
enum_txt=''.join(f'// source: schema.prisma:{s}-{e}\n{b}\n' for b,s,e in (block(t,'enum',n) for n in AR_ENUMS))
c=c[:sec2.start()]+enum_txt+c[sec2.start():]
sec3=re.search(r'^// =+\n// 3\. DRAFT — enums', c, re.M)
model_txt='// ---- Accounts Receivable '+'-'*51+'\n\n'+''.join(f'// source: schema.prisma:{s}-{e}\n{b}\n' for b,s,e in (block(t,'model',n) for n in AR_MODELS))
c=c[:sec3.start()]+model_txt+c[sec3.start():]
# 3) refresh the 4 parent models that gained back-relations
for n in ['tb_tax_profile','tb_gl_dimension','tb_gl_dimension_value','tb_bank_account']:
    newb,s,e=block(t,'model',n)
    if n=='tb_tax_profile':
        keep=[]; dropped=0
        for ln in newb.split('\n'):
            m=re.match(r'^\s+\w+\s+(tb_\w+)\[\]', ln)
            if m and not re.search(rf'^model {m.group(1)} \{{', c, re.M) and m.group(1) not in AR_MODELS: dropped+=1; continue
            keep.append(ln)
        keep.insert(1, f'  // NOTE: {dropped} back-relation list fields to non-accounting tables trimmed from this extract'); newb='\n'.join(keep)
    c=re.sub(rf'^(// source: schema\.prisma:)\d+-\d+\n(model {n} \{{\n.*?^\}}\n)', lambda m: f'{m.group(1)}{s}-{e}\n{newb}', c, count=1, flags=re.S|re.M)
# 4) header + counts
c=re.sub(r'@ b80024588 \(main, 2026-09-30\)', f'@ {sha} (main, 2026-09-30)', c)
c=c.replace('// AR draft (2026-09-30) was redesigned to mirror the implemented AP tables; see docs/PRD-module-ar.md §6.\n',
            '// AR tables were implemented (schema only) on 2026-09-30; see docs/superpowers/specs/2026-09-30-accounting-ar-schema-design.md.\n')
def count(kind, lo, hi): return len(re.findall(rf'^{kind} ', c[lo:hi], re.M))
i3=c.index('// 3. DRAFT'); i4=c.index('// 4. DRAFT')
c=re.sub(r'// 1\. IMPLEMENTED — enums \(\d+\)', f'// 1. IMPLEMENTED — enums ({count("enum",0,i3)})', c)
c=re.sub(r'// 2\. IMPLEMENTED — models \(\d+\)', f'// 2. IMPLEMENTED — models ({count("model",0,i3)})', c)
c=re.sub(r'// 3\. DRAFT — enums \(\d+\)', f'// 3. DRAFT — enums ({count("enum",i3,i4)})', c)
c=re.sub(r'// 4\. DRAFT — models \(\d+\)', f'// 4. DRAFT — models ({count("model",i4,len(c))})', c)
c=re.sub(r'\[tenant schema @ b80024588\]', f'[tenant schema @ {sha}]', c)
open(C,'w',encoding='utf-8').write(c); print('concept schema synced')
PY
grep -n '^// [1-4]\. ' prisma/schema.prisma
```
Expected: `concept schema synced`; header counts become `enums (28)`, `models (42)`, DRAFT `enums (7)`, `models (16)`.

- [ ] **Step 2: Re-run the structural checks used for PR #1**

```bash
cd /Users/samutpra/GitHub/carmensoftware-organize/carmen-accounting-concept
grep -oE '^\s+\w+\s+(tb_\w+)(\[\]|\?)?\s+@relation' prisma/schema.prisma | awk '{print $2}' | sed 's/\[\]//;s/?//' | sort -u | while read t; do grep -q "^model $t {" prisma/schema.prisma || echo "DANGLING: $t"; done
grep -oE '^\s+\w+\s+tb_\w+\[\]' prisma/schema.prisma | awk '{print $2}' | sed 's/\[\]//' | sort -u | while read t; do grep -q "^model $t {" prisma/schema.prisma || echo "MISSING: $t"; done
grep -oE '\benum_\w+' prisma/schema.prisma | sort -u | while read e; do grep -q "^enum $e {" prisma/schema.prisma || echo "UNDEFINED: $e"; done
echo "checks done"
```
Expected: only `checks done` (no DANGLING / MISSING / UNDEFINED lines). `tb_bank_reconciliation_item` (CB draft) still relates to `tb_ar_receipt`, which now exists as an implemented model, so no dangling target.

- [ ] **Step 3: Update the README module row**

```bash
python3 - <<'PY'
p='README.md'; s=open(p,encoding='utf-8').read()
old='| 3 | Accounts Receivable | AR | **New** | [PRD-module-ar.md](docs/PRD-module-ar.md) |'
new='| 3 | Accounts Receivable | AR | **Schema only** — 10 tables in the tenant schema (migration `accounting_ar_tables`), no service yet | [PRD-module-ar.md](docs/PRD-module-ar.md) · [schema spec](docs/superpowers/specs/2026-09-30-accounting-ar-schema-design.md) |'
assert s.count(old)==1; open(p,'w',encoding='utf-8').write(s.replace(old,new)); print('README updated')
PY
```
Expected: `README updated`.

- [ ] **Step 4: Commit**

```bash
git add prisma/schema.prisma README.md
git commit -m "docs: move AR tables to implemented section of concept schema (backend-v2 accounting_ar_tables)"
```
Do not push; the user decides.
