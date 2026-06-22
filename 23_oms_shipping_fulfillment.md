# Chapter 23: OMS Functional - Shipping, Pick/Pack/Ship, WSH
## ⏱️ ~15 min read

---

## 🧠 Think of it this way

> **Shipping** = the physical process of getting goods from warehouse to customer.
>
> Imagine an Amazon warehouse:
> 1. System generates a **pick list** (which items, which shelf, how many)
> 2. Warehouse worker **picks** items from shelf
> 3. Worker scans/confirms the pick (**Pick Confirm**)
> 4. Items go to **packing station** (Pack)
> 5. Package gets a label, goes on a truck (**Ship Confirm**)
> 6. Customer gets their order!
>
> **WSH** = Oracle Warehouse Management / Shipping Execution (the W-S-H schema).

---

## 1. Shipping Execution Architecture

```
Oracle Order Management (OMS/ONT)
        │
        │ Order Booked -> Line becomes AWAITING_SHIPPING
        │
        ▼
Oracle Shipping Execution (WSH Schema)
        │
        ├── Delivery Detail (one per order line quantity)
        ├── Delivery (groups delivery details for same shipment)
        └── Trip Stop (when/where truck stops)
        │
        ▼
Physical Warehouse (Pick -> Pack -> Ship)
        │
        ▼
Ship Confirm
        │
        ▼
Back to OMS: Order line -> SHIPPED
        │
        ▼
AutoInvoice -> AR Invoice created
```

---

## 2. Key Shipping Objects

### Delivery Detail
- Represents the **quantity to be shipped** for an order line
- One order line may have multiple delivery details (partial shipments, lot splits)

### Delivery
- Groups multiple **delivery details** going to the same ship-to, same trip
- Like: "All items going to Customer ABC on January 15 in one truck"
- Has: Ship-from, Ship-to, Carrier, Ship Method, Gross Weight

### Trip
- Represents a **vehicle/carrier trip**
- Has: multiple stops (pick-up and drop-off)

### Trip Stop
- Each location a trip visits (origin warehouse = pick-up stop, customer = drop-off stop)

---

## 3. Key Shipping Tables (WSH Schema)

```sql
WSH_DELIVERY_DETAILS      -- One row per order line qty (the item-to-ship)
WSH_NEW_DELIVERIES        -- Delivery header
WSH_DELIVERY_ASSIGNMENTS  -- Links delivery details to deliveries
WSH_TRIPS                 -- Trip records
WSH_TRIP_STOPS            -- Trip stops (pick-up, drop-off)
WSH_CARRIERS              -- Carrier definitions (UPS, FedEx, etc.)
WSH_CARRIER_SERVICES      -- Service levels (Ground, Express, etc.)
WSH_SHIPPING_PARAMETERS   -- Shipping org parameters
```

```sql
-- Check delivery details for an order
SELECT wdd.delivery_detail_id,
       wdd.source_header_number,    -- Order number
       wdd.source_line_number,      -- Line number
       wdd.inventory_item_id,
       wdd.ordered_quantity,
       wdd.shipped_quantity,
       wdd.released_status,         -- Key status field
       wdd.move_order_line_id
FROM wsh_delivery_details wdd
WHERE wdd.source_header_number = '10001'
ORDER BY wdd.source_line_number;

-- released_status values:
-- 'N' = Not Ready for Release (too early)
-- 'R' = Ready to Release (in pick queue)
-- 'S' = Released to Warehouse (pick slip created)
-- 'Y' = Staged (pick confirmed)
-- 'C' = Shipped
-- 'D' = Cancelled
-- 'X' = Not Applicable

-- Check delivery
SELECT wnd.delivery_id, wnd.name AS delivery_name,
       wnd.status_code,
       wnd.ship_method_code,
       wnd.gross_weight, wnd.net_weight, wnd.weight_uom_code,
       wnd.actual_departure_date
FROM wsh_new_deliveries wnd
JOIN wsh_delivery_assignments wda ON wnd.delivery_id = wda.delivery_id
JOIN wsh_delivery_details wdd ON wda.delivery_detail_id = wdd.delivery_detail_id
WHERE wdd.source_header_number = '10001';
```

---

## 4. Pick Release

> **Pick Release** = telling the warehouse "go pick these items from the shelves".

### What it does:
1. Selects eligible order lines (`released_status = 'R'`)
2. Creates **Move Order** (internal transfer instruction for the warehouse)
3. Updates `released_status` from `R` -> `S` (Released to Warehouse)

### Pick Release via UI
```
Oracle Shipping -> Transactions -> Pick Release
Criteria: Organization, Ship-to Zone, Order Number, Ship Date
-> Execute -> Pick Slip generated
```

### Pick Release Criteria
- Organization (warehouse)
- Order type
- Specific order number
- Customer
- Scheduled ship date range
- Carrier

---

## 5. Pick Confirm (Transact Move Orders)

> **Pick Confirm** = warehouse worker confirms items have been physically picked.

1. Worker scans item and quantity
2. System records inventory movement from storage to staging area
3. `released_status` changes: `S` -> `Y` (Staged/Pick Confirmed)
4. Inventory decremented from source locator

### Via UI
```
Oracle Inventory -> Transactions -> Transact Move Orders
OR
Oracle Shipping -> Transactions -> Quick Ship (combined pick+ship)
```

---

## 6. Auto-Pick vs Manual Pick

| Mode | Description |
|------|-------------|
| **Auto-Pick Confirm** | System automatically confirms pick (no worker scan) |
| **Manual Pick Confirm** | Warehouse worker confirms manually |
| **Pick and Ship** | Instant pick + ship confirm in one step |

> Setup: `Shipping Parameters` -> Autocreate Deliveries, Auto-Pack, Auto-Ship

---

## 7. Ship Confirm

> **Ship Confirm** = final confirmation that items have physically left the warehouse.

### What happens:
1. Mark delivery as shipped (actual departure date set)
2. `released_status` -> `C` (Complete/Shipped)
3. **Oracle Inventory decremented** (material leaves stock)
4. Order line `flow_status_code` -> `SHIPPED`
5. **Inventory Interface**: Cost of goods sold journal created
6. **AR Interface**: Transaction sent to AutoInvoice

### Ship Confirm via UI
```
Oracle Shipping -> Transactions -> Shipping Transactions
Find delivery -> Actions -> Confirm Delivery
Enter: Actual Departure Date, Carrier, Tracking Number
```

### Ship Confirm via API
```sql
WSH_DELIVERIES_PUB.CONFIRM_DELIVERY(
  p_delivery_id      => l_delivery_id,
  p_action_flag      => 'S',     -- S=Ship, B=Stage & Defer, T=Backorder
  p_intransit_flag   => 'Y',
  p_defer_interface_flag => 'N',
  x_return_status    => l_status
);
```

---

## 8. Backordering

> **Backorder** = ordered quantity > available quantity -> ship what you have now, backorder the rest.

```
Customer orders 100 units
Only 60 available to ship

Backorder:
  -> 60 units: Ship now -> Delivery Detail 1
  -> 40 units: Backordered -> New Delivery Detail 2 (future ship)
```

Types:
- **System Backorder**: Set at ship confirm if not enough quantity
- **Manual Backorder**: User manually backordering some lines

---

## 9. Shipping Parameters Setup

```
Oracle Shipping -> Setup -> Shipping Parameters
Key settings:
  - Default Organization
  - Autocreate Deliveries: Yes/No
  - Auto-Pack Ship: Yes/No
  - Document Set (what documents to print)
  - Pick to POD Sequence Numbers
```

---

## 10. Integration: OMS ↔ WSH ↔ INV ↔ AR

```
OMS (Order)
  │ Line BOOKED -> WSH_DELIVERY_DETAILS created (released_status='R')
  │
  ▼
WSH (Shipping)
  │ Pick Release -> Move Order created
  │ Pick Confirm -> Inventory staged
  │ Ship Confirm -> Inventory decremented
  │              -> Shipped qty written to OE_ORDER_LINES_ALL
  │              -> Costing interface to GL
  │
  ▼
AR (Accounts Receivable)
  │ AutoInvoice program picks up shipped lines
  │ Creates AR Invoice = Customer bill
  │
  ▼
GL (General Ledger)
    Journals posted:
    DR: COGS (Cost of Goods Sold)
    CR: Inventory Asset
    DR: AR Receivable
    CR: Revenue
```

---

## Key Queries for Shipping Troubleshooting

```sql
-- Find orders stuck in AWAITING_SHIPPING (not yet pick-released)
SELECT ooh.order_number, ool.ordered_item, ool.ordered_quantity,
       ool.schedule_ship_date, ool.flow_status_code
FROM oe_order_lines_all ool
JOIN oe_order_headers_all ooh ON ool.header_id = ooh.header_id
WHERE ool.flow_status_code = 'AWAITING_SHIPPING'
AND ool.schedule_ship_date < SYSDATE;

-- Find delivery details not yet shipped
SELECT wdd.source_header_number, wdd.source_line_number,
       wdd.released_status, wdd.ordered_quantity,
       wdd.shipped_quantity
FROM wsh_delivery_details wdd
WHERE wdd.released_status NOT IN ('C','D','X')
AND wdd.source_header_number = '10001';

-- Find ship confirm errors
SELECT wdd.source_header_number, wdd.interface_action_code,
       wdd.last_update_date
FROM wsh_delivery_details wdd
WHERE wdd.released_status = 'I'   -- Interface Error
ORDER BY wdd.last_update_date DESC;
```

---

## 🎯 Top Interview Questions & Answers

**Q1: What is the flow from order booking to shipping?**
> Booked -> AWAITING_SHIPPING -> Pick Release (Move Order created) -> Pick Confirm (inventory staged) -> Ship Confirm (inventory decremented, AR interface triggered, line = SHIPPED).

**Q2: What is a Delivery in Oracle Shipping?**
> A delivery groups multiple order lines going to the same destination in one shipment. It has attributes like ship-to address, carrier, ship method, weight. Ship Confirm is done at the delivery level.

**Q3: What is released_status on WSH_DELIVERY_DETAILS?**
> 'R' = Ready for release, 'S' = Released to warehouse (pick slip created), 'Y' = Staged (pick confirmed), 'C' = Shipped, 'D' = Cancelled.

**Q4: What triggers AR invoice creation?**
> Ship Confirm creates the AR interface record. The AutoInvoice concurrent program then picks up the interface records and creates actual AR invoices (RA_CUSTOMER_TRX_ALL).

**Q5: What is a Backorder?**
> When ship quantity < ordered quantity, the remaining quantity is backordered - it stays as a pending delivery detail for future fulfillment while the available quantity ships now.

**Q6: What are the key tables in Oracle Shipping?**
> WSH_DELIVERY_DETAILS (items to ship), WSH_NEW_DELIVERIES (delivery header), WSH_DELIVERY_ASSIGNMENTS (links details to delivery), WSH_TRIPS (carrier trips), WSH_TRIP_STOPS (pickup/dropoff).

---

## 📝 Quick Revision Summary

```
Shipping Flow:
  AWAITING_SHIPPING
  -> Pick Release: Move Order created, released_status 'R'->'S'
  -> Pick Confirm: Inventory staged, released_status 'S'->'Y'
  -> Ship Confirm: Inventory decremented, released_status 'Y'->'C'
  -> OMS line: SHIPPED
  -> AR Interface: AutoInvoice creates invoice

released_status: R=ready, S=released, Y=staged, C=shipped, D=cancelled

Key Tables:
  WSH_DELIVERY_DETAILS   -> the items (one per order line qty)
  WSH_NEW_DELIVERIES     -> the delivery (shipment)
  WSH_DELIVERY_ASSIGNMENTS -> link details to delivery
  WSH_TRIPS              -> carrier trips

Delivery = group of items going same place same carrier
Ship Confirm = at delivery level

Backorder = ship what's available, remaining stays pending
```

---

*Next: [Chapter 24 - OMS Holds & Returns (RMA)](24_oms_holds_returns.md)*
