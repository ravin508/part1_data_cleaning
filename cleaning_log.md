# Cleaning Log - Task 2: Clean Text Fields
**Source:** raw_orders.xlsx (Sheet: raw_orders)  
**Output:** cleaned_orders.xlsx (Sheet: cleaned_orders)  
**Business Rules Reference:** Sheet 'business_rules' copied to output for reference.

## Scope
Cleaned and standardized the following text fields as specified:
- customer_name, segment, region, state, city, category, sub_category, ship_mode, payment_status, order_status

## Cleaning Decisions & Assumptions
1. **Whitespace Handling**: Applied strip (TRIM) + collapse multiple whitespace to single space. This fixes extra/leading/trailing spaces seen in raw data (e.g., '  North ', 'Ananya Rao ', 'Vikram  Iyer').
2. **Case Standardization**: Converted all to .title() (Title Case / PROPER). This resolved mixed case issues:
   - 'OFFICE SUPPLIES', 'office supplies', 'Office  Supplies' → 'Office Supplies'
   - 'FURNITURE', '  Furniture ' → 'Furniture'
   - 'TECHNOLOGY', 'Technology ' → 'Technology'
   - Customer names: 'PRIYA MENON', 'mira das', 'kabir singh', 'Ananya Rao ' → 'Priya Menon', 'Mira Das', 'Kabir Singh', 'Ananya Rao'
   - Segments: 'SMALL BUSINESS', 'Small  Business', '  Consumer ' → 'Small Business', 'Consumer'
   - Statuses: 'PENDING', 'failed', 'COMPLETED', 'cancelled' → 'Pending', 'Failed', 'Completed', 'Cancelled'
3. **Similar Name Variants**: Title case + space collapse naturally merged duplicates like 'Standard  Class'/'STANDARD CLASS'/'standard class' → 'Standard Class'; 'Machines'/'machines'/'MACHINES ' → 'Machines'.
4. **Missing Values**:
   - Region: 26 missing → filled with 'Unknown' (per business rule). Flagged in data_quality_flag column.
   - Ship_Mode: 22 missing → filled with 'Unknown' (per business rule). Flagged in data_quality_flag.
5. **Unwanted Special Characters**: None prominent in these fields; cleaning focused on spaces/case. No removal needed beyond whitespace.
6. **Other Notes**:
   - State and City were already clean (consistent Title Case, no extras).
   - Added data_quality_flag for rows affected by missing fills (simple rule-based flag).
   - Added calculated fields referenced in business_rules (calculated_sales, profit_margin, shipping_delay_days, order_month, order_year) for completeness of cleaned dataset. Date parsing used flexible dayfirst=True with NaT for unparseable dates.
   - Non-completed orders (Cancelled/Failed/Refunded) retained in data but can be filtered for summaries per business rule #7.
   - No duplicate removal or conflicting ID flagging done in this task (see rules for future).
   - Discount column has some string values ('70%', '85%') and negatives; left as-is for Task 2 (text cleaning only). Numeric coercion used only for calculated_sales.

## Verification (Post-Clean Unique Counts)
- customer_name: 15 (standardized names)
- segment: 4 → ['Consumer', 'Corporate', 'Home Office', 'Small Business']
- region: 5 → ['East', 'North', 'South', 'Unknown', 'West']
- category: 3 → ['Furniture', 'Office Supplies', 'Technology']
- sub_category: 13
- ship_mode: 5 → ['First Class', 'Same Day', 'Second Class', 'Standard Class', 'Unknown']
- payment_status: 4 → ['Failed', 'Paid', 'Pending', 'Refunded']
- order_status: 3 → ['Cancelled', 'Completed', 'Returned']

## Quality Report Summary
- Rows with data_quality_flag = 'Review - Missing Region/Ship Mode Filled': 47
- Rows OK: 885
- Total rows in cleaned file: 932

## Recommendations for Next Tasks
- Handle discount parsing (strings like '70%') and negative discounts (flag per rule).
- Parse/fix date inconsistencies for accurate shipping_delay.
- Implement duplicate detection (exact row dups vs conflicting order_id).
- For completed-sales summaries: filter where order_status == 'Completed' and payment_status == 'Paid'.
- Consider adding original_raw_value columns or change log if full audit trail needed.

**Assumption**: Title Case is appropriate standard for all listed fields (common in business data for readability and consistency). If specific casing required (e.g., UPPER for codes), adjust in future iteration.


---

## Task 5: Apply Business Rules 

### Rules Applied

**1. Missing region / ship_mode**
- Already handled in earlier tasks (filled with "Unknown" + flagged).
- Confirmed in current file: region and ship_mode have "Unknown" where originally missing.

**2. Missing discount**
- Rule: Treat as 0 **only if** quantity, unit_price, and sales are valid.
- Result: 18 missing discounts processed.
  - Most were imputed to 0 where sales fields were valid.
  - A few left as NaN where sales fields were also problematic.
- All decisions logged in `discount_status` column.

**3. Negative discount**
- 15 negative discounts found.
- Action: Flagged as invalid in `discount_status` and `data_quality_flag`.
- These rows should be excluded from financial summaries or investigated.

**4. Unusually high / invalid discount format**
- String values like '70%', '85%' detected and converted where possible.
- Any discount > 100% flagged as invalid.
- These are treated as data quality issues.

**5. Cancelled / Failed orders**
- These should **not** contribute to final completed sales summary.
- Added column: `is_valid_completed_sale` (False for Cancelled/Failed).
- Flagged in `data_quality_flag`.

**6. Refunded orders**
- Must be separately summarized.
- Added column: `is_refunded` (True for Refunded status or payment_status = Refunded).
- These are excluded from standard completed sales but can be analyzed separately.

**7. Ship date before order date**
- Already flagged in Task 3.
- Reinforced in Task 5 with explicit column `is_invalid_shipping`.
- These are considered invalid shipping records.

### New Columns Added
| Column                        | Purpose |
|------------------------------|---------|
| `discount_clean`             | Clean numeric discount value |
| `discount_status`            | Detailed status of discount cleaning/validation |
| `is_valid_completed_sale`    | Boolean - safe to include in completed sales summary |
| `is_refunded`                | Boolean - should be summarized separately |
| `is_invalid_shipping`        | Boolean - ship date before order date |

### Documentation
All business rule decisions are documented above and reflected in the `data_quality_flag` column for filtering and reporting.

**Recommendation**: 
- Use `is_valid_completed_sale == True` for all standard sales reports.
- Create separate refund analysis using `is_refunded == True`.
- Review rows where `discount_status` contains "invalid" or "Negative".



---

## Task 6: Create Calculated Columns 

### Columns Created / Standardized

| Column Name            | How It Was Created / Updated |
|------------------------|------------------------------|
| **cleaned_discount**   | Standardized numeric discount value (from Task 5 `discount_clean`). Renamed to exact Task 6 requirement. |
| **calculated_sales**   | `quantity × unit_price × (1 - cleaned_discount)`. Recalculated using the cleaned discount value. |
| **calculated_profit**  | `calculated_sales - cost`. Newly created as per Task 6. |
| **profit_margin**      | `profit / sales × 100`. Already existed; recalculated for consistency with new calculated_sales. |
| **shipping_delay_days**| Difference between ship_date and order_date (absolute days). Already created in Task 3; verified. |
| **order_month**        | Extracted from order_date using `.dt.month`. Ensured present. |
| **order_year**         | Extracted from order_date using `.dt.year`. Ensured present. |
| **data_quality_flag**  | Comprehensive flag combining all data quality issues (discount, shipping, duplicates, order status, etc.). Updated throughout Tasks 3–6. |

### Calculation Logic Notes
- All monetary calculations use **cleaned_discount** to ensure consistency after discount validation rules from Task 5.
- `calculated_profit` gives the expected profit based on the cleaned discount (useful for comparison against the original `profit` column).
- `profit_margin` is based on the original `profit` and `sales` (as provided in source data) for transparency.
- Date-based columns (`shipping_delay_days`, `order_month`, `order_year`) use the cleaned datetime versions of `order_date` and `ship_date`.

### Final Column Inventory (Relevant Calculated Fields)
- cleaned_discount
- calculated_sales
- calculated_profit
- profit_margin
- shipping_delay_days
- order_month
- order_year
- data_quality_flag
- is_valid_completed_sale (bonus - very useful for reporting)
- is_refunded (bonus)
- is_invalid_shipping (bonus)

All Task 6 requirements have been met and documented.

