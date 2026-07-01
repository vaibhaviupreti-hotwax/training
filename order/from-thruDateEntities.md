# Effective-Dated Entities Catalog

In OFBiz / HotWax architectures, "Effective Dating" is the practice of using `fromDate` and `thruDate` to maintain a historical log of records rather than overwriting them. 

Below is a detailed list of the most critical effective-dated entities you will interact with during Order Management, Returns, and Customer Identity flows.

---

## 1. Customer Identity & Contact Data

### `PartyContactMech`
- **Description:** Links a `Party` (Customer/Company) to a `ContactMech` (the actual string representing the address, email, or phone number).
- **Usage:** If a customer moves to a new house, you do not update the `ContactMech`. Instead, you set the `thruDate` on their old `PartyContactMech` to `NOW`, and create a new `ContactMech` and `PartyContactMech` with a `fromDate` of `NOW`.
- **Related Entities:** `ContactMech` (holds the raw data), `PostalAddress`, `TelecomNumber`.
- **Interesting Insight:** Because the actual `ContactMech` is immutable (never updated), historical orders linked directly to a `contactMechId` will always show the exact address exactly as it was typed when they placed the order years ago.

### `PartyContactMechPurpose`
- **Description:** Defines *why* a party is linked to a contact mechanism (e.g., `SHIPPING_LOCATION`, `BILLING_LOCATION`, `PRIMARY_EMAIL`).
- **Usage:** A customer might use the same physical address for both shipping and billing. Later, they change their billing address. You would expire (`thruDate = NOW`) only the `PartyContactMechPurpose` record for `BILLING_LOCATION` tied to the old address, and create a new one for the new address.
- **Interesting Insight:** This entity is the root cause of the "Ghost Address" bug. If you query a customer's shipping address and forget `<date-filter/>`, the system returns every shipping address they have ever lived at.

### `PartyRelationship`
- **Description:** Defines a relationship between two `Party` records over a period of time.
- **Usage:** Used extensively in B2B (e.g., Party A is a "Subsidiary" of Party B) or POS (e.g., Party A is the "Assigned Sales Rep" for Customer Party B).
- **Related Entities:** `PartyRole` (must exist for both parties before a relationship can be formed).
- **Interesting Insight:** The `thruDate` here is vital for employee turnover. If a POS associate quits, you expire their `PartyRelationship` to the store. If you accidentally delete it instead, all past sales attribution reports for that employee will break.

---

## 2. Catalog, Pricing & Promotions

### `ProductPrice`
- **Description:** Defines the cost or selling price of a product.
- **Usage:** Prices change constantly (inflation, sales, bulk pricing). You create a new `ProductPrice` with a new `fromDate` when a price increases.
- **Related Entities:** `ProductPriceType` (e.g., `DEFAULT_PRICE`, `LIST_PRICE`), `ProductPricePurpose`.
- **Interesting Insight (Returns/Exchanges):** When processing a Return (like your `LOOP_RTN_EXC_NRS` scenario), you must evaluate the item's value based on the `ProductPrice` that was active at the *time of the original order*. If you query this table using `NOW()`, an even exchange could suddenly require the customer to pay out of pocket because the item's current price went up!

### `ProductCategoryMember`
- **Description:** Links a `Product` to a `ProductCategory`.
- **Usage:** E-commerce stores rotate collections (e.g., a shirt belongs to the "Summer 2024 Collection"). The `thruDate` automatically drops the product from the category on the website when the season ends.
- **Related Entities:** `Product`, `ProductCategory`.

### `ProductPromoRule` / `ProductPromo`
- **Description:** Represents discount codes, BOGO deals, and sitewide sales.
- **Usage:** Promotions run for specific timeframes. The `fromDate` and `thruDate` dictate exactly when the promo code becomes valid and when it expires.
- **Interesting Insight:** If a customer complains that a promo code didn't work, developers often check this table using the `OrderDate`.

---

## 3. Supply Chain & Facilities

### `FacilityParty`
- **Description:** Links a `Party` (typically an employee or manager) to a `Facility` (warehouse or retail store).
- **Usage:** Tracks who is working at or managing a specific location.
- **Interesting Insight:** Useful in POS Send-Sale scenarios (`POS_SSL_NO_ADJ`) to verify if the employee ringing up the order actually had an active employment record at that specific store location on the day of the sale.

### `SupplierProduct`
- **Description:** Defines the terms under which you purchase a product from a vendor/supplier (cost, lead time, minimum order quantity).
- **Usage:** Suppliers frequently change their wholesale prices. `fromDate` and `thruDate` keep a historical log of what you paid your vendors in previous quarters.

---

## 4. Why OFBiz / HotWax Uses This Pattern

1. **Auditability (The "Time Machine"):** 
   You can recreate the exact state of your business at any second in the past. If you need to know a customer's address, the price of a product, and the active promotions on **October 4th, 2022 at 2:14 PM**, you simply pass that exact timestamp into the `<date-filter valid-date="2022-10-04 14:14:00"/>` tag in your queries.

2. **Database Integrity:**
   `OrderHeader` and `OrderItem` tables are historically immutable. If you allowed a simple `UPDATE` on a product's price or a customer's address, historical invoices and financial reports generated today would conflict with the actual receipts printed out and handed to the customer years ago.

----

### Detailed..
# The Exhaustive Catalog of Effective-Dated Entities

OFBiz (and Moqui/HotWax) uses the Universal Data Model pattern. Because of this, effective dating (`fromDate` and `thruDate`) is applied across hundreds of tables. 

Here is a comprehensive breakdown covering all major domains of the system.

---

## 1. Party, Contacts & Customers (Identity Domain)

These entities track *who* people are, *where* they are, and *how* they relate to each other over time.

| Entity Name | What it tracks over time | Why it uses effective dating |
|-------------|--------------------------|------------------------------|
| **`PartyContactMech`** | Links a customer/party to an address/email/phone. | People move or change emails. You keep the old address linked to past orders, but it expires for future orders. |
| **`PartyContactMechPurpose`** | Tracks what an address is used for (`SHIPPING`, `BILLING`). | A customer might stop using a physical address for billing, but keep using it for shipping. |
| **`PartyRelationship`** | Links two parties (e.g., Parent/Child company, Employer/Employee). | Tracks employment history or corporate acquisitions. If an employee quits, the `thruDate` is populated. |
| **`PartyRate`** | How much a party is billed or paid per hour/unit. | Tracks salary or contractor rate changes over years. |
| **`PartySkill`** | The skill level of an employee. | An employee's proficiency might upgrade from "Junior" to "Senior" on a certain date. |
| **`Vendor`** | Specific terms for a vendor. | Vendor agreements (payment terms) change yearly. |
| **`CustRequestParty`** | Who is assigned to a customer service ticket. | If a ticket is transferred from Agent A to Agent B, the old assignment is expired, not deleted. |

---

## 2. Product, Catalog & Categories

Products themselves (`Product` table) are static, but how they are categorized, priced, and sold changes constantly.

| Entity Name | What it tracks over time | Why it uses effective dating |
|-------------|--------------------------|------------------------------|
| **`ProductCategoryMember`** | Links a Product to a Category. | "Summer 2024 Collection". Products are automatically added and removed from the website based on these dates. |
| **`ProdCatalogCategory`** | Links a Category to a Catalog (e.g., Web Store vs Wholesale). | You might expose a wholesale category to the portal only during Q3. |
| **`ProductAssoc`** | How products relate to each other (Cross-sell, Up-sell, Component). | A bundle (e.g., a Gift Basket) might contain different items in winter vs summer. The associations expire accordingly. |
| **`ProductFeatureAppl`** | Links a feature (Size/Color) to a product. | If a manufacturer stops making the "Red" version of a shirt, the feature application is expired. |

---

## 3. Pricing, Promotions & Agreements

This is the most financially sensitive area for effective dating. **Querying these tables without a `<date-filter/>` matching the original order date will destroy your accounting during returns.**

| Entity Name | What it tracks over time | Why it uses effective dating |
|-------------|--------------------------|------------------------------|
| **`ProductPrice`** | The selling cost of an item. | Prices increase due to inflation. You must know exactly what an item cost on November 3rd, 2019. |
| **`ProductPromo` & `ProductPromoRule`** | Discount campaigns and their specific rules (e.g., 20% off). | Sales run for specific periods (e.g., Black Friday). The promo engine automatically ignores promos where `thruDate < NOW()`. |
| **`ProductPromoCode`** | The actual coupon code a user types in. | Some codes are valid forever, others are time-boxed (e.g., "SPRING2024"). |
| **`Agreement` & `AgreementItem`** | B2B contracts outlining special pricing or terms. | Contracts have strict start and end dates. |
| **`AgreementTerm`** | Specific clauses in the contract (e.g., Net 30 payment). | Terms can be renegotiated without rewriting the entire agreement. |

---

## 4. Facility, Inventory & Supply Chain

| Entity Name | What it tracks over time | Why it uses effective dating |
|-------------|--------------------------|------------------------------|
| **`FacilityParty`** | Which employees work at which warehouse/store. | Retail employees transfer between stores. Essential for POS attribution. |
| **`SupplierProduct`** | Purchasing terms from your vendor (Cost, Lead time, Minimums). | Your supplier raises their wholesale prices; you expire the old record so past purchase orders remain accurate. |
| **`FacilityLocation`** | The physical bins/aisles in a warehouse. | If a warehouse is remapped, old bin locations are expired rather than deleted to preserve the history of where items *used* to be stored. |

---

## 5. Accounting, Billing & General Ledger

Financial systems are built on strict auditability.

| Entity Name | What it tracks over time | Why it uses effective dating |
|-------------|--------------------------|------------------------------|
| **`FinAccountRole`** | Who has access to a financial account (e.g., Store Credit, Gift Card). | If someone's access to a corporate card is revoked, the role is expired. |
| **`PaymentMethod`** | Credit cards, bank accounts. | Credit cards naturally expire. (`thruDate` is often tied to the physical card expiration). |
| **`BillingAccountRole`** | Who is authorized to charge against a B2B billing account. | Employees joining/leaving a B2B client company. |
| **`GlAccountOrganization`** | How a GL account maps to a specific corporate entity. | Chart of Accounts restructuring. |
| **`TaxAuthorityRateProduct`** | Sales tax rates. | Governments change tax rates (e.g., State tax increases from 7% to 7.5% on Jan 1st). Past invoices must use the 7% rate; future ones use 7.5%. |

---

## 6. WorkEffort & Human Resources

The WorkEffort module handles tasks, calendars, and manufacturing runs.

| Entity Name | What it tracks over time | Why it uses effective dating |
|-------------|--------------------------|------------------------------|
| **`WorkEffortPartyAssignment`** | Who is assigned to a task/project. | Tracking exactly when someone started and stopped working on a ticket or manufacturing run. |
| **`Employment`** | The formal employment record. | Start date and termination date of an employee. |
| **`PayHistory`** | Salary and wage history. | Pay raises. Overwriting a salary would ruin retroactive payroll calculations. |

---

## Why are some entities NOT effective-dated?

You might notice that `OrderHeader`, `OrderItem`, `ReturnHeader`, `Shipment`, and `Invoice` do **not** use `fromDate` and `thruDate`. 

**Why? Because they are Event Records, not State Records.**
- An **Event** (an order being placed, a shipment leaving the dock) happens at a singular point in time (recorded via `orderDate` or `createdDate`). Events are historically immutable—they are a permanent receipt of what happened.
- A **State** (the price of a product, a customer's address) is a truth that holds over a duration of time. Effective dating is used specifically to manage State.
