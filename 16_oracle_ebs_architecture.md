# Chapter 16: Oracle E-Business Suite (EBS) - Architecture & Navigation
## ⏱️ ~15 min read

---

## 🧠 Think of it this way

> Oracle E-Business Suite (EBS) is like a **giant corporate ERP system** that manages everything in a company:
> - HR hires people (HRMS)
> - Finance tracks money (GL, AR, AP)
> - Warehouse ships goods (Inventory, Shipping)
> - Sales takes orders (Order Management)
> - All talking to the same central Oracle Database.
>
> **Oracle "Labs"** typically refers to experience with Oracle EBS development/administration tools - Concurrent Programs, Flexfields, Workflows, APIs.

---

## 1. Oracle EBS Architecture - Three-Tier

```
┌─────────────────────────────────────────────────┐
│          TIER 1: CLIENT (Browser)                │
│   Internet Explorer / Firefox / Chrome           │
│   Users access via URL (http://server:8000)      │
└──────────────────────┬──────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────┐
│          TIER 2: APPLICATION (Middle)            │
│  ┌──────────────────────────────────────────┐   │
│  │  Oracle HTTP Server (Apache OHS)         │   │
│  │  Forms Services (for Forms-based UI)     │   │
│  │  OC4J / WebLogic (for HTML/OAF pages)   │   │
│  │  Concurrent Manager (background jobs)    │   │
│  └──────────────────────────────────────────┘   │
└──────────────────────┬──────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────┐
│          TIER 3: DATABASE                        │
│  Oracle Database (all EBS data lives here)       │
│  Schemas: APPS, AR, AP, GL, ONT, INV, ...       │
└─────────────────────────────────────────────────┘
```

---

## 2. Key EBS Modules to Know

| Module | Code | What it does |
|--------|------|-------------|
| **General Ledger** | GL | Accounting, financial statements |
| **Accounts Payable** | AP | Vendor invoices, payments |
| **Accounts Receivable** | AR | Customer invoices, collections |
| **Order Management** | ONT | Sales orders, shipping |
| **Inventory** | INV | Stock, warehouses |
| **Purchasing** | PO | Purchase orders |
| **HRMS** | HR/PAY | Employees, payroll |
| **Fixed Assets** | FA | Asset tracking, depreciation |
| **Project Accounting** | PA | Project costs/billing |
| **Manufacturing** | WIP/BOM | Work-in-progress, bills of material |

---

## 3. EBS Database - Key Schemas

```
APPS       -> Main schema. ALL EBS objects (synonyms point here)
            SELECT * FROM apps.oe_order_headers_all;

AR         -> Accounts Receivable tables
AP         -> Accounts Payable tables
GL         -> General Ledger tables
ONT        -> Order Management tables (ONT = Order Non-Tax)
INV        -> Inventory tables
HR/PER     -> Human Resources tables
PO         -> Purchasing tables
```

> **Important**: In EBS, all access should go through the `APPS` schema. Direct schema access (AR.RA_CUSTOMERS) is sometimes used for reporting but `APPS` is the standard.

---

## 4. Key Table Naming Conventions

| Suffix | Meaning | Example |
|--------|---------|---------|
| `_ALL` | Multi-org table (data for all orgs) | `OE_ORDER_HEADERS_ALL` |
| No suffix | Single-org view | `OE_ORDER_HEADERS` (view on _ALL filtered by ORG_ID) |
| `_V` | View | `OE_ORDER_HEADERS_V` |
| `_B` | Base table (translations exist) | `FND_LOOKUPS_B` |
| `_TL` | Translation table (multilingual) | `FND_LOOKUPS_TL` |
| `_S` | Sequence | `OE_ORDER_HEADERS_S` |

---

## 5. Multi-Org (Multi-Organization) Architecture

> A company has multiple subsidiaries in different countries. Each subsidiary = an Operating Unit. The database has ONE table for ALL orders (`OE_ORDER_HEADERS_ALL`), but each subsidiary only sees THEIR orders.

### Key Org Concepts
```
Business Group -> Legal Entity -> Operating Unit -> Inventory Organization
(highest level)                (AP, AR, OM)      (INV, WIP)
```

### ORG_ID vs ORGANIZATION_ID
- `ORG_ID` = Operating Unit ID (used in AR, AP, OM, PO)
- `ORGANIZATION_ID` = Inventory Organization ID (used in INV)

### MOAC - Multi-Org Access Control (R12+)
```sql
-- In EBS R12, use MOAC to set org context
EXEC MO_GLOBAL.SET_POLICY_CONTEXT('S', 101);  -- Single OU
EXEC MO_GLOBAL.SET_POLICY_CONTEXT('M', NULL); -- Multiple OUs

-- R11i: use client_info
EXEC DBMS_APPLICATION_INFO.SET_CLIENT_INFO('101');
```

### Security Profiles
- Users are assigned to Operating Units via Security Profiles
- `FND_PROFILE.VALUE('ORG_ID')` -> get current org context

---

## 6. Key EBS Tables to Know

### Foundation / Setup Tables
```sql
FND_APPLICATION               -- Registered applications (AR, GL, ONT...)
FND_RESPONSIBILITY            -- User responsibilities
FND_USER                      -- EBS user accounts
FND_PROFILE_OPTIONS           -- Profile option definitions
FND_PROFILE_OPTION_VALUES     -- Profile option values
FND_LOOKUPS                   -- Lookup types and values
HR_OPERATING_UNITS            -- Operating units
```

### Common Queries
```sql
-- Find a user's responsibilities
SELECT fr.responsibility_name, fa.application_name
FROM fnd_user_resp_groups furg
JOIN fnd_responsibility_tl fr ON furg.responsibility_id = fr.responsibility_id
JOIN fnd_application_tl fa ON fr.application_id = fa.application_id
JOIN fnd_user fu ON furg.user_id = fu.user_id
WHERE fu.user_name = 'JSMITH'
AND furg.end_date IS NULL;

-- Find profile option value
SELECT fpo.profile_option_name, fpov.profile_option_value
FROM fnd_profile_options fpo
JOIN fnd_profile_option_values fpov ON fpo.profile_option_id = fpov.profile_option_id
WHERE fpo.profile_option_name = 'DEFAULTOU';
```

---

## 7. EBS Navigation (User Interface)

### Forms-Based (Classic UI)
```
Navigator -> Responsibility -> Menu -> Form
Example: Oracle Order Management -> Orders, Returns -> Sales Orders
```

### HTML/OAF-Based (HTML UI)
```
Browser URL -> Login -> Navigator -> Application
Example: Self Service Expense -> Create Expense
```

### Responsibility
> A "role" that controls what menus, programs, and data a user can access.
> Like a "badge" that opens certain doors in a building.

```
sysadmin -> System Administrator
           -> users, responsibilities, concurrent programs

order_entry -> Order Entry Superuser
              -> sales orders, customers, pricing
```

---

## 8. EBS Application Object Library (AOL / FND)

> The **FND** (Foundation/AOL) layer is the plumbing of EBS:
> - User management
> - Security
> - Concurrent programs
> - Flexfields
> - Profile options
> - Messages

```sql
-- Key AOL tables
FND_USER                    -- Users
FND_RESPONSIBILITY          -- Responsibilities
FND_MENUS                   -- Menu definitions
FND_FORM_FUNCTIONS          -- Functions (forms, web pages, programs)
FND_REQUEST_GROUPS          -- Group of concurrent programs
FND_CONCURRENT_PROGRAMS     -- Concurrent program definitions
FND_DESCRIPTIVE_FLEXS       -- DFF definitions
FND_ID_FLEXS                -- KFF definitions
```

---

## 9. Profile Options

> Settings that control behavior. Like a light dimmer - same switch but adjustable.

### Profile Option Levels (hierarchy - lower overrides higher)
```
Site Level    -> Default for entire system
Application   -> Overrides for a specific module
Responsibility -> Overrides for a specific responsibility
User          -> Overrides for a specific user (highest precedence)
```

### Key Profile Options to Know
| Profile Option | Purpose |
|---------------|---------|
| `MO: Operating Unit` | Default operating unit |
| `MO: Security Profile` | Controls which OUs user can access |
| `HR: Business Group` | Default business group |
| `OM: Source Code` | Order Management source |
| `GL Ledger Name` | Default ledger |

```sql
-- Get profile value from PL/SQL
l_org_id := FND_PROFILE.VALUE('ORG_ID');
l_user_id := FND_PROFILE.VALUE('USER_ID');

-- Set profile value
FND_PROFILE.PUT('ORG_ID', '101');
```

---

## 10. EBS Setup Steps - Understanding the Pattern

> Every EBS module follows a setup pattern before you can use it:

```
1. Define Lookups (for dropdown values)
2. Set Profile Options (system behavior)
3. Define Key/Descriptive Flexfields (custom fields)
4. Set up Organizational hierarchy (Business Group -> OU -> Inv Org)
5. Define Items/Customers/Vendors (master data)
6. Set up transaction types and rules
7. Assign responsibilities to users
8. Test with transaction
```

---

## 🎯 Top Interview Questions & Answers

**Q1: What are the three tiers of Oracle EBS architecture?**
> Client tier (browser), Application tier (OHS, Forms Services, Concurrent Managers), Database tier (Oracle DB with all EBS schemas).

**Q2: What is the difference between ORG_ID and ORGANIZATION_ID in EBS?**
> ORG_ID is the Operating Unit ID used in financial modules (OM, AR, AP, PO). ORGANIZATION_ID is the Inventory Organization ID used in INV and WIP. They are different levels of the organizational hierarchy.

**Q3: What is Multi-Org (MOAC)?**
> Multi-Org allows one database to serve multiple operating units. Tables with `_ALL` suffix contain data for all orgs. In R12, MO_GLOBAL package sets the security context to determine which org's data is visible.

**Q4: What is a Responsibility in Oracle EBS?**
> A responsibility grants users access to a set of menus, forms, and concurrent programs. It also sets the default organization and other settings. Users can have multiple responsibilities.

**Q5: What is the APPS schema?**
> The APPS schema owns all the public synonyms and database packages for EBS. All application code uses APPS. It grants allow other schemas' objects to be accessed via APPS. Direct access to base schemas (AR, GL) is avoided.

**Q6: What are Profile Options?**
> Configurable system parameters that control module behavior. They have a hierarchy: Site -> Application -> Responsibility -> User. Lower levels override higher levels.

---

## 📝 Quick Revision Summary

```
3-Tier: Client(Browser) -> App(OHS+Forms+CM) -> DB(Oracle)

Key Schemas: APPS (main access), AR, AP, GL, ONT, INV, PO

Multi-Org:
  ORG_ID = Operating Unit (OM, AR, AP, PO)
  ORGANIZATION_ID = Inventory Org (INV, WIP)
  _ALL tables = all orgs. Views = filtered by context.
  MOAC R12: MO_GLOBAL.SET_POLICY_CONTEXT()

Responsibility = Role (menus + programs + org context)

Profile Options:
  FND_PROFILE.VALUE('ORG_ID')
  Hierarchy: User > Responsibility > Application > Site

Key tables:
  FND_USER, FND_RESPONSIBILITY, FND_PROFILE_OPTION_VALUES
  HR_OPERATING_UNITS, FND_LOOKUP_VALUES
```

---

*Next: [Chapter 17 - Concurrent Programs & Request Sets](17_oracle_ebs_concurrent.md)*
