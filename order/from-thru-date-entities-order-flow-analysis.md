# Order Flow & Effective Dating Analysis

This document provides a deep dive into the provided Shopify/OMS order scenarios (Web, POS, Returns, Warranties, Appeasements) and specifically analyzes the critical role of **effective-dated entities** (entities with `fromDate` and `thruDate`) within these flows.

---

## 1. The Danger of Effective Dating (`fromDate` & `thruDate`)

In standard relational databases, if a user updates their shipping address, you `UPDATE` the row. 
In OFBiz/Moqui architectures, **we never update or delete historical records.** Instead, we use effective dating: we expire the old record by setting its `thruDate` to `NOW()`, and we create a new record with a `fromDate` of `NOW()`.

### Why are these entities considered "Dangerous" or "Sensitive"?

> [!WARNING]
> **The "Multiple Records" Bug (Missing Date Filters)**
> If a developer queries `PartyContactMechPurpose` (to get a customer's billing address) and forgets to add a `<date-filter/>`, the query will return **all historical addresses** for that customer. If the code expects a single object (`entity-find-one` or `.first()`), it will either crash or randomly pick an outdated address.

> [!CAUTION]
> **The "Destroyed History" Bug (Incorrect Updates)**
> If a sync job from Shopify updates a customer's phone number by doing an `entity-update` on an existing `TelecomNumber` (instead of expiring the old `PartyContactMech` and creating a new one), it permanently alters historical orders. If customer service looks at an order from 2022, the phone number will falsely show the 2024 number.

> [!IMPORTANT]
> **The "Overlap" Bug (Data Corruption)**
> If two concurrent processes (e.g., a Webhook from Shopify and a manual sync from the UI) try to update a customer's address simultaneously, they might both insert new records with `fromDate = NOW()` without properly expiring the others. The system now has two "Active" shipping addresses, which breaks routing and fulfillment logic.

---

## 2. Sensitive Entities in Your Scenarios

Based on the 50-60 scenarios provided, the following effective-dated entities are the most heavily touched and pose the highest risk for discrepancies:

### Party & Customer Identity
- **`PartyContactMechPurpose`**: Links a Party (Customer) to a ContactMech (Email, Phone, Address) with a specific purpose (Shipping, Billing). 
  - *Risk:* Frequent updates from Shopify webhooks.
- **`PartyRole`**: Identifies a party as a CUSTOMER, SUPPLIER, etc.
- **`PartyRelationship`**: Links a customer to an employee (e.g., POS SendSale).

### Orders & Pricing
- **`OrderRole`**: Links a party to an order (e.g., Bill-To Customer, Ship-To Customer, Sales Rep).
- **`ProductPrice`**: Prices change over time. 
  - *Risk:* Returns and exchanges must reference the price *at the time of the original order*, not the currently active price.
- **`ProductPromoCode` / `ProductPromoRule`**: Discount rules are effective-dated.

---

## 3. Scenario Flow Tracing & Data Discrepancies

### A. Standard Web & POS Orders (WEB_STD_*, POS_CMP_*, POS_SSL_*)

**The Flow:**
1. Order drops from Shopify (Payload includes Customer, Address, Items).
2. OMS checks if the Customer (`Party`) exists.
3. If Address is new or changed, OMS expires the old `PartyContactMechPurpose` and creates a new one.
4. Order is created (`OrderHeader`, `OrderItem`).
5. `OrderRole` records are created, linking the `Party` to the `OrderHeader`.

**Common Discrepancies Caused Here:**
- **WEB_STD_UNF_DCT_ADD (Address Added):** When a user adds an address post-order creation in Shopify, a webhook triggers. If the code updates the `OrderHeader` without checking if the new `PartyContactMechPurpose` is correctly date-filtered, the order might point to a ghost address.
- **POS_SSL_* (Send Sales):** POS orders often involve multiple roles (the Sales Rep creating it, the Customer receiving it). If `OrderRole` lacks strict `fromDate` enforcement during assignment, sales attribution reports will duplicate entries.

### B. Returns & Loop Returns (SCN_RTN_*, LOOP_*)

Returns are the most complex flow because they require looking *backward* in time.

**The Flow:**
1. A Return is initiated in Loop or Shopify.
2. OMS creates a `ReturnHeader` and `ReturnItem`.
3. OMS must calculate the refund amount based on the *original* `OrderHeader` and `OrderItem`.
4. Restock logic executes (`restockType: NO_RESTOCK` vs `RETURN`).

**Common Discrepancies Caused Here:**
- **SCN_RTN_RFD_ORIG (Refund to Original Payment):** The system must look up the original `OrderPaymentPreference`. If the payment gateway token mapping uses an effective-dated entity, missing a `<date-filter/>` could attempt to refund a credit card that was replaced months ago.
- **LOOP_RTN_EXC_NRS (Loop Return & Exchange):** Loop creates a Return and a *new* Draft Order for the exchange. The pricing for the exchange item often needs to match historical `ProductPrice` rules to ensure an "Even Exchange." If the system queries the *current* active price instead of the price active at the `OrderDate`, the customer might be overcharged or undercharged.

### C. Warranties & Appeasements (WEB_APS_*, POS_GFT_RTN_*)

**The Flow:**
1. Customer Support issues an appeasement (partial refund) or warranty replacement.
2. A Credit Memo or Customer Refund is generated.
3. If Store Credit is issued, an Invoice is generated and the Credit Memo is applied to it.

**Common Discrepancies Caused Here:**
- **POS/Ecom Appeasement to Store Credit:** Creating Store Credit involves generating a FinAccount or Gift Card. These financial accounts have effective dates (`fromDate` for when the credit was issued, `thruDate` for expiration). 
- **"Universal Warranty (No Proof of Purchase)" (M125145):** Since there is no original `OrderHeader` to link to, the system must create a standalone `ReturnHeader`. When creating the replacement order, any automated pricing engines will use the *current* `ProductPrice` (since there is no historical order date to reference). This can cause accounting discrepancies if the item's price has increased, making the "free" warranty replacement look like a large financial loss on the books.

---

## 4. Best Practices for Developers

To prevent these discrepancies in your OMS code:

1. **Always use `<date-filter/>` in View-Entities:**
   Whenever you join a table like `PartyContactMechPurpose`, you *must* include `<date-filter/>` in the `<entity-condition>`.
   
2. **Never `UPDATE` an effective-dated record:**
   Instead, use a service like `update#PartyContactMech` which is explicitly designed to handle the expiration and recreation of the record under the hood.
   
3. **Pass `moment()` or `OrderDate` to Pricing Engines:**
   When doing Returns or Exchanges, never query `ProductPrice` with `NOW()`. Always pass the `OrderDate` of the original order to ensure you fetch the `ProductPrice` that was active *at that exact moment in history*.
