# Chapter 19: Oracle Workflow & Alerts
## ⏱️ ~20 min read

---

## 🧠 Think of it this way

> **Oracle Workflow** = An automated business process manager.
> Like a relay race: Person A passes the baton (notification) to Person B. If B approves, baton goes to Person C. If B rejects, baton goes back to Person A. The workflow tracks all this automatically.
>
> **Oracle Alerts** = The alarm system.
> "Every morning, check if any orders are overdue. If yes, email the manager."
> Set it and forget it - Oracle runs the check automatically.

---

## PART A: ORACLE WORKFLOW

## 1. What is Oracle Workflow?

- Automates and tracks **business processes** across people and systems
- Processes are defined as **flowcharts** (processes, activities, transitions)
- Used heavily in: Order Management booking, AP invoice approval, HR offers, PO approval
- Users interact via **Worklist** (notifications in EBS inbox)
- Built on **Workflow Engine** (WF_ENGINE) and **Notification System**

---

## 2. Key Workflow Concepts

### Process
> The overall workflow definition - the flowchart.
> Example: "Order Approval Process", "PO Approval Process"

### Activity
> Each step/node in the process:
- **Function Activity**: Calls a PL/SQL procedure (automated)
- **Notification Activity**: Sends a message to a user (requires human action)
- **Process Activity**: Sub-process (nested workflow)
- **Event Activity**: Waits for an external event

### Transition
> The arrow between activities - defines what happens next.
> Can be conditional: "On Approve -> next step", "On Reject -> back to requester"

### Item Key
> Unique identifier for each workflow instance.
> Like a ticket number. Example: Order #1001 -> item_key = '1001'

### Item Type
> The workflow definition template.
> Example: `OEOH` = Order Management Header workflow

---

## 3. Key Workflow Tables

```sql
WF_ITEMS                    -- Active/completed workflow instances
WF_ITEM_ACTIVITY_STATUSES   -- Status of each activity in each instance
WF_NOTIFICATIONS            -- Notifications sent to users
WF_NOTIFICATION_ATTRIBUTES  -- Attributes of each notification
WF_PROCESS_ACTIVITIES       -- Activities defined in process
WF_ACTIVITIES               -- Activity definitions
WF_MESSAGES                 -- Message templates
WF_ROLES                    -- Roles/users in workflow
```

---

## 4. Monitoring Workflow Status

```sql
-- Check workflow status for an order
SELECT wi.item_type, wi.item_key, wi.begin_date, wi.end_date,
       wi.activity_status, wi.root_activity
FROM wf_items wi
WHERE wi.item_type = 'OEOH'        -- OM Order Header workflow
AND wi.item_key = '1001';           -- Order ID

-- Find stuck workflows (active but no recent progress)
SELECT item_type, item_key, begin_date,
       ROUND(SYSDATE - begin_date) AS days_running
FROM wf_items
WHERE end_date IS NULL
AND begin_date < SYSDATE - 7       -- Running > 7 days
ORDER BY begin_date;

-- Find pending notifications
SELECT wn.notification_id, wn.subject, wn.sent_date,
       wn.recipient_role, wn.status
FROM wf_notifications wn
WHERE wn.status = 'OPEN'
AND wn.recipient_role = 'JSMITH'
ORDER BY wn.sent_date DESC;

-- Find errored activities
SELECT wias.item_type, wias.item_key, wias.process_activity,
       wias.activity_status, wias.error_name, wias.error_message
FROM wf_item_activity_statuses wias
WHERE wias.activity_status = 'ERROR'
ORDER BY wias.begin_date DESC;
```

---

## 5. Key Workflow APIs

```sql
-- Start a workflow process
WF_ENGINE.CREATEPROCESS(
  itemtype => 'OEOH',
  itemkey  => TO_CHAR(l_header_id),
  process  => 'ORDER_FLOW'
);

-- Set an attribute value
WF_ENGINE.SETITEMATTR(
  itemtype => 'OEOH',
  itemkey  => TO_CHAR(l_header_id),
  aname    => 'ORDER_AMOUNT',
  avalue   => l_amount
);

-- Get an attribute value
l_amount := WF_ENGINE.GETITEMATTR(
  itemtype => 'OEOH',
  itemkey  => TO_CHAR(l_header_id),
  aname    => 'ORDER_AMOUNT'
);

-- Start the process
WF_ENGINE.STARTPROCESS(
  itemtype => 'OEOH',
  itemkey  => TO_CHAR(l_header_id)
);

-- Complete an activity (advance workflow)
WF_ENGINE.COMPLETEACTIVITY(
  itemtype   => 'OEOH',
  itemkey    => TO_CHAR(l_header_id),
  activity   => 'APPROVAL',
  result     => 'APPROVED'
);
```

---

## 6. Workflow Troubleshooting Common Issues

### Errored Workflow
```sql
-- Find error and retry
SELECT error_name, error_message, error_stack
FROM wf_item_activity_statuses
WHERE item_type = 'OEOH' AND item_key = '1001'
AND activity_status = 'ERROR';

-- Retry errored activity via API
WF_ENGINE.HANDLEERROR(
  itemtype  => 'OEOH',
  itemkey   => '1001',
  activity  => 'ORDER_APPROVAL',
  command   => 'RETRY',           -- or 'SKIP'
  result    => ''
);
```

### Workflow Admin UI
```
System Administrator -> Workflow -> Administrator Workflow
-> Find -> Status = Error / Active
-> Select -> Retry / Skip / Abort
```

---

## PART B: ORACLE ALERTS

## 7. What are Oracle Alerts?

Two types:
1. **Event Alert**: Triggered by a database INSERT/UPDATE (uses database trigger underneath)
2. **Periodic Alert**: Runs on a schedule, checks a condition, sends notification if rows returned

---

## 8. Periodic Alert (Schedule-Based)

> "Every morning at 7 AM, run this SQL. If it returns any rows, email the result to the team."

### How it works
1. You write a SELECT statement
2. Define output columns (what to include in the email)
3. Schedule when to run (daily, hourly, etc.)
4. Define actions (email, run concurrent program)
5. Alert Definition saves query -> Scheduled check runs it -> If rows found -> Actions fire

### Example: Alert for Overdue Orders
```sql
-- Alert SQL (returns rows when there's a problem)
SELECT o.order_number, c.customer_name,
       o.ordered_date, (SYSDATE - o.ordered_date) AS days_open
FROM oe_order_headers_all o
JOIN hz_cust_accounts c ON o.sold_to_org_id = c.cust_account_id
WHERE o.flow_status_code = 'BOOKED'
AND o.ordered_date < SYSDATE - 30
AND o.org_id = :$PROFILES$.ORG_ID;

-- If this returns rows -> send email:
-- "Order &ORDER_NUMBER for &CUSTOMER_NAME is &DAYS_OPEN days old!"
```

### Alert Output in Email
```
Order 12345 for ABC Corp is 35 days old!
Order 12346 for XYZ Ltd is 32 days old!
```

---

## 9. Event Alert (DML-Triggered)

> "The moment anyone inserts a new high-value order (> $50,000), immediately email the VP."

### How it works
1. You define the alert on a specific table
2. Specify the trigger event (INSERT, UPDATE, DELETE)
3. Define a SELECT to get relevant data
4. Define action (email, concurrent program, script)
5. Oracle creates a database trigger behind the scenes

### Example: Alert on High-Value Order Insert
```
Table: OE_ORDER_HEADERS_ALL
Event: After Insert
Condition SQL:
  SELECT order_number, ordered_amount
  FROM oe_order_headers_all
  WHERE ROWID = :ROWID
  AND ordered_amount > 50000;

Action: Email VP with order details
```

---

## 10. Key Alert Tables

```sql
ALR_ALERTS                     -- Alert definitions
ALR_SCHEDULED_PROGRAMS         -- Periodic alert schedule info
ALR_ALERT_ACTIONS              -- Actions to take when alert fires
ALR_CHECKS                     -- History of alert runs
ALR_ACTION_HISTORY             -- History of actions taken
```

---

## 11. Workflow vs Alerts - When to Use Which

| | Oracle Workflow | Oracle Alert |
|--|----------------|-------------|
| **Purpose** | Model complex processes with approvals/routing | Simple notifications and checks |
| **Trigger** | Explicit: started by code | Event: DML trigger, or Periodic: schedule |
| **Human interaction** | Yes (Worklist, approvals) | No (notification only, no approve/reject) |
| **Multi-step** | Yes (multiple activities) | No (single check -> action) |
| **Use case** | Order booking, PO approval, HR approval | Overdue invoice alert, error notification |

---

## 12. Oracle Approval Management Engine (AME) - Brief

> AME decides WHO needs to approve WHAT based on rules.
> "If PO amount > $10,000, require VP approval. If > $100,000, require CFO."

- Used with Workflow for dynamic approval routing
- Rules engine: Transaction attributes -> Approval chain
- Used in: PO Approval, AP Invoice Approval, HR Actions

```sql
-- Key AME tables
AME_CALLING_APPS          -- Applications using AME
AME_RULES                 -- Approval rules
AME_CONDITIONS            -- Rule conditions
AME_ACTIONS               -- Actions when conditions met
```

---

## 🎯 Top Interview Questions & Answers

**Q1: What is the difference between Oracle Workflow and Oracle Alerts?**
> Workflow models complex multi-step processes with human interactions (approvals, routing). Alerts are simpler - they check a condition and send a notification. Workflow has state; Alerts don't.

**Q2: What is an Item Type and Item Key in workflow?**
> Item Type is the workflow definition template (like OEOH for OM orders). Item Key uniquely identifies each workflow instance (like the order ID). Together they identify one specific workflow run.

**Q3: How do you troubleshoot an errored workflow?**
> Query WF_ITEM_ACTIVITY_STATUSES for status = 'ERROR' and read error_message. Use WF_ENGINE.HANDLEERROR to retry or skip. Use Workflow Administrator UI for visual troubleshooting.

**Q4: What is the difference between an Event Alert and a Periodic Alert?**
> Event Alert fires immediately when a DML event (INSERT/UPDATE/DELETE) occurs on a table - Oracle creates a trigger automatically. Periodic Alert runs on a schedule (like cron) and checks a SQL query.

**Q5: What does WF_ENGINE.COMPLETEACTIVITY do?**
> Advances a workflow by marking an activity as complete with a result. Used to programmatically respond to a notification or progress through a step. The result determines which transition to follow next.

**Q6: How does Workflow store state between steps?**
> Workflow Item Attributes (WF_ITEM_ATTRIBUTE_VALUES) - key-value pairs stored per item instance. Use WF_ENGINE.SETITEMATTR/GETITEMATTR to read/write. Examples: order amount, customer name, approval result.

---

## 📝 Quick Revision Summary

```
Workflow:
  Item Type  = template (OEOH, POAPPRV)
  Item Key   = instance ID (order_id)
  Activity   = step (Function/Notification/Process)
  Transition = flow between steps

Key APIs:
  WF_ENGINE.CREATEPROCESS -> start
  WF_ENGINE.SETITEMATTR   -> set attribute
  WF_ENGINE.STARTPROCESS  -> launch
  WF_ENGINE.COMPLETEACTIVITY -> advance

Monitor: WF_ITEMS, WF_ITEM_ACTIVITY_STATUSES
Errors: status='ERROR' in WF_ITEM_ACTIVITY_STATUSES
Fix: WF_ENGINE.HANDLEERROR('RETRY'/'SKIP')

Alerts:
  Event Alert  -> triggered by INSERT/UPDATE/DELETE
  Periodic Alert -> scheduled SQL check
  Both -> if rows returned -> send email / run program

Tables: ALR_ALERTS, ALR_ALERT_ACTIONS, ALR_CHECKS
```

---

*Next: [Chapter 20 - Oracle EBS APIs, Interfaces & RICE](20_oracle_ebs_apis_interfaces.md)*
