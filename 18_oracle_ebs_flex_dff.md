# Chapter 18: Oracle EBS Flexfields - KFF & DFF
## ⏱️ ~15 min read

---

## 🧠 Think of it this way

> **Flexfields** = customizable columns in Oracle EBS without changing the base code.
>
> **Key Flexfield (KFF)** = your company's **account number structure**.
> Like a phone number format: Country Code - Area Code - Number - Extension
> Each segment means something specific. "01-HR-123-USA" tells you Division=01, Dept=HR, etc.
>
> **Descriptive Flexfield (DFF)** = **extra blank boxes** on a form you can label and use however you want.
> Like a form with "Additional Info" fields - your company decides what goes there.

---

## 1. What are Flexfields?

- **Extensibility mechanism** in Oracle EBS - add custom fields without modifying base code
- Stored in base tables as generic column names (`SEGMENT1`, `SEGMENT2`, ... for KFF; `ATTRIBUTE1`, `ATTRIBUTE2`, ... for DFF)
- Configuration stored in FND tables - no code changes

---

## 2. Key Flexfield (KFF) - The Structure Fields

> Used to **define key business identifiers** - things that need a hierarchical, segmented structure.

### Examples of KFFs in EBS
| KFF | Module | Example |
|-----|--------|---------|
| **Accounting Flexfield (COA)** | GL | `01-HR-5000-USD` (Company-Dept-Account-Currency) |
| **Item Flexfield** | INV | `LAPTOP-14IN-SILVER` (Category-Size-Color) |
| **Territory Flexfield** | AR | `US-WEST-CA` (Country-Region-State) |
| **System Items** | INV | Item number structure |
| **Grade Flexfield** | HR | Employee grade structure |
| **Location Flexfield** | FA | Asset location structure |

### KFF Structure
```
Structure (One KFF can have multiple structures):
  └── Segments (each segment = one part of the key)
        ├── Segment 1: Company (SEGMENT1) - from value set "Company Values"
        ├── Segment 2: Department (SEGMENT2) - from value set "Department Values"
        ├── Segment 3: Account (SEGMENT3) - from value set "Natural Accounts"
        └── Segment 4: Currency (SEGMENT4) - from value set "Currency Codes"
Delimiter: - (dash between segments)
Full combination: 01-HR-5000-USD
```

### KFF Tables
```sql
FND_ID_FLEXS                  -- KFF definitions
FND_ID_FLEX_STRUCTURES        -- Structures per KFF
FND_ID_FLEX_SEGMENTS          -- Segments per structure
FND_SEGMENT_ATTRIBUTE_VALUES  -- Segment attributes
FND_FLEX_VALUES               -- Values in value sets
FND_FLEX_VALUES_TL             -- Translated descriptions
```

### Querying KFF Data (Accounting Flexfield Example)
```sql
-- GL Code Combinations
SELECT gcc.code_combination_id,
       gcc.segment1 AS company,
       gcc.segment2 AS department,
       gcc.segment3 AS account,
       gcc.concatenated_segments  -- "01-HR-5000" pre-built
FROM gl_code_combinations gcc
WHERE gcc.chart_of_accounts_id = 101;

-- Using APPS.FND_FLEX_EXT package to build combinations
SELECT APPS.FND_FLEX_EXT.GET_SEGS(
  'SQLGL', 'GL#', 101, gcc.code_combination_id
) AS full_account
FROM gl_code_combinations gcc;
```

---

## 3. Value Sets - The Foundation of KFFs

> A value set is the list of valid options for each dropdown. Like a validation list.

### Value Set Types
| Type | Description |
|------|-------------|
| **None** | No validation - any value accepted |
| **Independent** | Fixed list of values (defined in FND_FLEX_VALUES) |
| **Dependent** | Values depend on another segment's value |
| **Table** | Values come from a DB table/view |
| **Special** | Custom validation using PL/SQL |
| **Pair** | Low-High value range |

```sql
-- View value sets
SELECT fvs.flex_value_set_name, fvs.validation_type,
       fvs.format_type, fvs.maximum_size
FROM fnd_flex_value_sets fvs
WHERE fvs.flex_value_set_name = 'GL_NATURAL_ACCOUNTS';

-- View values in a value set
SELECT flex_value, enabled_flag, description
FROM fnd_flex_values
WHERE flex_value_set_id = (
  SELECT flex_value_set_id
  FROM fnd_flex_value_sets
  WHERE flex_value_set_name = 'GL_NATURAL_ACCOUNTS'
);
```

---

## 4. Descriptive Flexfield (DFF) - The Extra Fields

> Used to **capture additional information** not in standard fields.
> Every EBS form has DFF columns (ATTRIBUTE1 to ATTRIBUTE30 typically).

### DFF Structure
```
DFF Definition (attached to a table/form):
  └── Context (optional - to group attributes)
        ├── Global Segments (always visible, no context required)
        │     ATTRIBUTE1 -> "Customer Priority"
        │     ATTRIBUTE2 -> "Contract Number"
        └── Context-Specific Segments (shown only when context matches)
              Context = "PREMIUM_CUSTOMER":
                ATTRIBUTE3 -> "Preferred Delivery"
                ATTRIBUTE4 -> "VIP Level"
              Context = "STANDARD_CUSTOMER":
                ATTRIBUTE3 -> "Standard SLA Days"
```

### DFF Tables
```sql
FND_DESCRIPTIVE_FLEXS              -- DFF definitions
FND_DESCR_FLEX_CONTEXTS            -- Context definitions
FND_DESCR_FLEX_COL_USAGE           -- Column/segment usage per context
```

### DFF in Data - Querying
```sql
-- OE_ORDER_HEADERS_ALL has DFF columns
SELECT order_id,
       attribute1 AS custom_field_1,
       attribute2 AS contract_number,
       attribute_category AS dff_context
FROM oe_order_headers_all
WHERE order_id = 1001;
```

### Common DFF Locations
| Table | DFF Purpose |
|-------|------------|
| `RA_CUSTOMERS` | Customer DFF |
| `OE_ORDER_HEADERS_ALL` | Sales Order Header DFF |
| `OE_ORDER_LINES_ALL` | Sales Order Line DFF |
| `MTL_SYSTEM_ITEMS_B` | Item DFF |
| `PO_HEADERS_ALL` | PO Header DFF |
| `AP_INVOICES_ALL` | Invoice DFF |

---

## 5. Setting Up a DFF (Step by Step)

```
System Administrator -> Application -> Flexfield -> Descriptive -> Register
  (if new table)

System Administrator -> Application -> Flexfield -> Descriptive -> Segments
  Application: Oracle Order Management
  Title: Sales Order Header Information (OE_ORDER_HEADERS)
  
  Define Global Segments:
    Name: Contract Number
    Column: ATTRIBUTE1
    Value Set: 30 Char Max (or custom)
    Prompt: Contract #
    
  Define Context:
    Context Field: ATTRIBUTE_CATEGORY
    Context Code: PREMIUM
    Segment: VIP Level
    Column: ATTRIBUTE2

Freeze & Compile -> Deploy
```

---

## 6. KFF vs DFF Summary

| | KFF | DFF |
|--|-----|-----|
| **Purpose** | Key/identifier structure | Extra descriptive fields |
| **Required?** | Yes (must have value) | Optional |
| **Segments** | SEGMENT1..SEGMENTn | ATTRIBUTE1..ATTRIBUTE30 |
| **Context column** | Not applicable | ATTRIBUTE_CATEGORY |
| **Examples** | GL Account, Item Number | Custom order fields |
| **Used for** | Navigation/reporting keys | Additional info capture |

---

## 7. FND_FLEX_EXT - Useful API Package

```sql
-- Get concatenated segments for a code combination
l_account := FND_FLEX_EXT.GET_SEGS(
  appl_short_name    => 'SQLGL',
  key_flex_code      => 'GL#',
  structure_number   => 101,   -- Chart of Accounts ID
  combination_id     => 12345  -- code_combination_id
);

-- Validate and get combination ID
l_valid := FND_FLEX_EXT.GET_COMBINATION_ID(
  appl_short_name  => 'SQLGL',
  key_flex_code    => 'GL#',
  structure_number => 101,
  validation_date  => SYSDATE,
  concatenated_segs => '01-HR-5000-USD',
  combination_id   => l_ccid
);

-- Create a new combination
l_new_ccid := FND_FLEX_EXT.GET_CCID(
  appl_short_name  => 'SQLGL',
  key_flex_code    => 'GL#',
  structure_number => 101,
  validation_date  => SYSDATE,
  concatenated_segs => '01-HR-9999-USD'
);
```

---

## 8. Common Interview Scenarios

### Scenario 1: "We need to capture 3 extra fields on the Sales Order"
**Answer**: Add a Descriptive Flexfield (DFF) on `OE_ORDER_HEADERS_ALL`. Define 3 segments mapped to ATTRIBUTE1, ATTRIBUTE2, ATTRIBUTE3. Assign Value Sets for validation. Freeze and compile.

### Scenario 2: "The GL account format needs a new segment"
**Answer**: Modify the Accounting Flexfield (KFF) structure - add a new segment. This requires careful planning: new CCID combinations, revalidation of existing combinations, and potentially mass updating existing accounts.

### Scenario 3: "We need different extra fields based on order type"
**Answer**: Use DFF with Context. Set `ATTRIBUTE_CATEGORY` as the context field. Define one context per order type (e.g., "DOMESTIC", "INTERNATIONAL"). Each context can have different segments using the same ATTRIBUTE columns.

---

## 🎯 Top Interview Questions & Answers

**Q1: What is the difference between a KFF and a DFF?**
> KFF is used for key/identifier structures (like GL account number, Item number) - they're required and meaningful. DFF adds optional extra descriptive information to any record - like additional fields on a form that vary by context.

**Q2: What are segments in a KFF?**
> Each "part" of the combined key. Example: GL Account "01-HR-5000-USD" has 4 segments: Company, Department, Account, Currency. Each segment has its own Value Set for validation.

**Q3: What is a Value Set?**
> The validation set for a flexfield segment. Types include Independent (fixed list), Dependent (depends on another segment), Table (validated against a DB table), or None (no validation).

**Q4: What does "Freeze and Compile" do for a flexfield?**
> Freezing locks the flexfield definition. Compiling generates the database views and backing queries that the forms/applications use to display and validate the flexfield. Required after any change.

**Q5: What is the context in a DFF?**
> Context allows different sets of segments to appear depending on a context field value. Example: If Order Type = "PREMIUM", show VIP fields; if = "STANDARD", show standard fields. The context column is ATTRIBUTE_CATEGORY.

**Q6: Where is DFF data physically stored?**
> In the same table as the main record - in generic columns named ATTRIBUTE1 through ATTRIBUTE30 (usually). The context is in ATTRIBUTE_CATEGORY. No separate table is needed.

---

## 📝 Quick Revision Summary

```
KFF (Key Flexfield):
  = Structured identifier (GL Account, Item Number)
  = SEGMENT1..SEGMENTn columns
  = Required, meaningful key
  = Has Structure -> Segments -> Value Sets
  = FND_FLEX_EXT package for API access

DFF (Descriptive Flexfield):
  = Extra optional fields on any form
  = ATTRIBUTE1..ATTRIBUTE30 columns
  = Context (ATTRIBUTE_CATEGORY) -> different fields per context
  = "Freeze and Compile" required after changes

Value Set:
  None / Independent / Dependent / Table / Special
  = Validation list for each segment

Common KFFs:
  Accounting Flexfield = GL Account (Company-Dept-Account)
  System Items = Item Number
  Territory Flexfield = AR Territory

DFF usage: "Add custom fields to Orders -> use DFF on OE_ORDER_HEADERS_ALL"
```

---

*Next: [Chapter 19 - Oracle Workflow & Alerts](19_oracle_ebs_workflow_alerts.md)*
