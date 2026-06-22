# Chapter 25: OMS Integration - AR, INV, GL & Troubleshooting
## ⏱️ ~20 min read

---

## 🧠 Think of it this way

> Oracle OMS doesn't work alone - it's like a relay race.
> OMS books the order (runner 1), Shipping ships it (runner 2), AR invoices it (runner 3), GL records it (runner 4).
>
> When something breaks in the relay - a "drop" - you need to know WHERE the baton was dropped and WHY.
> That's integration troubleshooting.

---

## 1. The Full Integration Flow

```
Oracle Order Management (OMS)
  ├── Customer places order -> OE_ORDER_HEADERS_ALL
  ├── Order Booked -> Workflow, Credit Check, Scheduling
  ├── Pick Release -> WSH_DELIVERY_DETAILS
  ├── Ship Confirm -> SHIPPED status
  │
  ▼ (After Ship Confirm)
Shipping Interface -> RA_INTERFACE_LINES_ALL (staging)
  │
  ▼ (AutoInvoice Program)
Oracle AR (Accounts Receivable)
  ├── RA_CUSTOMER_TRX_ALL (Invoice header)
  ├── RA_CUSTOMER_TRX_LINES_ALL (Invoice lines)
  │
  ▼ (Revenue Recognition / Posting)
Oracle GL (General Ledger)
  ├── GL_JE_HEADERS (Journal header)
  └── GL_JE_LINES   (Journal lines - DR AR, CR Revenue)
  
Also at Ship Confirm:
  ▼
Oracle Inventory (INV)
  ├── MTL_MATERIAL_TRANSACTIONS (inventory decremented)
  ├── MTL_TRANSACTION_ACCOUNTS (accounting entries)
  └── DR: COGS, CR: Inventory Asset
```

---

## 2. OMS -> AR Interface (AutoInvoice)

### What is AutoInvoice?
- An Oracle AR concurrent program that takes data from `RA_INTERFACE_*` staging tables and creates real AR invoices

### Interface Tables
```sql
RA_INTERFACE_LINES_ALL       -- Invoice line data (from OMS ship confirm)
RA_INTERFACE_DISTRIBUTIONS   -- Accounting distributions
RA_INTERFACE_SALESCREDITS    -- Sales rep credits
RA_INTERFACE_ERRORS          -- Errors that prevented invoice creation
```

### How to Monitor AutoInvoice
```sql
-- Check for interface errors (invoices that failed)
SELECT ril.interface_line_id, ril.interface_line_context,
       ril.interface_line_attribute1 AS order_number,
       ril.amount, ril.error_flag,
       rie.message_text AS error_message
FROM ra_interface_lines_all ril
LEFT JOIN ra_interface_errors rie ON ril.interface_line_id = rie.interface_line_id
WHERE ril.interface_status = 'P'   -- 'P' = Pending (failed)
AND ril.request_id IS NOT NULL
ORDER BY ril.creation_date DESC;

-- Check successfully created invoices
SELECT rct.trx_number, rct.trx_date, rct.invoice_currency_code,
       rct.bill_to_customer_name, rct.amount_due_original
FROM ra_customer_trx_all rct
WHERE rct.interface_header_attribute1 = '10001'   -- Order number
AND rct.type = 'INV';
```

### Run AutoInvoice
```
Oracle Receivables -> Interfaces -> AutoInvoice Master Program
Parameters:
  Invoice Source = ORDER ENTRY
  Default Date
  Run Validation (Yes/No)
```

---

## 3. OMS -> INV Interface (COGS/Inventory)

### When Ship Confirm happens:
1. `MTL_MATERIAL_TRANSACTIONS` record created - inventory movement
2. `MTL_TRANSACTION_ACCOUNTS` - accounting entries
3. **Debit**: COGS (Cost of Goods Sold)
4. **Credit**: Inventory Asset account

```sql
-- Check inventory transaction for a shipped order
SELECT mmt.transaction_id, mmt.transaction_date,
       mmt.transaction_type_id, mtt.transaction_type_name,
       mmt.inventory_item_id, mmt.transaction_quantity,
       mmt.transaction_uom,
       mmt.organization_id
FROM mtl_material_transactions mmt
JOIN mtl_transaction_types mtt ON mmt.transaction_type_id = mtt.transaction_type_id
WHERE mmt.source_header_id = (
  SELECT header_id FROM oe_order_headers_all WHERE order_number = '10001'
)
ORDER BY mmt.transaction_date;
```

---

## 4. OMS -> GL Flow

### Journal Entries Created by OM

**At Ship Confirm (Inventory side)**:
```
DR: Cost of Goods Sold (COGS)    $75.00
CR: Inventory Asset               $75.00
```

**At AutoInvoice (AR side)**:
```
DR: Accounts Receivable           $100.00
CR: Revenue                       $100.00
```

**At Payment (AR side)**:
```
DR: Cash                          $100.00
CR: Accounts Receivable           $100.00
```

---

## 5. Common Integration Problems & Fixes

### Problem 1: Order SHIPPED but no AR Invoice

**Symptom**: Order line = SHIPPED, but no invoice in AR.

**Investigate**:
```sql
-- Check if interface record exists in AutoInvoice staging
SELECT COUNT(*) FROM ra_interface_lines_all
WHERE interface_line_attribute1 = '10001'  -- Order number
AND interface_line_context = 'ORDER ENTRY';

-- If zero rows: ship confirm didn't create interface record
-- Check: Shipping Parameters -> OM Interface enabled?
-- Check: Run "Inventory Interface" and "Interface Trip Stop" programs
```

**Fix Steps**:
1. Run "Interface Trip Stop" -> pushes ship confirm to OM
2. Run "Workflow Background Process" for ONT/WSH
3. Run AutoInvoice
4. Check RA_INTERFACE_ERRORS for specific error

---

### Problem 2: AutoInvoice Error

**Symptom**: Interface record exists but invoice not created. Error in RA_INTERFACE_ERRORS.

**Common AutoInvoice errors**:
| Error | Cause | Fix |
|-------|-------|-----|
| Invalid Bill-to Customer | Customer not set up in AR | Create/fix customer in AR |
| Invalid Payment Terms | Payment terms not in AR | Set up payment terms |
| Invalid Revenue Account | Wrong GL account | Fix account code combination |
| Line type mismatch | OM line type ≠ AR transaction type | Check transaction type setup |
| Invalid Sales Rep | Sales rep not in AR | Add sales rep to AR |

```sql
-- Get detailed error for an interface line
SELECT rie.error_message, rie.attribute_name, rie.invalid_value
FROM ra_interface_errors rie
WHERE rie.interface_line_id = (
  SELECT interface_line_id FROM ra_interface_lines_all
  WHERE interface_line_attribute1 = '10001'
  AND interface_line_context = 'ORDER ENTRY'
);
```

---

### Problem 3: Order Line Stuck in AWAITING_SHIPPING

**Investigate**:
```sql
-- Check if delivery detail exists
SELECT wdd.released_status, wdd.delivery_detail_id
FROM wsh_delivery_details wdd
WHERE wdd.source_header_number = '10001';

-- Check for scheduling issues
SELECT ool.schedule_status_code, ool.schedule_ship_date,
       ool.request_date
FROM oe_order_lines_all ool
JOIN oe_order_headers_all ooh ON ool.header_id = ooh.header_id
WHERE ooh.order_number = '10001';
```

**Common causes**:
- Line not scheduled (run AutoSchedule)
- Insufficient inventory (check MTL_ONHAND_QUANTITIES)
- Active hold (check OE_ORDER_HOLDS_ALL)
- Workflow stuck (check WF_ITEM_ACTIVITY_STATUSES)

---

### Problem 4: Booking Error

**Investigate**:
```sql
-- Check workflow error
SELECT wias.activity_status, wias.error_name, wias.error_message
FROM wf_item_activity_statuses wias
WHERE wias.item_type = 'OEOH'
AND wias.item_key = TO_CHAR(:header_id)
AND wias.activity_status = 'ERROR';

-- Check for mandatory field errors
SELECT oel.message_name, oel.message_text
FROM oe_msg_hist oel
WHERE oel.header_id = :header_id;
```

---

## 6. Key Concurrent Programs for OMS Integration

| Program | Purpose |
|---------|---------|
| **Order Import** | Import orders from staging tables |
| **Workflow Background Process** | Process deferred workflow activities |
| **Interface Trip Stop** | Transfer ship confirm to OM |
| **Inventory Interface** | Create MTL_MATERIAL_TRANSACTIONS |
| **AutoInvoice Master Program** | Create AR invoices from interface |
| **AutoInvoice Purge** | Clean processed interface records |
| **Check Credit** | Manually re-check customer credit |
| **Schedule Orders** | Run ATP for unscheduled lines |

---

## 7. OMS Table Relationship Quick Reference

```sql
-- Complete order-to-invoice query
SELECT ooh.order_number,
       ooh.flow_status_code AS order_status,
       ool.line_number,
       ool.ordered_item,
       ool.ordered_quantity,
       ool.shipped_quantity,
       ool.flow_status_code AS line_status,
       rct.trx_number AS invoice_number,
       rct.trx_date AS invoice_date,
       rctla.unit_selling_price AS invoice_price,
       rctla.quantity_ordered AS invoice_qty
FROM oe_order_headers_all ooh
JOIN oe_order_lines_all ool ON ooh.header_id = ool.header_id
LEFT JOIN ra_customer_trx_lines_all rctla
  ON rctla.interface_line_attribute6 = TO_CHAR(ool.line_id)
  AND rctla.interface_line_context = 'ORDER ENTRY'
LEFT JOIN ra_customer_trx_all rct
  ON rctla.customer_trx_id = rct.customer_trx_id
WHERE ooh.order_number = '10001'
ORDER BY ool.line_number;
```

---

## 8. Common OM Tables - Quick Reference Card

```sql
-- Order Management
OE_ORDER_HEADERS_ALL     -> Order header
OE_ORDER_LINES_ALL       -> Order lines
OE_TRANSACTION_TYPES_ALL -> Order types
OE_ORDER_HOLDS_ALL       -> Holds on orders
OE_PRICE_ADJUSTMENTS     -> Discounts applied

-- Customers (TCA)
HZ_PARTIES               -> Party master (org/person)
HZ_CUST_ACCOUNTS         -> Customer accounts
HZ_CUST_ACCT_SITES_ALL   -> Customer address sites
HZ_CUST_SITE_USES_ALL    -> BILL_TO, SHIP_TO designations

-- Shipping
WSH_DELIVERY_DETAILS     -> Items to ship
WSH_NEW_DELIVERIES       -> Delivery header
WSH_DELIVERY_ASSIGNMENTS -> Links DD to delivery

-- AR
RA_CUSTOMER_TRX_ALL      -> Invoice headers
RA_CUSTOMER_TRX_LINES_ALL -> Invoice lines
RA_CUST_TRX_TYPES_ALL    -> Transaction types (INV, CM, DM)
AR_PAYMENT_SCHEDULES_ALL  -> Amount due / payment schedule

-- Inventory
MTL_SYSTEM_ITEMS_B       -> Item master
MTL_ONHAND_QUANTITIES    -> Current stock
MTL_MATERIAL_TRANSACTIONS -> Inventory movements
MTL_TRANSACTION_TYPES    -> Type of movement
```

---

## 🎯 Top 10 Interview Questions - Final Review

**Q1: End-to-end: What happens after an order is booked in OMS?**
> Booking -> ATP Scheduling -> Workflow fires -> AWAITING_SHIPPING -> Pick Release (Move Order) -> Pick Confirm (inventory staged) -> Ship Confirm (inventory decremented, AR Interface created) -> AutoInvoice creates Invoice -> GL journal posted.

**Q2: What is the RA_INTERFACE_LINES_ALL table?**
> Staging table where ship-confirmed order lines are written before AutoInvoice converts them to actual AR invoices. If AutoInvoice fails, errors are in RA_INTERFACE_ERRORS.

**Q3: What causes an AutoInvoice error?**
> Common causes: invalid customer/bill-to site, missing payment terms, invalid GL revenue account, line type mismatch, missing sales rep. Check RA_INTERFACE_ERRORS for specific message.

**Q4: How do you link an invoice back to the original order?**
> `RA_CUSTOMER_TRX_LINES_ALL.interface_line_attribute1` = order number, `attribute6` = line ID. Context = 'ORDER ENTRY'. Join on these fields to get order line -> invoice line relationship.

**Q5: What is Interface Trip Stop?**
> A concurrent program that sends Ship Confirm information from Shipping to Order Management, updating order line status to SHIPPED and creating the AR interface record.

**Q6: What key tables would you check to troubleshoot an order not invoiced?**
> OE_ORDER_HEADERS_ALL (order status), WSH_DELIVERY_DETAILS (shipped?), RA_INTERFACE_LINES_ALL (interface record exists?), RA_INTERFACE_ERRORS (why invoice failed).

---

## 📝 Final Summary - OMS Integration Points

```
OMS -> WSH:
  Book -> WSH_DELIVERY_DETAILS created (released_status='R')
  Ship Confirm -> Order line = SHIPPED, INV decremented

WSH -> AR:
  Ship Confirm -> RA_INTERFACE_LINES_ALL (staging)
  AutoInvoice -> RA_CUSTOMER_TRX_ALL (invoice)

OMS -> INV:
  Ship Confirm -> MTL_MATERIAL_TRANSACTIONS
  DR: COGS, CR: Inventory Asset

AR -> GL:
  AutoInvoice -> GL Journal
  DR: AR, CR: Revenue

Key troubleshooting:
  Stuck in AWAITING_SHIPPING -> check hold, schedule, inventory
  No invoice -> check RA_INTERFACE_LINES, run Interface Trip Stop
  AutoInvoice error -> check RA_INTERFACE_ERRORS
  Workflow error -> check WF_ITEM_ACTIVITY_STATUSES

Must-run programs:
  Interface Trip Stop -> ship confirm to OM
  Inventory Interface -> INV transactions
  AutoInvoice -> staging to AR invoice
  Workflow Background -> process deferred activities
```

---

*🎉 Congratulations! You have completed all 25 chapters.*
*Review the [README](README.md) for the quick reference cheatsheet before your interview.*
