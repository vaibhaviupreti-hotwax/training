# Scenario Analysis & Insights

Based on the 50-60 scenarios you provided, here is a breakdown of the specific issues to watch out for, and the architectural patterns these scenarios form.

---

## Part 1: Issues & What to Notice (The "Gotchas")

When implementing these scenarios in OMS, here are the hidden technical issues you need to watch out for:

### 1. The "Equal Exchange" Price Drift
- **Scenario:** `POS_EXC_EQUAL_RS` (Equal Exchange on POS).
- **The Issue:** A customer buys a shirt for $50. Two weeks later, the price of the shirt is increased to $60. The customer comes in to exchange it for a different color. 
- **What to Notice:** The system evaluates the *returned* item using the historical `$50` price (via the original `OrderDate`). But the *new* exchange item is evaluated at the current `$60` price. A supposedly "Equal Exchange" will suddenly prompt the POS to ask the customer for $10. Your OMS logic must handle price-matching for exchanges to override the current effective-dated `ProductPrice`.

### 2. Universal Warranties have no `OrderDate`
- **Scenario:** `Universal Warranty (No Proof of Purchase)`
- **The Issue:** Normally, returns fetch the exact tax rates (`TaxAuthorityRateProduct`) and discounts (`ProductPromo`) that were active when the original order was placed.
- **What to Notice:** Without an original `OrderHeader`, you have no `OrderDate`. When creating a replacement order or credit memo for this warranty, the system is forced to use `NOW()`. If tax rates or item values have changed, your accounting general ledger (GL) will show a discrepancy because you are giving away inventory based on today's value, not what was actually paid.

### 3. "No Restock" Inventory Shrinkage
- **Scenarios:** `WEB_FUL_RFD_NRS`, `LOOP_RETURN_NRS` (`restockType: NO_RESTOCK`).
- **The Issue:** A refund is issued, but the item is not restocked (maybe it's broken, lost, or a Loop return where the customer keeps it).
- **What to Notice:** In the OMS, you cannot just "delete" the inventory. If it's a fulfilled item, it left the warehouse. If it's not restocked, you still have to account for the financial loss. This usually requires routing the return to a specific "Damage/Shrinkage" `Facility` rather than the main selling warehouse.

### 4. The "Store Credit" Accounting Loop
- **Scenarios:** Appeasements to Store Credit, Returns > 30 Days to Store Credit.
- **What to Notice:** Refunding to an Original Payment Method (OPM) is simple: you create a `Credit Memo` -> `Customer Refund`. 
However, issuing Store Credit requires a completely different accounting path: you must create an `Invoice` (to represent the sale of the store credit) and apply the `Credit Memo` to that `Invoice`. If this step fails, the customer gets the store credit, but your accounting books will show an unbalanced liability.

---

## Part 2: Patterns & Flows Noticed

Looking at the data structure of your scenarios, several very distinct logic patterns emerge that dictate how your OMS behaves.

### Pattern 1: The `restockType` Pivot
Every return/cancellation scenario pivots on the `restockType` flag coming from Shopify. This single flag dictates the entire downstream inventory and accounting flow:
- **`restockType: "RETURN"` / `"CANCEL"`**: Triggers the creation of an `ItemReceipt`. Inventory is added back to a `Facility`.
- **`restockType: "NO_RESTOCK"`**: No `ItemReceipt` is generated. The flow stops at the `RefundAgreement`. 

### Pattern 2: The "30-Day" Branching Logic
Your return policies (`SCN_RTN_*`) form a strict decision tree based on the original `OrderDate`:
- **Inside 30 Days:** System attempts to refund to Original Payment Method (OPM) or process an Exchange.
- **Outside 30 Days:** System forcefully routes the refund to **Store Credit** (or allows an Exchange/Warranty).
- *Takeaway:* The OMS must constantly calculate `NOW() - OrderHeader.orderDate` to validate if a Shopify webhook is adhering to the 30-day rule.

### Pattern 3: Loop vs Native Shopify Returns
- **Shopify POS Returns:** Shopify handles the Return and Exchange natively in one fell swoop (`POS_EXC_LESSER_RS`). You receive both the `RefundAgreement` and `ReturnAgreement` together.
- **Loop Returns (`LOOP_*`):** Loop often uses `restockType: "NO_RESTOCK"` for the original item and generates a completely separate "Draft Order" for the exchange item. 
- *Takeaway:* Your OMS has to link two completely separate orders together when Loop is used, whereas POS exchanges are contained within the same Shopify order payload.

### Pattern 4: Send-Sale Attribution Complexity
- **Scenario:** `POS_SSL_NO_ADJ` (POS Send Sale).
- **The Flow:** An order is placed physically at a POS terminal (Store A), but fulfilled by a warehouse (Warehouse B) and shipped to the customer's house.
- *Takeaway:* This requires complex `PartyRelationship` effective-dating. The OMS must record the POS Associate who placed the order for commission purposes, link the Store `Facility` for store-revenue attribution, but route the fulfillment to the Warehouse `Facility`.
