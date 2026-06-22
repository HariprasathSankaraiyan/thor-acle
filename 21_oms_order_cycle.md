# Chapter 21: OMS Functional - Order Life Cycle & Order Types
## ⏱️ ~15 min read

---

## 🧠 Think of it this way

> An Order in Oracle OMS is like **ordering from a restaurant**:
> 1. You place the order (Entered)
> 2. Restaurant confirms (Booked)
> 3. Kitchen starts preparing (Awaiting Shipping)
> 4. Food is picked/packed (Picked, Packed)
> 5. Waiter brings it to you (Shipped)
> 6. You're charged (Invoiced)
> 7. Transaction is done (Closed)
>
> At any step, someone could hold the order, cancel it, or change it.

---

## 1. Order Life Cycle - The Full Flow

```
                    ENTERED
                       │
            (Book Order button / auto-book)
                       │
                   BOOKED
                       │
              (Scheduling/ATP check)
                       │
              AWAITING SCHEDULING
                       │
                (Scheduled date set)
                       │
            AWAITING SHIPPING
           (flow_status_code = 'AWAITING_SHIPPING')
                       │
            ┌──────────────────────────┐
            │      WAREHOUSE STEPS     │
            │  Pick Release ->          │
            │  Pick Confirm ->          │
            │  Pack (optional) ->       │
            │  Ship Confirm ->          │
            └──────────────────────────┘
                       │
                   SHIPPED
                       │
               (AutoInvoice runs)
                       │
                  INVOICED
                       │
                   CLOSED
```

---

## 2. Key Status Fields

Oracle OMS uses two status fields on order lines:

```sql
-- OE_ORDER_LINES_ALL
flow_status_code       -- Overall line status (most important)
schedule_status_code   -- Scheduling status
shipping_interfaced_flag -- 'Y' when sent to shipping
fulfillment_date       -- When line was fulfilled

-- OE_ORDER_HEADERS_ALL  
flow_status_code       -- Header status (rolls up from lines)
booked_flag            -- 'Y' when order is booked
open_flag              -- 'Y' while order is active
```

### flow_status_code Values (Line Level)
| Status | Meaning |
|--------|---------|
| `ENTERED` | Saved but not booked |
| `BOOKED` | Confirmed, ready for processing |
| `AWAITING_SHIPPING` | Ready to ship, in warehouse queue |
| `PICKED` | Pick released |
| `PACKED` | Items packed |
| `SHIPPED` | Ship confirmed |
| `INVOICED` | Invoice created (AutoInvoice run) |
| `CLOSED` | Fully complete |
| `CANCELLED` | Cancelled (full or partial) |
| `AWAITING_SCHEDULING` | Waiting for delivery date |

---

## 3. Order Entry Details

### Required Fields for an Order
- Order Type (determines defaults and processing)
- Sold-to Customer
- Price List
- Ordered Date
- Request Date (customer's requested delivery date)
- Order Lines with Item, Quantity, Unit Price

### Order Header Key Tables
```sql
OE_ORDER_HEADERS_ALL      -- Order header
OE_ORDER_LINES_ALL        -- Order lines
OE_ORDER_SOURCES          -- Order source definition
```

### Useful Query
```sql
-- Get order with details
SELECT ooh.order_number,
       ooh.flow_status_code AS header_status,
       ooh.booked_flag,
       ooh.ordered_date,
       ooh.org_id,
       hca.account_name    AS customer,
       ooh.ordered_amount
FROM oe_order_headers_all ooh
JOIN hz_cust_accounts hca ON ooh.sold_to_org_id = hca.cust_account_id
WHERE ooh.order_number = 10001;

-- Get order lines
SELECT ool.line_number,
       ool.ordered_item,
       ool.ordered_quantity,
       ool.unit_selling_price,
       ool.flow_status_code,
       ool.schedule_ship_date,
       ool.shipped_quantity
FROM oe_order_lines_all ool
WHERE ool.header_id = (SELECT header_id FROM oe_order_headers_all WHERE order_number = 10001)
ORDER BY ool.line_number;
```

---

## 4. Booking an Order

> **Booking** = formal confirmation. Order becomes binding.

What happens during booking:
1. **Validation**: Mandatory fields checked, pricing applied
2. **Credit Check**: If credit check enabled, customer's credit is verified
3. **Workflow start**: Order booking workflow fires
4. **Status change**: `ENTERED` -> `BOOKED`
5. **Scheduling**: ATP (Available-to-Promise) check runs
6. **Holds check**: If any active hold, booking may be prevented

### Book via UI
```
Sales Order form -> Actions -> Book Order
OR
Check "Book Order" checkbox on Order header form
```

### Book via API
```sql
-- Book an order programmatically
l_request_rec.entity_code := 'HEADER';
l_request_rec.entity_id   := l_header_id;
l_request_rec.request_type := OE_GLOBALS.G_BOOK_ORDER;
l_action_request_tbl(1)   := l_request_rec;

OE_ORDER_PUB.PROCESS_ORDER(
  p_action_request_tbl => l_action_request_tbl,
  ...
);
```

---

## 5. Order Types (Transaction Types)

> Templates that set defaults for an order. Like order "categories" - "Standard Sale", "Rush Order", "Web Order", "Sample Order".

### What Order Type Controls
- **Order Category**: ORDER or RETURN
- **Workflow assignment**: Which workflow process to run
- **Scheduling rules**: Auto or manual
- **Shipping method defaults**
- **Invoice source**: Which AR transaction type to use
- **Price list default**
- **Credit check rules**

### Key Tables
```sql
OE_TRANSACTION_TYPES_ALL       -- Transaction type definitions
OE_TRANSACTION_TYPES_TL        -- Translated names
OE_WORKFLOW_ASSIGNMENTS        -- Workflow assigned to order types
```

```sql
-- Query order types
SELECT ott.transaction_type_code,
       ott.order_category_code,  -- 'ORDER' or 'RETURN'
       ott.name,
       ott.description
FROM oe_transaction_types_tl ott
WHERE ott.language = USERENV('LANG')
ORDER BY ott.name;
```

---

## 6. Order Line Types

- Each line on an order also has a **Line Type** (part of the Order Type definition)
- Controls how the LINE is processed - standard, service, kit, config
- Different line types can have different workflows

---

## 7. ATP - Available-to-Promise

> "Can we actually deliver this item by the date the customer wants?"
> Oracle checks inventory on-hand + incoming supply - existing demand.

```
Customer wants: 100 units of Item X by Jan 15

Oracle ATP checks:
  On-hand inventory: 60 units
  Incoming PO:       50 units arriving Jan 10
  Existing demand:   30 units already promised
  
  Available Jan 10: 60 + 50 - 30 = 80 units
  Can meet 80 (not 100) by Jan 15
  Promise date for 100 units: Feb 1
```

### ATP Result in OMS
- Schedule Ship Date populated
- If ATP fails: `SCHEDULE_STATUS_CODE = 'EXCEPTION'`
- User can override or accept exception date

---

## 8. Scheduling

Types of scheduling:
- **Auto-Schedule**: Automatic when line is saved/booked
- **Manual Schedule**: User manually schedules
- **Demand Class**: Priority class for scheduling
- **Request Date**: Customer's requested date
- **Schedule Ship Date**: Date Oracle promises to ship

```sql
-- Lines awaiting scheduling
SELECT ool.order_number, ool.ordered_item,
       ool.ordered_quantity,
       ool.request_date,
       ool.schedule_ship_date,
       ool.schedule_status_code
FROM oe_order_lines_all ool
JOIN oe_order_headers_all ooh ON ool.header_id = ooh.header_id
WHERE ool.schedule_status_code IN ('EXCEPTION', NULL)
AND ool.flow_status_code = 'AWAITING_SCHEDULING';
```

---

## 9. Order Numbering

```sql
-- Order numbers come from a sequence (defined per order type or globally)
-- Check the sequence used
SELECT sequence_name FROM oe_order_types_v WHERE ORDER_TYPE = 'Standard';

-- Configure: System Parameters -> Order Numbering
-- Can be: Automatic (sequence), Manual, or Document Sequence
```

---

## 10. Multi-Line Orders - Kits and Configurations

### Model / Kit Lines
- **Model**: BOM (Bill of Materials) component - one parent line, multiple child lines
- **PTO (Pick to Order)**: Options chosen at order time
- **ATO (Assemble to Order)**: Manufactured specifically for this order
- **Standard Item**: Regular stocked item

```sql
-- Find parent-child line relationships
SELECT ool.line_number, ool.ordered_item, ool.item_type_code,
       ool.top_model_line_id, ool.ato_line_id
FROM oe_order_lines_all ool
WHERE ool.header_id = :header_id
ORDER BY ool.line_number;
```

---

## 🎯 Top Interview Questions & Answers

**Q1: What is the order life cycle in Oracle OMS?**
> Entered -> Booked -> Awaiting Scheduling -> Awaiting Shipping -> Picked -> Packed -> Shipped -> Invoiced -> Closed. Status tracked in `flow_status_code` on OE_ORDER_HEADERS_ALL and OE_ORDER_LINES_ALL.

**Q2: What happens when an order is booked?**
> Validation runs, pricing finalizes, credit check runs (if enabled), booking workflow fires, status changes to BOOKED, ATP scheduling runs, order appears in warehouse pick queue.

**Q3: What is an Order Type?**
> A template/category that defines defaults and processing rules for an order - workflow, pricing, credit check, shipping defaults, and which AR transaction type to use for invoicing.

**Q4: What is ATP?**
> Available-to-Promise. A check that calculates if inventory can meet the customer's requested delivery date considering on-hand stock, incoming supply, and existing demand commitments.

**Q5: What is the difference between flow_status_code at header vs line level?**
> Line level shows individual item status (AWAITING_SHIPPING, SHIPPED, etc.). Header status rolls up - header is BOOKED when at least one line is booked, CLOSED when all lines are closed.

**Q6: What is the difference between Request Date and Schedule Ship Date?**
> Request Date is when the customer WANTS delivery. Schedule Ship Date is when Oracle PROMISES to ship. If ATP finds a gap, Schedule Ship Date may be later than Request Date.

---

## 📝 Quick Revision Summary

```
Order Life Cycle:
  ENTERED -> BOOKED -> AWAITING_SHIPPING -> PICKED ->
  PACKED -> SHIPPED -> INVOICED -> CLOSED

Key tables:
  OE_ORDER_HEADERS_ALL -> header status, customer, amount
  OE_ORDER_LINES_ALL   -> line status, item, qty, price

Booking: validates + schedules + starts workflow
  Credit check -> Hold if failed
  ATP -> Schedule Ship Date set

Order Type:
  = Template for order processing
  Controls: workflow, pricing, credit check, AR invoice type

ATP = "Can we deliver by requested date?"
  Request Date = customer wants
  Schedule Ship Date = Oracle promises

Status: flow_status_code (most important field)
```

---

*Next: [Chapter 22 - OMS Pricing & Tax](22_oms_pricing_tax.md)*
