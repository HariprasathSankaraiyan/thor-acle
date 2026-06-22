# Chapter 22: OMS Functional - Pricing & Tax
## ⏱️ ~15 min read

---

## 🧠 Think of it this way

> **Pricing** = How Oracle decides what price to charge.
> Like a supermarket: regular price, membership discount, buy-2-get-1-free, loyalty points - all layered.
>
> **Price List** = The menu. One for regular customers, one for VIP, one for wholesale.
>
> **Modifier** = The coupon or promotion. "10% off this week for returning customers."
>
> **Tax** = What the government adds on top.

---

## 1. Oracle Advanced Pricing - Architecture

```
Customer Places Order
       │
       ▼
Pricing Engine invoked
       │
       ├── Step 1: Find applicable Price Lists
       │           (by currency, dates, customer, order type)
       │
       ├── Step 2: Get List Price from Price List
       │
       ├── Step 3: Apply Qualifiers
       │           (Is this customer eligible for this price/modifier?)
       │
       ├── Step 4: Apply Modifiers (discounts, surcharges, promotions)
       │
       └── Step 5: Calculate Selling Price
                   List Price -> Apply Discounts -> Unit Selling Price
```

---

## 2. Price Lists

> A master catalog of item prices. Each price list can have different prices for different items.

### Price List Structure
```
Price List: "Corporate Customers USD"
  ├── Effective Dates: 01-Jan-2024 to 31-Dec-2024
  ├── Currency: USD
  ├── Rounding Factor: -2 (to 2 decimal places)
  └── Price List Lines:
        Item A: $100.00
        Item B: $250.00
        Item Category "Electronics": $XXX.00 (all items in category)
        All Items (formula): Cost × Markup %
```

### Price List Tables
```sql
QP_LIST_HEADERS_B         -- Price list header (name, dates, currency)
QP_LIST_HEADERS_TL        -- Translated descriptions
QP_LIST_LINES             -- Price list lines (item -> price)
QP_PRICING_ATTRIBUTES     -- Pricing attributes (item, category, etc.)
```

```sql
-- Find price lists
SELECT qlh.list_header_id, qlh.name, qlh.list_type_code,
       qlh.currency_code, qlh.active_flag,
       qlh.start_date_active, qlh.end_date_active
FROM qp_list_headers_b qlh
WHERE qlh.active_flag = 'Y'
AND SYSDATE BETWEEN NVL(qlh.start_date_active, SYSDATE - 1)
                AND NVL(qlh.end_date_active, SYSDATE + 1);

-- Find price for a specific item in a price list
SELECT qlh.name AS price_list, qll.operand AS list_price,
       qpa.product_attr_value AS item
FROM qp_list_headers_b qlh
JOIN qp_list_lines qll ON qlh.list_header_id = qll.list_header_id
JOIN qp_pricing_attributes qpa ON qll.list_line_id = qpa.list_line_id
WHERE qlh.name = 'Corporate Customers USD'
AND qpa.product_attribute = 'PRICING_ATTRIBUTE1'   -- Item attribute
AND qpa.product_attr_value = '12345';              -- Inventory item ID
```

---

## 3. Modifiers (Discounts, Surcharges, Promotions)

> Rules that change (modify) the price up or down.
> "Give 10% discount if order total > $5000"
> "Add $50 freight surcharge"

### Modifier Types
| Type | Description | Example |
|------|------------|---------|
| **Discount** | Reduce price | 10% off |
| **Surcharge** | Increase price | $50 handling fee |
| **Price Break** | Tiered pricing | 1-10 qty: $100, 11-100: $90 |
| **Promotional Goods** | Free item | Buy 10, get 1 free |
| **Freight/Special Charge** | Add shipping cost | $25 shipping |

### Modifier Header = List (like Price List but for discounts)
```sql
QP_LIST_HEADERS_B   -- Modifier header (list_type_code = 'DIS', 'SUR', 'PML'...)
QP_LIST_LINES       -- Modifier lines (the actual discount %)
QP_QUALIFIERS       -- Who qualifies for this modifier
QP_PRICING_ATTRIBUTES -- What items/categories it applies to
```

---

## 4. Qualifiers - Who Gets the Price/Discount?

> The eligibility rules. "This discount is ONLY for customers in the 'Gold' customer class AND orders placed in January."

### Qualifier Types
- Customer name / Account
- Customer Class (Bronze, Silver, Gold)
- Order Type
- Order Amount (> $X)
- Dates (promotion window)
- Item category

```sql
QP_QUALIFIERS         -- Qualifier rules
QP_QUALIFIER_GROUPS   -- Group of qualifiers (AND/OR logic)
```

---

## 5. Pricing in OMS - What the User Sees

On the Sales Order line:
```
Ordered Quantity:     10
Unit List Price:    $100.00  ← from price list
Discount %:          10%    ← from modifier
Unit Selling Price:  $90.00  ← list price - discount
Extended Amount:    $900.00  ← selling price × qty
```

### Price Adjustments Table
```sql
OE_PRICE_ADJUSTMENTS      -- Discounts/surcharges applied to order lines
  list_header_id           -- Which modifier/price list
  list_line_id             -- Which line in modifier
  adjusted_amount          -- Dollar amount of adjustment
  percent                  -- Percentage
  applied_flag             -- 'Y' if applied
```

---

## 6. Pricing Formulas

> For complex pricing that can't be a fixed amount - e.g., "Price = Cost + 20%"

```sql
QP_PRICING_FORMULAS       -- Formula definitions
QP_FORMULA_LINES          -- Formula calculation steps
```

---

## 7. Repricing an Order

When you:
- Change quantity on a line
- Change price list
- Add/remove a modifier
- Change a date

Oracle re-invokes the pricing engine. You can also manually reprice:
```
Order Line -> Actions -> Price Line (reprice the line)
Order Header -> Actions -> Reprice Order
```

---

## 8. Tax in Oracle EBS

> Tax is calculated based on: Item taxability + Customer location + Tax rules.

### Tax Approaches in EBS
| Approach | Version | Description |
|----------|---------|-------------|
| **Sales Tax (Basic)** | R11i | Basic US sales tax, location-based |
| **E-Business Tax (eBTax)** | R12 | Full global tax engine, replaces old |

---

## 9. E-Business Tax (eBTax) - R12

> A rules engine that figures out what tax applies based on WHERE you are, WHAT you're selling, and WHO you're selling to.

### Key Concepts
- **Tax Regime**: The overarching tax law (e.g., "US Sales Tax")
- **Tax**: Specific tax within regime (e.g., "California State Tax")
- **Tax Status**: Taxable / Exempt / Zero-rated
- **Tax Rate**: The percentage (e.g., 8.5%)
- **Tax Jurisdiction**: Geographic area (California, Cook County)
- **Tax Applicability Rule**: When to apply which tax

### Tax Determination Flow
```
Order Line
    │
    ▼
eBTax Engine
    ├── Check Party Tax Profile (Is customer tax-exempt?)
    ├── Check Product Tax Classification (Is item taxable?)
    ├── Determine Jurisdiction (Ship-to location)
    ├── Find applicable Tax + Rate
    └── Calculate Tax Amount
```

### Key eBTax Tables
```sql
ZX_REGIMES_B              -- Tax regimes
ZX_TAXES_B                -- Taxes
ZX_RATES_B                -- Tax rates
ZX_JURISDICTIONS_B        -- Jurisdictions
ZX_PARTY_TAX_PROFILE      -- Customer/supplier tax profile
ZX_LINES                  -- Tax lines on transactions
ZX_TRANSACTION_LINES      -- Link to transaction lines
```

```sql
-- Find tax lines on an order/invoice
SELECT zl.tax_regime_code, zl.tax, zl.tax_status_code,
       zl.tax_rate, zl.tax_amt, zl.taxable_amt
FROM zx_lines zl
WHERE zl.trx_id = (SELECT header_id FROM oe_order_headers_all WHERE order_number = 10001)
AND zl.application_id = 660;  -- 660 = OM application
```

---

## 10. Tax Exemptions

> Some customers or items are tax-exempt:
- **Customer Exempt**: Nonprofit, government, reseller
- **Item Exempt**: Food, medicine in some jurisdictions

### Customer Tax Exemption
```sql
-- Check customer tax exemption
SELECT he.exempt_number, he.exempt_percent, he.customer_name,
       he.bill_to_site_use_id, he.tax_code
FROM ra_tax_exemptions he
WHERE he.customer_id = 12345
AND he.status = 'PRIMARY'
AND SYSDATE BETWEEN he.start_date AND NVL(he.end_date, SYSDATE + 1);
```

---

## 🎯 Top Interview Questions & Answers

**Q1: What is a Price List and what does it contain?**
> A price list is a collection of item prices organized by item or category. It's associated with a currency, date range, and rounding rules. The Pricing Engine looks up item prices from the applicable price list.

**Q2: What is the difference between List Price and Selling Price?**
> List Price is the base price from the price list. Selling Price (Unit Selling Price) is after modifiers (discounts/surcharges) are applied. Extended Amount = Selling Price × Quantity.

**Q3: What are Qualifiers in Oracle Pricing?**
> Rules that determine eligibility - which customers/orders/items a price or modifier applies to. Examples: customer class = 'GOLD', order amount > $10,000, order type = 'WEB_ORDER'.

**Q4: What is E-Business Tax (eBTax)?**
> Oracle's global tax engine introduced in R12. Replaced older sales tax approach. Handles complex tax scenarios using configurable rules for regime, tax, jurisdiction, and rates.

**Q5: How do you make a customer tax-exempt in Oracle?**
> Create a Tax Exemption for the customer/site in RA_TAX_EXEMPTIONS (R11i) or configure the Party Tax Profile as exempt in eBTax (R12). The exemption is tied to customer, ship-to site, tax type, and date range.

**Q6: How is the pricing engine triggered on an order?**
> Automatically when a line is added/updated, when dates change, or when you click Price Line/Reprice Order manually. Can also be programmatically triggered via OE_ORDER_PUB.PROCESS_ORDER with pricing request.

---

## 📝 Quick Revision Summary

```
Price List: master price catalog (item -> price)
  QP_LIST_HEADERS_B, QP_LIST_LINES, QP_PRICING_ATTRIBUTES

Modifiers: change price (discount/surcharge/promo)
  List Price -> Apply Modifier -> Unit Selling Price

Qualifiers: eligibility rules (who gets what price)
  Customer class, order amount, dates, order type

Order line:
  Unit List Price   -> from price list
  Unit Selling Price -> after modifiers
  Extended Amount  -> selling price × qty

Price Adjustments: OE_PRICE_ADJUSTMENTS -> applied modifiers

eBTax (R12):
  Regime -> Tax -> Rate -> Jurisdiction
  Check: customer exempt? Item taxable? Ship-to location?
  ZX_LINES -> tax applied to transaction

Tax Exemptions:
  RA_TAX_EXEMPTIONS (R11i)
  ZX_PARTY_TAX_PROFILE (R12)
```

---

*Next: [Chapter 23 - OMS Shipping & Fulfillment](23_oms_shipping_fulfillment.md)*
