# Data Migration Project — Source Data Setup

This describes how the source files for your migration project were generated and what kinds of issues were intentionally injected. Use this document as a reference when designing your cleansing logic, and keep `degradation_changes_detail.csv` as your reconciliation ground truth.

## Files in this package

| File | Purpose | Rows |
|---|---|---|
| `customer_master_clean.csv` | Ground-truth customer master (enriched, no degradation) | 99,457 |
| `customer_master_degraded.csv` | **Migration source — your script reads this** | 102,440 |
| `sales_data_clean.csv` | Ground-truth sales data (original, unchanged) | 99,457 |
| `sales_data_degraded.csv` | **Migration source — your script reads this** | 100,650 |
| `degradation_changes_detail.csv` | Row-level log of every change made | 169,625 |

## Scenario framing

You can use the following business scenario in your README:

> Acme Retail Group operated a chain of 10 shopping mall outlets across Turkey, with a customer base spanning Turkey, Germany, the US, the UK, and France. The legacy CRM was poorly maintained over a decade of mergers, multiple data entry teams, and at least one botched character-encoding migration. Acme is now migrating to SAP S/4HANA. Your job is to cleanse, validate, transform, and load the customer master and historical sales data, producing load-ready files plus a reconciliation report identifying every record that was rejected and why.

## Customer master enrichment (clean baseline)

The original four-column customer file (`customer_id`, `gender`, `age`, `payment_method`) was enriched to a 23-field SAP-style customer master. Each `customer_id` was assigned a country (~80% Turkey, ~10% Germany, ~5% US, ~3% UK, ~2% France) and given consistent locale-appropriate name, address, contact info, tax ID, and SAP organizational fields:

- **Identity:** `first_name`, `last_name`
- **Address:** `street`, `city`, `region`, `postal_code`, `country`
- **Contact:** `phone`, `email`
- **Tax/Legal:** `tax_id`
- **SAP org structure:** `account_group`, `sales_org`, `distribution_channel`, `division`
- **Financial:** `currency`, `payment_terms`, `credit_limit`
- **Other:** `language`, `created_date`
- **Original fields preserved:** `gender`, `age`, `payment_method`

The clean enriched file is your reconciliation target — every record in it should ultimately be accounted for in your migration output (loaded, rejected, or flagged for manual review).

## Degradation summary (medium aggressiveness)

**Customer master** — 92,818 unique original rows had at least one issue injected (93.3% of records affected), plus 2,983 duplicate rows added.

**Sales data** — 40,220 unique original rows had at least one issue injected (40.4% of records affected), plus 1,193 duplicate invoices added.

### Categories injected — customer master

| Category | Changes | What you'll see |
|---|---:|---|
| Format inconsistencies | 77,928 | Mixed country codes (`TR`/`Turkey`/`TÜRKİYE`/`tr`), 5+ phone formats, mixed-case emails, name capitalization (UPPER/lower/Mixed), leading whitespace, leading zeros stripped from postal codes, multiple date formats (`%d/%m/%Y`, `%m-%d-%Y`, `%b %d, %Y`, etc.) |
| Missing data | 22,872 | Random nulls in non-required fields (phone, email, tax_id, region, credit_limit) |
| Junk placeholders | 7,458 | `N/A`, `NULL`, `Unknown`, `TBD`, `-`, `?`, blank-spaces — used in place of nulls |
| Invalid values | 5,269 | Malformed emails (missing `@`, missing TLD), out-of-range ages (`-5`, `0`, `200`, `999`), inconsistent gender codes (`M`/`MALE`/`1`/`Man`), negative credit limits |
| Encoding issues | 1,197 | UTF-8 → Latin-1 mojibake on Turkish characters (e.g., `Şahin` → `ÅŸahin`, `Müller` → `MÃ¼ller`). The classic post-migration encoding bug. |
| Missing required | 2,485 | Nulls in `first_name`, `last_name`, or `country` — these should be **rejected** by your cleansing logic |
| Duplicates | 2,983 | ~1% exact duplicates, ~2% near-duplicates (trailing whitespace, capitalization variants, `Street`→`St.`) |

### Categories injected — sales data

| Category | Changes | What you'll see |
|---|---:|---|
| Format inconsistencies | 36,545 | Date format mixing (source is `%d-%m-%Y`, ~20% reformatted), prices with European decimal commas, leading dollar signs, whitespace, mall name case/whitespace |
| Missing data | 6,961 | Nulls in `shopping_mall`, `category`, `quantity` |
| Invalid values | 2,745 | Negative quantities, negative prices, zero quantities, text in numeric quantity field (`"five"`, `"N/A"`) |
| Referential integrity | 1,989 | ~2% of sales rows reference a `customer_id` not in the customer master (orphan transactions) |
| Duplicates | 1,193 | ~1.2% exact duplicate invoices |

## How to use the detailed change log

`degradation_changes_detail.csv` contains every individual change with these columns:

- `file` — `customers` or `sales`
- `row_index` — index in the degraded file (or `NEW_n` for added duplicate rows)
- `customer_id` — for traceability
- `field`, `original_value`, `degraded_value` — what changed
- `category` — degradation type
- `reason` — short description

This is your **reconciliation ground truth.** Your final migration script can compare its output against this to prove:

- Every issue you should have caught, you did
- Every record that should have been rejected, was
- The reject-reason categories your script reports match the categories injected

## Suggested cleansing checklist for your script

A solid migration script should at minimum handle:

1. **Whitespace normalization** — strip leading/trailing whitespace from all string fields
2. **Junk placeholder detection** — convert `N/A`, `NULL`, `Unknown`, `TBD`, `-`, `?` to actual nulls before validation
3. **Country code standardization** — map all variants to ISO 3166-1 alpha-2
4. **Phone normalization** — strip to digits, apply country-specific format
5. **Email validation** — regex check, lowercase, then deduplicate by email
6. **Encoding repair** — detect and fix mojibake (a common approach: `s.encode('latin-1').decode('utf-8')` if a heuristic detects mojibake patterns like `Ã©`, `Å`, etc.)
7. **Date parsing** — try multiple formats; reject if unparseable
8. **Postal code repair** — pad with leading zeros where country format requires
9. **Range validation** — age 0–120, no negative credit limits/quantities/prices
10. **Gender normalization** — map all variants to a canonical set
11. **Duplicate detection** — exact and fuzzy (after normalization)
12. **Referential integrity check** — flag sales rows whose `customer_id` is not in the customer master

## Reconciliation report — suggested format

Your final reconciliation should show, at minimum:

- Records read in (per file)
- Records loaded (clean output)
- Records rejected (with breakdown by reason)
- Records cleansed but loaded (e.g., reformatted phone numbers — counts of how many got fixed in each category)
- Cross-check: rejected count + loaded count = read count
- Cross-check: against this `degradation_changes_detail.csv`, how many of the injected issues did your script detect and address?

---

*Generated with seed=42, so the degradation is reproducible if you ever need to regenerate.*
