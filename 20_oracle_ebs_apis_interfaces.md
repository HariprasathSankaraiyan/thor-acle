# Chapter 20: Oracle EBS - APIs, Open Interfaces & RICE Components
## ⏱️ ~20 min read

---

## 🧠 Think of it this way

> **Open Interface** = A **loading dock** at a warehouse.
> External systems drop data into staging tables (like unloading trucks at the dock).
> Oracle's import program picks up the staged data, validates it, and loads it into production tables.
>
> **API (Application Program Interface)** = A **valet parking attendant**.
> You hand your keys (data) directly to the valet (API). They handle parking (validation + loading) correctly without you touching the car.
>
> **RICE** = The 5 categories of EBS customization work:
> Reports, Interfaces, Conversions, Extensions, Workflows.

---

## 1. The RICE Framework

| Letter | Component | What it is |
|--------|-----------|-----------|
| **R** | Reports | Custom reports - SQL, XML Publisher, Oracle Reports |
| **I** | Interfaces | Data exchange with external systems (in/out) |
| **C** | Conversions | One-time data migration from legacy systems |
| **E** | Extensions | Custom code extending standard functionality |
| **W** | Workflows | Custom or extended workflow processes |

> In interviews, you'll always be asked about RICE experience. Know each one with an example.

---

## 2. Interfaces - Inbound vs Outbound

### Inbound Interface
> External system -> Staging tables -> Oracle Import Program -> EBS tables

```
Legacy System
     │
     ▼
Staging Table (e.g., OE_HEADERS_IFACE_ALL)
     │
     ▼
Open Interface Concurrent Program (e.g., Order Import)
     │
     ▼ (if valid)
EBS Production Table (e.g., OE_ORDER_HEADERS_ALL)
     │
     ▼ (if error)
Error Table / FND_INTERFACE_CONTROL (error flag + message)
```

### Outbound Interface
> EBS tables -> Extract program -> File/Staging -> External system

---

## 3. Oracle Order Management Open Interface

### Key Staging Tables
```sql
OE_HEADERS_IFACE_ALL       -- Order header interface staging
OE_LINES_IFACE_ALL         -- Order lines interface staging
OE_ACTIONS_IFACE_ALL       -- Order actions (holds, etc.)
OE_PRICE_ADJS_IFACE_ALL    -- Price adjustments
OE_LOTSERIALS_IFACE_ALL    -- Lot/serial numbers
```

### How to Import Orders (Interface)
```sql
-- Step 1: Insert into staging tables
INSERT INTO oe_headers_iface_all (
  order_source_id, orig_sys_document_ref,
  order_type_id, sold_to_org_id, ship_to_org_id,
  price_list_id, pricing_date,
  request_date, org_id,
  operation_code
) VALUES (
  1001,           -- Order source ID
  'EXT-ORD-001',  -- External order number (unique)
  1234,           -- Order type ID
  5678,           -- Customer ID (sold-to)
  5678,           -- Ship-to address ID
  100,            -- Price list ID
  SYSDATE,
  SYSDATE + 7,    -- Requested delivery date
  101,            -- Org ID
  'INSERT'        -- CREATE/UPDATE/DELETE
);

-- Step 2: Submit Order Import concurrent program
-- Program: Order Import (OEXOEORD)
-- Param: Order Source = 1001, Original Reference = 'EXT-ORD-001'

-- Step 3: Check results
SELECT ohai.orig_sys_document_ref,
       ohai.error_flag,
       ohai.status_flag,  -- 'P'=pending, 'I'=imported, 'E'=error
       ohai.message_text
FROM oe_headers_iface_all ohai
WHERE ohai.orig_sys_document_ref = 'EXT-ORD-001';
```

---

## 4. Oracle Process Order API - Direct API Approach

> Better than open interface for real-time, transactional use.

```sql
DECLARE
  l_header_rec         OE_ORDER_PUB.HEADER_REC_TYPE;
  l_line_tbl           OE_ORDER_PUB.LINE_TBL_TYPE;
  l_action_request_tbl OE_ORDER_PUB.REQUEST_TBL_TYPE;
  l_header_val_rec     OE_ORDER_PUB.HEADER_VAL_REC_TYPE;
  l_line_val_tbl       OE_ORDER_PUB.LINE_VAL_TBL_TYPE;
  l_header_adj_tbl     OE_ORDER_PUB.HEADER_ADJ_TBL_TYPE;
  l_line_adj_tbl       OE_ORDER_PUB.LINE_ADJ_TBL_TYPE;
  x_header_rec         OE_ORDER_PUB.HEADER_REC_TYPE;
  x_line_tbl           OE_ORDER_PUB.LINE_TBL_TYPE;
  x_return_status      VARCHAR2(1);
  x_msg_count          NUMBER;
  x_msg_data           VARCHAR2(2000);
BEGIN
  -- Initialize missing for required fields
  l_header_rec := OE_ORDER_PUB.G_MISS_HEADER_REC;

  -- Set header values
  l_header_rec.operation       := OE_GLOBALS.G_OPR_CREATE;
  l_header_rec.order_type_id   := 1234;
  l_header_rec.sold_to_org_id  := 5678;
  l_header_rec.price_list_id   := 100;
  l_header_rec.ordered_date    := SYSDATE;

  -- Initialize org context
  FND_GLOBAL.APPS_INITIALIZE(
    user_id       => 1234,
    resp_id       => 5678,
    resp_appl_id  => 660   -- OM application ID
  );
  MO_GLOBAL.SET_POLICY_CONTEXT('S', 101);

  -- Call the API
  OE_ORDER_PUB.PROCESS_ORDER(
    p_api_version_number => 1.0,
    p_header_rec         => l_header_rec,
    p_line_tbl           => l_line_tbl,
    p_action_request_tbl => l_action_request_tbl,
    x_header_rec         => x_header_rec,
    x_line_tbl           => x_line_tbl,
    x_return_status      => x_return_status,
    x_msg_count          => x_msg_count,
    x_msg_data           => x_msg_data
  );

  IF x_return_status = FND_API.G_RET_STS_SUCCESS THEN
    DBMS_OUTPUT.PUT_LINE('Order created: ' || x_header_rec.header_id);
    COMMIT;
  ELSE
    -- Get error messages
    FOR i IN 1..x_msg_count LOOP
      DBMS_OUTPUT.PUT_LINE(FND_MSG_PUB.GET(i, 'F'));
    END LOOP;
    ROLLBACK;
  END IF;
END;
```

---

## 5. Customer Interface (TCA - Trading Community Architecture)

```sql
-- Create a new Customer (Account)
DECLARE
  l_create_update_flag  VARCHAR2(1) := 'C';
  x_return_status       VARCHAR2(1);
  x_msg_count           NUMBER;
  x_msg_data            VARCHAR2(2000);
  l_party_rec           HZ_PARTY_V2PUB.PARTY_REC_TYPE;
  x_party_id            NUMBER;
  x_party_number        VARCHAR2(30);
  x_profile_id          NUMBER;
BEGIN
  l_party_rec.party_type := 'ORGANIZATION';
  l_party_rec.party_name := 'New Customer Ltd';

  HZ_PARTY_V2PUB.CREATE_ORGANIZATION(
    p_organization_rec => l_party_rec,
    x_return_status    => x_return_status,
    x_msg_count        => x_msg_count,
    x_msg_data         => x_msg_data,
    x_party_id         => x_party_id,
    x_party_number     => x_party_number,
    x_profile_id       => x_profile_id
  );

  IF x_return_status = 'S' THEN
    DBMS_OUTPUT.PUT_LINE('Party created: ' || x_party_id);
  END IF;
END;
```

---

## 6. Common Oracle EBS APIs - Know These

| Module | API Package | Use |
|--------|------------|-----|
| Order Management | `OE_ORDER_PUB` | Create/update sales orders |
| Customers (TCA) | `HZ_PARTY_V2PUB`, `HZ_CUST_ACCOUNT_V2PUB` | Create customers |
| Inventory | `MTL_ONLINE_TRANSACTION_PUB` | Inventory transactions |
| Accounts Payable | `AP_INVOICES_PKG`, `AP_IMPORT_INVOICES_PKG` | Create invoices |
| GL Journal | `GL_JOURNAL_IMPORT_PKG` | Import journal entries |
| PO | `PO_HEADERS_API` | Create purchase orders |
| Concurrent | `FND_REQUEST.SUBMIT_REQUEST` | Submit programs |
| Workflow | `WF_ENGINE` | Workflow operations |

---

## 7. Standard API Pattern (All Oracle APIs follow this)

```sql
-- Standard Oracle API call pattern
API_PACKAGE.API_PROCEDURE(
  -- Standard IN parameters
  p_api_version_number  => 1.0,
  p_init_msg_list       => FND_API.G_TRUE,   -- Clear message stack
  p_commit              => FND_API.G_FALSE,   -- Don't auto-commit
  p_validation_level    => FND_API.G_VALID_LEVEL_FULL,

  -- Application-specific IN parameters
  p_xyz                 => l_value,

  -- Standard OUT parameters
  x_return_status       => x_return_status,  -- 'S'=Success, 'E'=Error, 'U'=Unexpected
  x_msg_count           => x_msg_count,
  x_msg_data            => x_msg_data
);

-- Check return status
IF x_return_status = FND_API.G_RET_STS_SUCCESS THEN  -- 'S'
  COMMIT;
ELSIF x_return_status = FND_API.G_RET_STS_ERROR THEN -- 'E'
  -- Business rule violation
ELSE  -- 'U' = unexpected
  -- System error
END IF;

-- Read error messages
FOR i IN 1..x_msg_count LOOP
  l_msg := FND_MSG_PUB.GET(i, FND_API.G_FALSE);
  DBMS_OUTPUT.PUT_LINE(l_msg);
END LOOP;
```

---

## 8. Conversions - One-Time Data Migration

```
Legacy System Data
      │
      ▼
Transformation (map fields, cleanse, enrich)
      │
      ▼
Staging Tables (custom or EBS interface tables)
      │
      ▼
Validation (check mandatory fields, lookups, FKs)
      │
      ▼ Pass          ▼ Fail
Production Tables  Error Report
```

### Conversion Checklist
1. Data profiling (understand source data quality)
2. Mapping document (source column -> target column)
3. Transformation rules
4. Staging table design
5. Validation queries
6. Migration script (SQL/PL/SQL)
7. Reconciliation report (counts, amounts)
8. UAT sign-off
9. Cutover plan

---

## 9. Outbound Interface Example (Extract to File)

```sql
-- Extract open orders and write to file for external system
CREATE OR REPLACE PROCEDURE extract_open_orders (
  errbuf  OUT VARCHAR2,
  retcode OUT NUMBER,
  p_org_id IN NUMBER
) AS
BEGIN
  FND_FILE.PUT_LINE(FND_FILE.OUTPUT, 'ORDER_NUMBER|CUSTOMER|AMOUNT|STATUS');

  FOR r IN (
    SELECT ooh.order_number,
           hca.account_name,
           ooh.ordered_amount,
           ooh.flow_status_code
    FROM oe_order_headers_all ooh
    JOIN hz_cust_accounts hca ON ooh.sold_to_org_id = hca.cust_account_id
    WHERE ooh.flow_status_code = 'BOOKED'
    AND ooh.org_id = p_org_id
  ) LOOP
    FND_FILE.PUT_LINE(
      FND_FILE.OUTPUT,
      r.order_number || '|' || r.account_name || '|' ||
      r.ordered_amount || '|' || r.flow_status_code
    );
  END LOOP;

  retcode := 0;
  errbuf := 'Extract complete';
END;
```

---

## 10. Extensions - Customizations

### Types of Extensions
1. **Custom Packages/Procedures**: Business logic not in standard product
2. **Custom Tables**: Additional data storage
3. **Custom Forms / OAF Pages**: Additional UI
4. **Database Triggers on Standard Tables**: Risky - may break upgrades
5. **Business Events / Hooks**: Preferred - Oracle provides hooks for customization

### Good Extension Pattern (Using Standard Hooks)
```sql
-- OM uses Oracle Workflow for order processing
-- Extend by adding custom function activity to workflow
-- Don't touch the base tables directly via trigger

-- Use database triggers only on CUSTOM tables, not EBS tables
-- Preferred: Oracle provides CUSTOM.pll, Personalization, Workflow extensions
```

---

## 🎯 Top Interview Questions & Answers

**Q1: What is the difference between an API and an Open Interface?**
> API (Application Program Interface) calls a PL/SQL package directly - real-time, immediate validation, immediate result. Open Interface uses staging tables - data is staged first, then an import program processes it in batch. APIs are better for real-time; interfaces for batch loads.

**Q2: What is RICE?**
> Reports (custom reporting), Interfaces (data exchange with external systems), Conversions (one-time data migration), Extensions (custom functionality), Workflows (process automation). Standard way to categorize Oracle EBS customization work.

**Q3: How do you import orders into Oracle EBS?**
> Two ways: 1) Open Interface - insert into OE_HEADERS_IFACE_ALL / OE_LINES_IFACE_ALL, then run "Order Import" concurrent program. 2) API - call OE_ORDER_PUB.PROCESS_ORDER directly with the order data.

**Q4: What does FND_GLOBAL.APPS_INITIALIZE do?**
> Sets the security context for an EBS API call - establishes which user, responsibility, and application the API is running as. Required before calling most EBS APIs to set org context and security.

**Q5: What is the standard return status convention in Oracle APIs?**
> x_return_status = 'S' for success, 'E' for business error, 'U' for unexpected (system) error. Always check this after calling an API. Read error messages from FND_MSG_PUB.GET().

**Q6: What is TCA (Trading Community Architecture)?**
> Oracle's master data model for parties (customers, suppliers, contacts). Central tables: HZ_PARTIES (the entity), HZ_CUST_ACCOUNTS (the business relationship), HZ_LOCATIONS (address). All customer/supplier data flows through TCA.

---

## 📝 Quick Revision Summary

```
RICE:
  R = Reports (XML Publisher, Oracle Reports)
  I = Interfaces (staging tables -> import program)
  C = Conversions (legacy migration)
  E = Extensions (custom packages, custom tables)
  W = Workflows (process automation)

Open Interface Pattern:
  Insert into staging -> Run import program -> Check error_flag
  OM: OE_HEADERS_IFACE_ALL -> Order Import program

API Pattern:
  FND_GLOBAL.APPS_INITIALIZE(user_id, resp_id, resp_appl_id)
  MO_GLOBAL.SET_POLICY_CONTEXT('S', org_id)
  API_PKG.API_PROC(p_api_version=1.0, ..., x_return_status=>l_status)
  Check: 'S'=success, 'E'=error, 'U'=unexpected
  Errors: FND_MSG_PUB.GET(i, 'F')

Key APIs:
  OE_ORDER_PUB.PROCESS_ORDER  -> create/update orders
  HZ_PARTY_V2PUB              -> create customers
  FND_REQUEST.SUBMIT_REQUEST  -> run concurrent programs
```

---

*Next: [Chapter 21 - OMS Order Life Cycle](21_oms_order_cycle.md)*
