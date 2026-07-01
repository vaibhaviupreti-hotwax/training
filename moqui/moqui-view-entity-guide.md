# How to Write a Moqui View-Entity — Complete Guide

> Based on Moqui's XML Entity Definition patterns.

A `view-entity` in Moqui is the equivalent of a **SQL View**. It allows you to join multiple physical entities together, apply default filters, aggregate data, and create custom fields using SQL functions.

---

## 1. Where do they go?

View-entities are defined in the exact same XML files as regular entities. 
Location: `your-component/entity/YourEntities.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<entities xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:noNamespaceSchemaLocation="http://moqui.org/xsd/entity-definition-3.xsd">

    <!-- Regular entities -->
    <entity entity-name="MoquiTraining" package="moquitraining">...</entity>

    <!-- View entities go here too! -->
    <view-entity entity-name="A1Q01NewCustomers" package="co.hotwax.upreti">
        ...
    </view-entity>

</entities>
```

---

## 2. Basic Structure of a View-Entity

A view-entity consists of three main parts:
1. **Members (`<member-entity>`)**: The tables you are joining.
2. **Aliases (`<alias>` / `<alias-all>`)**: The columns you want to select.
3. **Conditions (`<entity-condition>`)**: Global `WHERE` clauses applied to the whole view.

### A Simple Example
```xml
<view-entity entity-name="OrderAndParty" package="co.hotwax.training">
    
    <!-- 1. The primary table -->
    <member-entity entity-alias="OH" entity-name="org.apache.ofbiz.order.order.OrderHeader"/>
    
    <!-- 2. The joined table -->
    <member-entity entity-alias="P" entity-name="org.apache.ofbiz.party.party.Party" join-from-alias="OH">
        <key-map field-name="partyId"/>
    </member-entity>

    <!-- 3. What to select -->
    <alias-all entity-alias="OH"/> <!-- Select all fields from OrderHeader -->
    <alias name="partyTypeEnumId" entity-alias="P"/> <!-- Select specific field from Party -->
    
</view-entity>
```

---

## 3. The `<member-entity>` Tag (Joins)

This is how you bring tables into the query.

### Attributes
- `entity-alias`: A short name for the table (like `OH` for OrderHeader). **Required.**
- `entity-name`: The fully qualified name of the entity you are pulling from (e.g., `org.apache.ofbiz.order.order.OrderHeader`). **Required.**
- `join-from-alias`: The alias of the table you are joining *to*. If omitted, this is the primary table.
- `join-optional`: Set to `true` for a `LEFT OUTER JOIN`. Default is `false` (`INNER JOIN`).

### How to Join (`<key-map>`)
Inside the `<member-entity>`, use `<key-map>` to specify the `ON` condition of the join.

```xml
<member-entity entity-alias="PER" entity-name="org.apache.ofbiz.party.party.Person" 
               join-from-alias="P" join-optional="true">
    
    <!-- Simplest: When field names match in both tables -->
    <!-- Translates to: ON P.partyId = PER.partyId -->
    <key-map field-name="partyId"/>
    
    <!-- When field names differ -->
    <!-- Translates to: ON OH.shippingContactMechId = CM.contactMechId -->
    <key-map field-name="contactMechId" related="shippingContactMechId"/>
    
</member-entity>
```

### Advanced Joins with Conditions
Sometimes you need more than just `ID = ID` in your `ON` clause. You can add an `<entity-condition>` inside the `<member-entity>`.

```xml
<member-entity entity-alias="GI" entity-name="org.apache.ofbiz.product.product.GoodIdentification" 
               join-from-alias="P" join-optional="true">
    <key-map field-name="productId" />
    <entity-condition>
        <!-- Adds: AND GI.goodIdentificationTypeId = 'ERP_ID' to the JOIN ON clause -->
        <econdition field-name="goodIdentificationTypeId" value="ERP_ID"/>
    </entity-condition>
</member-entity>
```

---

## 4. Aliases (SELECT clause)

You must tell the view-entity which columns to expose. If you don't alias a column, it won't exist in the output!

### Method 1: `<alias-all>`
Selects every field from that entity.
```xml
<alias-all entity-alias="P"/>

<!-- You can also exclude specific fields -->
<alias-all entity-alias="GI">
    <exclude field="productId"/> <!-- Exclude so we don't have duplicate ID columns -->
</alias-all>
```

### Method 2: `<alias>`
Select specific fields, and optionally rename them.
```xml
<!-- Selects PER.firstName and keeps the name 'firstName' -->
<alias name="firstName" entity-alias="PER"/>

<!-- Selects P.partyId but renames it to 'hotwaxId' -->
<alias name="hotwaxId" entity-alias="P" field="partyId"/>
```

---

## 5. `<complex-alias>` (SQL Functions & Math)

If you need SQL functions like `CONCAT`, math operations, or `CASE` statements, use a complex alias.

### Example: String Concatenation (`CONCAT`)
```xml
<alias name="contactNumber" type="text-medium">
    <complex-alias function="CONCAT">
        <complex-alias-field entity-alias="TN" field="countryCode"/>
        <complex-alias expression="'-'"/>
        <complex-alias-field entity-alias="TN" field="areaCode"/>
        <complex-alias expression="'-'"/>
        <complex-alias-field entity-alias="TN" field="contactNumber"/>
    </complex-alias>
</alias>
```
*SQL Equivalent:* `CONCAT(TN.countryCode, '-', TN.areaCode, '-', TN.contactNumber)`

### Example: Math Operators
```xml
<alias name="totalRevenue" type="number-decimal">
    <complex-alias operator="*">
        <complex-alias-field entity-alias="OI" field="quantity" default-value="0"/>
        <complex-alias-field entity-alias="OI" field="unitPrice" default-value="0"/>
    </complex-alias>
</alias>
```
*SQL Equivalent:* `(COALESCE(OI.quantity, 0) * COALESCE(OI.unitPrice, 0))`

---

## 6. Aggregation (GROUP BY & Aggregate Functions)

Moqui view-entities handle `GROUP BY` automatically! 
If *any* alias uses an aggregate function (like `sum`, `count`, `min`, `max`), Moqui will automatically `GROUP BY` all the *non-aggregate* aliases.

```xml
<view-entity entity-name="OrderCompletedHourly" package="co.hotwax.upreti">
    <member-entity entity-alias="OS" entity-name="org.apache.ofbiz.order.order.OrderStatus"/>

    <!-- Non-aggregate field -> Moqui will GROUP BY this -->
    <alias name="hour" type="number-integer">
        <complex-alias function="hour">
            <complex-alias-field entity-alias="OS" field="statusDatetime"/>
        </complex-alias>
    </alias>

    <!-- Aggregate field -->
    <alias name="totalOrders" function="count" type="number-integer">
        <complex-alias operator="*"/> <!-- Equivalent to COUNT(*) -->
    </alias>
</view-entity>
```

> [!TIP]
> Standard aggregate functions: `count`, `count-distinct`, `sum`, `min`, `max`, `avg`.

---

## 7. Global Conditions (The WHERE Clause)

You can place an `<entity-condition>` at the bottom of the `<view-entity>` (outside of any member-entities). This applies a global `WHERE` clause to the entire view.

```xml
<view-entity entity-name="CancelledOrders" package="co.hotwax.upreti">
    <member-entity entity-alias="OS" entity-name="org.apache.ofbiz.order.order.OrderStatus"/>
    
    <alias-all entity-alias="OS"/>

    <!-- This applies globally to the view -->
    <entity-condition>
        <econdition entity-alias="OS" field-name="statusId" value="ORDER_CANCELLED"/>
        <econdition entity-alias="OS" field-name="changeReason" operator="is-not-null" />
    </entity-condition>
</view-entity>
```

### The `<date-filter/>` Magic Tag
If you are joining entities that use Moqui's effective-dating pattern (`fromDate` and `thruDate`), you can use `<date-filter/>` to automatically filter out expired records.

```xml
<member-entity entity-alias="PCMP" entity-name="org.apache.ofbiz.party.contact.PartyContactMechPurpose" 
               join-from-alias="PER" join-optional="true">
    <key-map field-name="partyId"/>
    <entity-condition>
        <!-- Magic! Automatically adds: 
             (fromDate <= NOW OR fromDate IS NULL) AND (thruDate > NOW OR thruDate IS NULL) -->
        <date-filter/>
        <econdition field-name="contactMechPurposeTypeId" value="PRIMARY_EMAIL"/>
    </entity-condition>
</member-entity>
```

---

## Summary Cheat Sheet

| Tag / Attribute | What it does | Example / SQL Equivalent |
|-----------------|--------------|--------------------------|
| `<view-entity>` | Defines the view | `CREATE VIEW ...` |
| `<member-entity>` | Adds a table to the view | `FROM table`, `JOIN table` |
| `join-optional="true"` | Makes it a Left Join | `LEFT OUTER JOIN table` |
| `<key-map>` | Join condition | `ON A.id = B.id` |
| `<alias-all>` | Select all columns | `SELECT A.*` |
| `<alias>` | Select specific column | `SELECT A.col AS name` |
| `<complex-alias>` | SQL functions/math | `SELECT CONCAT(...)`, `SELECT (A * B)` |
| `function="sum"` | Aggregate data | `SUM(col)` (Triggers auto GROUP BY) |
| `<entity-condition>` | Where clause | `WHERE col = 'value'` |
| `<date-filter/>` | Active date check | `WHERE fromDate <= NOW AND thruDate > NOW` |
