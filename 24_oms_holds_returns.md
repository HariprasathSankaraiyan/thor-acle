# Chapter 24: OMS Functional - Order Holds, Returns (RMA) & Credits
## ⏱️ ~20 min read

---

## 🧠 Think of it this way

> **Hold** = A red flag on an order. "Don't ship this until we resolve something."
> Like a bank putting a hold on your credit card - you can't use it until the issue is cleared.
>
> **RMA (Return Merchandise Authorization)** = A customer wants to return something.
> Like returning a shirt to a store - you get a return number, bring the item back, and get a refund or exchange.
>
> **Credit Memo** = A refund or credit note. "We owe you money" note from seller to buyer.

---

## PART A: ORDER HOLDS

## 1. What is a Hold?

- A **hold** prevents an order or order line from progressing through the order cycle
- Holds can be on: Header level (entire order) or Line level (specific item)
- Order cannot be booked, cannot be shipped while on hold (depending on hold type)
- Must be **released** before order can proceed

---

## 2. Hold Types

| Hold Type | Who applies | When triggered |
|-----------|------------|----------------|
| **Credit Hold** | Automatic | Customer exceeds credit limit |
| **Manual Hold** | User manually | Any business reason |
| **Booking Hold** | Setup/automatic | Prevents booking specifically |
| **Shipping Hold** | Setup/automatic | Prevents shipping specifically |
| **Payment Hold** | Business logic | Payment issues |
| **Tax Hold** | Tax rules | Tax calculation issues |
| **Fulfillment Hold** | Workflow | Approval pending |

---

## 3. Hold Lifecycle

```
Order Entered/Booked
        │
        ▼
Hold Check (automatically or manually applied)
        │
        ▼
Order HELD
        │
Hold NOT released -> Order stays in current status
Hold RELEASED -> Order continues to next step
```

---

## 4. Credit Hold - Most Important

> **How Credit Hold works**:
1. Customer has a Credit Limit (set in customer account site)
2. When an order is booked, Oracle checks:
   - Current AR balance (unpaid invoices)
   + Open orders not yet invoiced
   vs. Credit Limit
3. If over limit -> **Automatic Credit Hold** applied
4. Collections team resolves -> removes hold
5. Order proceeds

```sql
-- Check customer credit hold status
SELECT hca.account_name, hca.account_number,
       hcp.credit_hold, hcp.credit_limit, hcp.overall_credit_limit,
       hcp.currency_code
FROM hz_cust_accounts hca
JOIN hz_customer_profiles hcp ON hca.cust_account_id = hcp.cust_account_id
WHERE hca.account_name LIKE '%ABC Corp%';

-- Check holds on an order
SELECT ooh.order_number, oeh.hold_id,
       ohe.name AS hold_name, oeh.hold_source_id,
       oeh.released_flag,
       oeh.hold_comment
FROM oe_order_headers_all ooh
JOIN oe_hold_sources_all ohsa ON ooh.header_id = ohsa.header_id
JOIN oe_order_holds_all oeh ON ohsa.hold_source_id = oeh.hold_source_id
JOIN oe_hold_definitions ohd ON oeh.hold_id = ohd.hold_id
WHERE ooh.order_number = '10001'
AND oeh.released_flag = 'N';
```

---

## 5. Key Hold Tables

```sql
OE_HOLD_DEFINITIONS           -- All hold types defined in system
OE_HOLD_SOURCES_ALL           -- Hold source (when/why applied)
OE_ORDER_HOLDS_ALL            -- Holds on specific orders/lines
```

---

## 6. Applying and Releasing Holds via API

```sql
-- Apply a hold
DECLARE
  l_hold_source_rec OE_HOLDS_PVT.HOLD_SOURCE_REC_TYPE;
  x_return_status   VARCHAR2(1);
  x_msg_count       NUMBER;
  x_msg_data        VARCHAR2(2000);
BEGIN
  l_hold_source_rec.hold_id          := 5;          -- Hold Definition ID
  l_hold_source_rec.hold_entity_code := 'O';         -- O=Order H=Order Header
  l_hold_source_rec.hold_entity_id   := 12345;        -- Header ID or Line ID
  l_hold_source_rec.responsibility_id := 50;

  OE_HOLDS_PUB.APPLY_HOLDS(
    p_api_version       => 1.0,
    p_hold_source_rec   => l_hold_source_rec,
    x_return_status     => x_return_status,
    x_msg_count         => x_msg_count,
    x_msg_data          => x_msg_data
  );
END;

-- Release a hold
OE_HOLDS_PUB.RELEASE_HOLDS(
  p_hold_source_rec   => l_hold_source_rec,
  p_release_reason_code => 'CREDIT_CLEARED',
  p_release_comments  => 'Customer paid outstanding balance',
  x_return_status     => x_return_status,
  x_msg_count         => x_msg_count,
  x_msg_data          => x_msg_data
);
```

---

## PART B: RETURNS (RMA)

## 7. What is an RMA?

- **RMA = Return Merchandise Authorization**
- An ORDER with Order Category = **RETURN**
- Customer returns goods; company gives refund or credit
- Triggers: inventory receipt back into warehouse AND credit memo in AR

---

## 8. RMA Process Flow

```
Customer requests return
        │
        ▼
Create RETURN Order (Order Type = Return)
  - Reference to original order (copy from)
  - Return reason
  - Return qty
        │
        ▼
BOOK the Return Order
        │
        ▼
Receive Returned Goods (RMA Receipt)
  WSH: Receive against the return order
  INV: Stock returned to inventory
        │
        ▼
Credit Memo Created (via AutoInvoice)
  AR: Credit Memo = negative invoice
        │
        ▼
Apply Credit Memo to Customer Account
  AR: Match credit memo to original invoice
  Result: Customer balance reduced
```

---

## 9. Creating a Return Order

### Via UI
```
Oracle Order Management -> Orders, Returns -> Sales Orders
-> Order Type: RETURN ORDER (or your return order type)
-> Copy From: Select original order to reference

OR:
-> Actions -> Copy (with Return)
-> Specify: Quantity, Return Reason, Reference
```

### Key Return Fields
```sql
OE_ORDER_HEADERS_ALL:
  order_category_code = 'RETURN'   -- This is a return order

OE_ORDER_LINES_ALL:
  line_category_code = 'RETURN'
  return_reason_code               -- Why returned
  reference_header_id              -- Original order header
  reference_line_id                -- Original order line
  reference_type = 'ORDER'
```

---

## 10. Return Line Types - What Happens After Receipt

| Line Type | After Receipt | What it does |
|-----------|--------------|-------------|
| **Credit Only** | No receipt needed | Just creates credit memo |
| **Receipt and Credit** | Receipt required first | Receive goods -> credit |
| **Receipt Only** | No credit | Just receives goods (exchange) |
| **Exchange** | Returns old, ships new | Full exchange scenario |

---

## 11. RMA Receipt - Returning Goods to Inventory

```sql
-- Check if return line is received
SELECT ool.line_number, ool.ordered_item,
       ool.ordered_quantity,
       ool.shipped_quantity,   -- For returns, this = received qty
       ool.flow_status_code,
       ool.return_reason_code
FROM oe_order_lines_all ool
WHERE ool.header_id = :return_header_id
AND ool.line_category_code = 'RETURN';
```

---

## 12. Credit Memos in AR

> After RMA is processed, AutoInvoice creates a **Credit Memo** in AR.

```sql
-- Find credit memos
SELECT rct.trx_number, rct.trx_date, rct.bill_to_customer_name,
       rct.type, rct.amount_due_remaining
FROM ra_customer_trx_all rct
WHERE rct.type = 'CM'   -- Credit Memo
AND rct.bill_to_customer_id = :customer_id
ORDER BY rct.trx_date DESC;

-- Find source return order for a credit memo
SELECT rctla.interface_line_attribute1 AS order_number
FROM ra_customer_trx_lines_all rctla
WHERE rctla.customer_trx_id = :credit_memo_trx_id
AND rctla.interface_line_context = 'ORDER ENTRY';
```

---

## 13. Hold vs Cancel - Common Confusion

| | Hold | Cancel |
|--|------|--------|
| Order status | Blocked/frozen | Permanently stopped |
| Reversible? | Yes (release hold) | No (can't uncancel) |
| Inventory | Not reserved to go | Nothing happens |
| Billing | Not invoiced | Not invoiced |
| Use case | Temporary block | Permanent stop |

### Cancel an Order
```sql
-- Cancel via API
l_request_rec.entity_code  := 'HEADER';
l_request_rec.entity_id    := l_header_id;
l_request_rec.request_type := OE_GLOBALS.G_CANCEL_LINE;
l_request_rec.param1       := 'CUSTOMER REQUEST';  -- cancel reason
```

---

## 14. Common Return/Hold Scenarios in Interviews

### Scenario 1: "Order is stuck, customer is asking for status"
-> Check `flow_status_code` on OE_ORDER_LINES_ALL
-> Check OE_ORDER_HOLDS_ALL for active holds (released_flag = 'N')
-> Check WSH_DELIVERY_DETAILS for shipping status

### Scenario 2: "Customer received wrong item, wants exchange"
-> Create Return Order with line type = EXCHANGE
-> Reference original order line
-> New sales order line (outbound) created automatically

### Scenario 3: "Credit hold is blocking customer's order even though they paid"
-> Check hz_customer_profiles for credit_hold flag
-> Verify if credit exposure recalculated
-> Run "Check Credit" manually
-> Release hold if credit is OK

---

## 🎯 Top Interview Questions & Answers

**Q1: What is a hold on an order? Name some types.**
> A hold prevents an order from advancing in the order cycle. Types: Credit Hold (auto, over credit limit), Manual Hold (user-applied), Booking Hold (prevents booking), Shipping Hold (prevents shipping).

**Q2: What is an RMA?**
> Return Merchandise Authorization - a return order with order_category_code = 'RETURN'. Allows customer to return goods. After receipt, creates a credit memo in AR to credit the customer.

**Q3: What is the difference between Credit Only and Receipt and Credit return?**
> Credit Only: creates credit memo without requiring physical return of goods. Receipt and Credit: physical goods must be received back into warehouse before credit memo is created.

**Q4: How does a credit hold get applied automatically?**
> When an order is booked, Oracle checks customer's total exposure (AR balance + open orders) against credit limit. If exceeded, credit hold is automatically applied via workflow, blocking shipping.

**Q5: What happens to inventory when an RMA is received?**
> Goods are received back into Oracle Inventory at the specified organization/locator. The inventory balance increases. A costing journal reverses the original COGS entry.

**Q6: What is the difference between Hold and Cancel?**
> Hold is temporary - order is frozen but can be released later. Cancel is permanent - order (or line) is stopped and cannot be reactivated. Hold preserves the order; Cancel ends it.

---

## 📝 Quick Revision Summary

```
Holds:
  OE_HOLD_DEFINITIONS   -> hold types
  OE_ORDER_HOLDS_ALL    -> holds on orders (released_flag='N' = active)
  Types: Credit Hold, Manual Hold, Shipping Hold, Booking Hold

Credit Hold:
  Triggered when AR balance + open orders > credit limit
  HZ_CUSTOMER_PROFILES.credit_hold = 'Y'
  Release when customer pays -> recalculate credit -> remove hold

RMA (Return):
  order_category_code = 'RETURN'
  line_category_code = 'RETURN'
  reference_header_id/line_id -> original order
  return_reason_code

Return Types:
  Credit Only   -> credit memo, no receipt needed
  Receipt+Credit -> receipt first, then credit memo
  Exchange      -> return + new outbound line

After RMA Receipt:
  1. Inventory received back
  2. AutoInvoice creates Credit Memo (type='CM')
  3. Applied to customer AR balance

Hold vs Cancel:
  Hold = temporary, reversible
  Cancel = permanent, not reversible
```

---

*Next: [Chapter 25 - OMS Integration & Troubleshooting](25_oms_integration_troubleshoot.md)*
