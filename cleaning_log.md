# Cleaning Log — raw_orders.xlsx

## 1. Issues Found

### Text Fields
- **Case inconsistencies**: All text fields (`segment`, `region`, `category`, `sub_category`, `ship_mode`, `payment_status`, `order_status`) contained mixed-case variants (UPPER, lower, Title, mixed).
- **Extra/leading/trailing spaces**: Found in multiple fields (e.g., `"  Consumer "`, `"North "`, `"Completed "`).
- **Double spaces in values**: e.g., `"Office  Supplies"`, `"Small  Business"`, `"Standard  Class"`.
- **Variant spellings**: e.g., `"Smallbusiness"` → `"Small Business"`, `"Homeoffice"` → `"Home Office"`.

### Date Fields
- **Mixed date formats**: `order_date` and `ship_date` used at least 5 different formats:
  - `21 Jul 2024` (human-readable)
  - `08/31/2024` (MM/DD/YYYY)
  - `2024-05-24` (ISO)
  - `28-11-2024` (DD-MM-YYYY)
  - `05 Sep 2024` (D Mon YYYY)
- **Ship date before order date**: 35 records had `ship_date < order_date` (invalid).

### Duplicates
- **Exact duplicate rows**: 20 found — identical in all 21 columns.
- **Conflicting duplicate order_ids**: 12 additional rows shared an `order_id` but had differing field values.
- **Total removed**: 32 rows.

### Missing Values
- `region`: 26 missing
- `ship_mode`: 22 missing
- `discount`: 18 missing

### Discount Issues
- **Negative discounts**: Multiple records with negative discount values (e.g., -0.19, -0.23).
- **Percentage strings**: Some discounts stored as `"70%"`, `"85%"` instead of numeric decimals.
- **Missing discounts**: 18 records with null discount.
- **Over-range discounts**: Values > 1.0 after % conversion.

### Sales/Profit Mismatches
- **62 records** where `calculated_sales (qty × unit_price × (1-discount))` differed from the stored `sales` by more than ₹1.
- Likely caused by invalid discount values being used in original calculation.

### Order Status Issues
- Mix of Completed, Cancelled, Returned across different case variants (cleaned).
- Cancelled and Failed payment orders must be excluded from completed sales summaries.

---

## 2. Cleaning Actions Performed

| Step | Action |
|------|--------|
| Text normalization | Applied `.strip()`, collapsed multiple spaces, converted to `.title()` case |
| Date parsing | Tried 5+ format strings; fell back to `pd.to_datetime()` with `dayfirst=False` |
| Date standardization | All dates stored as Python datetime, formatted `DD-MMM-YYYY` in output |
| Exact deduplication | Removed 20 exact duplicate rows (kept first occurrence) |
| Conflict deduplication | Removed 12 rows with duplicate `order_id` and differing data (kept first) |
| Discount cleaning | Converted `"70%"` → `0.70`; negative values flagged; missing → 0 |
| Missing region | Filled with `"Unknown"` |
| Missing ship_mode | Filled with `"Unknown"` |

---

## 3. Business Rules Applied

| Rule | Action Taken |
|------|-------------|
| Missing region | Filled as `Unknown`, flagged in quality report |
| Missing ship_mode | Filled as `Unknown`, flagged in quality report |
| Missing discount | Treated as 0 where all other sales fields valid |
| Negative discount | Flagged as `Invalid_Negative`; used discount=0 for `calculated_sales` |
| Discount > 1.0 | Flagged as `Invalid_Over_Range`; used discount=0 for `calculated_sales` |
| Cancelled orders | Excluded from completed sales pivot summaries |
| Failed payments | Excluded from completed sales pivot summaries |
| Returned orders | Separately summarized in P5 of pivot report |
| Ship before order | Flagged as `Invalid` in `data_quality_flag` column |

---

## 4. Assumptions Made

1. Date format `08/31/2024` is parsed as MM/DD/YYYY (US format), not DD/MM/YYYY, since values like `08/31` cannot be day-first.
2. Where `order_id` conflicts exist, the **first occurrence** is treated as the primary record.
3. Discount values stored as percentages (e.g., `"70%"`) are divided by 100.
4. `calculated_sales = quantity × unit_price × (1 − cleaned_discount)` is the correct formula.
5. `calculated_profit = calculated_sales − cost` (not `sales − cost`).
6. Records with `data_quality_flag = Invalid` are excluded from pivot summaries but kept in cleaned file for traceability.

---

## 5. Records Removed

| Reason | Count |
|--------|-------|
| Exact duplicate rows | 20 |
| Conflicting duplicate order_ids | 12 |
| **Total removed** | **32** |

---

## 6. Records Flagged

| Flag | Count | Reason |
|------|-------|--------|
| Clean | 786 | No issues detected |
| Warning | 79 | Minor issues (unknown region/ship_mode, sales mismatch) |
| Invalid | 35 | Missing dates, ship before order, invalid discount |

---

## 7. Limitations

1. **Date ambiguity**: Dates like `06/08/2024` could be June 8 or August 6 depending on locale. Assumed MM/DD/YYYY where day > 12 rules it out, otherwise US format.
2. **Conflict duplicate handling**: For conflicting `order_id` rows, no business logic exists to determine which is "correct" — first row is retained.
3. **Sales mismatch**: Cannot determine whether original `sales` or recalculated value is correct; both preserved in output.
4. **Product name** not cleaned (free-text, no canonical list available).
5. **Cost field** not validated — assumed correct as-is.
