I have completely rewritten the artifact into The Ultimate Moqui Services & Scripting Manual!

It is now a massive, 6-chapter masterclass designed to take you from a complete beginner to an advanced framework engineer. Here is the new breakdown:

Chapter 1: The Basics - Explains the architecture, what a service is, and the exact anatomy of a Services.xml file.
Chapter 2: The Complete XML DSL - An exhaustive reference on how to write 90% of your code without Groovy. It covers variables (<set>), conditionals (<if>), loops (<iterate>), database queries (<entity-find>, <entity-create>), and calling other services (<service-call>).
Chapter 3: The ec Context Deep Dive - Moves into Groovy, explaining the magic context map, and detailing exactly how to use ec.entity, ec.service, ec.user, ec.logger, and ec.transaction (database rollbacks).
Chapter 4: Advanced Framework Hacking - Teaches the strategy of moving from Public APIs to Internal org.moqui.impl classes.
Chapter 5: Deep Dive Deconstruction - The most detailed explanation of your Q1 script yet. I broke down every single line, explaining why we needed to translate strings into FieldInfo arrays, and how the EntityConditionImplBase condition tree works.
Chapter 6: Codebase Study Guide - References where to find this level of Groovy in the actual OMS component.
Take your time going through this massive new version, and let me know if it hits the perfect level of detail you were looking for!
---
# The Ultimate Moqui Services & Scripting Manual

Welcome to the comprehensive masterclass on Moqui Services and Groovy Scripting. This manual is designed to take you from a beginner who is just learning how to define a service, to an advanced framework engineer capable of hacking into Moqui’s internal Java classes to perform highly complex operations.

This document is bifurcated into chapters, starting with the absolute basics and progressively diving deeper into the framework's core.

---

## CHAPTER 1: The Basics of Moqui Architecture

### 1.1 What is a Moqui Service?
In Moqui, an **Entity** represents a database table (data storage), while a **Service** represents an action or business logic (data manipulation). 

If you want to create a new user, calculate a discount, or send an API request to Shopify, you write a Service.

### 1.2 The Structure of a Service file
Services are written in XML files (usually ending in `Services.xml` or `Services.secas.xml`). A single file can hold many services.

```xml
<services xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="http://moqui.org/xsd/service-definition-3.xsd">
    <!-- Services go here -->
</services>
```

### 1.3 The Anatomy of a Single Service
Every service has three main parts:
1. **Inputs (`<in-parameters>`)**: What data does this service need to run?
2. **Outputs (`<out-parameters>`)**: What data will this service return to the caller?
3. **Logic (`<actions>`)**: The actual code that runs.

```xml
<service verb="calculate" noun="OrderTotal" authenticate="false">
    <in-parameters>
        <parameter name="orderId" required="true" type="String"/>
        <parameter name="applyDiscount" type="Boolean" default-value="false"/>
    </in-parameters>
    <out-parameters>
        <parameter name="finalTotal" type="BigDecimal"/>
    </out-parameters>
    <actions>
        <!-- The logic goes here -->
    </actions>
</service>
```
*Note on Naming:* Services are named using a `verb#noun` convention (e.g., `calculate#OrderTotal`).

---

## CHAPTER 2: The XML DSL (Domain Specific Language)

Before you write any Groovy, you must master the XML DSL. Moqui is highly optimized to run XML tags incredibly fast. It is always best practice to use XML tags instead of Groovy whenever possible.

### 2.1 Context and Variables
In Moqui, all variables live in a massive invisible Map called the **Context**. 
- Any `<in-parameters>` are automatically added to the Context.
- Any variable you create in the `<actions>` block is added to the Context.
- When the service finishes, Moqui looks in the Context for variables matching your `<out-parameters>` and returns them.

**Creating variables:**
```xml
<!-- Hardcoded string -->
<set field="greeting" value="Hello World"/>

<!-- Groovy expression evaluation (use 'from') -->
<set field="calculatedTax" from="basePrice * 0.05"/>

<!-- Copying from one map to another -->
<set field="userMap.firstName" from="party.firstName"/>
```

### 2.2 Logic and Flow Control
Moqui provides standard programming logic through XML tags.

**If / Else:**
```xml
<if condition="calculatedTax > 100">
    <then>
        <log level="warn" message="High tax detected: ${calculatedTax}"/>
    </then>
    <else-if condition="calculatedTax == 0">
        <log level="info" message="Tax exempt."/>
    </else-if>
    <else>
        <log level="info" message="Normal tax applied."/>
    </else>
</if>
```
*(Note: The `condition` attribute always evaluates as a Groovy boolean expression).*

**Loops (`<iterate>`):**
```xml
<!-- Assuming 'orderItems' is a List of Maps -->
<iterate list="orderItems" entry="item">
    <log level="info" message="Processing Item ID: ${item.productId} with quantity ${item.quantity}"/>
</iterate>
```

**Exiting a Service (`<return>`):**
```xml
<if condition="!userHasAccess">
    <return error="true" message="You do not have permission to do this."/>
</if>
```

### 2.3 Interacting with the Database (Entity Facade)
Moqui's XML DSL provides powerful tags for database CRUD (Create, Read, Update, Delete) operations.

**Fetching a single record (`<entity-find-one>`):**
You must provide the Primary Key.
```xml
<entity-find-one entity-name="mantle.product.Product" value-field="myProduct">
    <field-map field-name="productId" from="inputProductId"/>
</entity-find-one>
```

**Fetching multiple records (`<entity-find>`):**
Equivalent to a SQL `SELECT`.
```xml
<entity-find entity-name="mantle.order.OrderHeader" list="orderList" limit="50" offset="0">
    <!-- WHERE partyId = inputPartyId -->
    <econdition field-name="partyId" from="inputPartyId"/>
    
    <!-- WHERE statusId IN ('ORDER_APPROVED', 'ORDER_COMPLETED') -->
    <econdition field-name="statusId" operator="in" value="ORDER_APPROVED,ORDER_COMPLETED"/>
    
    <!-- WHERE orderName LIKE '%searchQuery%' (Ignored entirely if searchQuery is null/empty) -->
    <econdition field-name="orderName" operator="like" value="%${searchQuery}%" ignore-if-empty="true"/>
    
    <!-- SELECT specific columns only -->
    <select-field field-name="orderId, grandTotal, statusId"/>
    
    <!-- ORDER BY orderDate DESC -->
    <order-by field-name="-orderDate"/>
</entity-find>
```

**Modifying Data (`<entity-create>`, `<entity-update>`, `<entity-delete>`):**
```xml
<!-- CREATE -->
<entity-make-value entity-name="mantle.party.Party" value-field="newParty"/>
<set field="newParty.partyTypeEnumId" value="PtyPerson"/>
<entity-create value-field="newParty"/>

<!-- UPDATE -->
<set field="myProduct.description" value="Updated description"/>
<entity-update value-field="myProduct"/>

<!-- DELETE -->
<entity-delete value-field="myProduct"/>
```

### 2.4 Calling Other Services
Services often call other services to reuse logic.
```xml
<service-call name="create#mantle.order.OrderHeader" in-map="context" out-map="newOrderData"/>
<log level="info" message="Successfully created order: ${newOrderData.orderId}"/>
```
- `in-map="context"`: Passes all current variables to the next service.
- `out-map="newOrderData"`: Saves the result into a specific map to prevent it from overwriting your current variables.

---

## CHAPTER 3: Intermediate - Groovy and the Execution Context (`ec`)

When XML is not powerful enough (e.g., complex math, parsing JSON, cryptographic hashing, massive list filtering), you drop into Groovy using the `<script>` tag.

```xml
<actions>
    <script><![CDATA[
        // Groovy code goes here!
    ]]></script>
</actions>
```

### 3.1 The Magic `context` Map
Inside a Groovy block, you have instant access to everything in the Moqui Context.
```xml
<in-parameters><parameter name="price"/></in-parameters>
<out-parameters><parameter name="tax"/></out-parameters>
<actions>
    <script><![CDATA[
        // 'price' automatically exists because it's an in-parameter
        BigDecimal calculatedTax = price * 0.05
        
        // Setting 'tax' automatically returns it because it's an out-parameter
        tax = calculatedTax
    ]]></script>
</actions>
```

### 3.2 The `ec` (ExecutionContext) Object
The `ec` object is globally available in every Groovy script. It is the core interface to the Moqui engine. Here are its most important sub-systems:

**A. `ec.entity` (Database)**
```groovy
// Finding multiple records
EntityList orders = ec.entity.find("mantle.order.OrderHeader")
    .condition("statusId", "ORDER_APPROVED")
    .condition("grandTotal", EntityCondition.GREATER_THAN, 100.00)
    .orderBy("-orderDate")
    .limit(10)
    .list()

// Finding one record
EntityValue product = ec.entity.find("mantle.product.Product")
    .condition("productId", "1001").one()
    
// Updating a record
product.description = "New description via Groovy"
product.update()
```

**B. `ec.service` (Services)**
```groovy
// Synchronous service call
Map result = ec.service.sync().name("calculate#OrderTotal")
    .parameters([orderId: "1001", applyDiscount: true])
    .call()

// Asynchronous service call (runs in the background, non-blocking)
ec.service.async().name("send#OrderConfirmationEmail")
    .parameters([orderId: "1001"])
    .call()
```

**C. `ec.logger` (Logging)**
```groovy
ec.logger.trace("Deep debugging info")
ec.logger.info("Normal operational info: ${result}")
ec.logger.warn("Something looks weird, but not broken.")
ec.logger.error("System crash!", new Exception("Oops"))
```

**D. `ec.user` (Security)**
```groovy
String userId = ec.user.userId
if (ec.user.hasPermission("ORDER_APPROVE")) {
    ec.logger.info("User is authorized to approve orders.")
}
```

**E. `ec.transaction` (Rollbacks and Commits)**
If you are doing risky data modifications, wrap it in a transaction.
```groovy
boolean beganTransaction = ec.transaction.begin(null)
try {
    // Risky updates...
    ec.entity.find("OrderHeader").condition("statusId", "PENDING").updateAll([statusId: "CANCELLED"])
    ec.transaction.commit(beganTransaction)
} catch (Exception e) {
    ec.transaction.rollback(beganTransaction, "Failed to cancel orders", e)
}
```

---

## CHAPTER 4: Advanced - Hacking the Framework Internals

This is where you become a true Moqui Engineer. 

Everything you learned in Chapter 3 (`ec.entity`, `ec.service`) is the **Public API**. It is designed to be safe and easy to use. But what if you want to do something the Public API doesn't support? Like *extracting the raw SQL of a query without executing it?*

You cannot find `.getRawSql()` on `ec.entity`. You must hack the internals.

### 4.1 The Core Concept: Interfaces vs. Implementations
Moqui is written in Java and Groovy. When you type `def query = ec.entity.find("Product")`, the object returned is technically an interface (`org.moqui.entity.EntityFind`). Interfaces hide their variables.

However, behind every interface is a **concrete implementation class** that does the actual work. For `EntityFind`, the implementation class is `org.moqui.impl.entity.EntityFindBase`.

**The Hacking Strategy:**
1. Find the internal Implementation class in the source code.
2. Forcefully "Typecast" your public object into the Implementation class.
3. Access the hidden protected variables and methods that the framework creators use.

### 4.2 Finding the Source Code
In your local environment, navigate to:
`/moqui-framework/framework/src/main/groovy/org/moqui/impl/`

This folder contains the literal heart of the Moqui engine. 
- `/entity/`: Contains `EntityFindBase.groovy`, `EntityFacadeImpl.groovy`, `EntityFindBuilder.groovy`.
- `/service/`: Contains `ServiceFacadeImpl.groovy`, `ServiceCallSyncImpl.groovy`.

If you ever wonder "how does this work under the hood?", you open these files and read the framework's own code!

---

## CHAPTER 5: Deep Dive Deconstruction (The Q1 SQL Script)

Let's apply Chapter 4 to your specific assignment (`A1Q01NewCustomers`). You were asked to output the exact SQL string that Moqui generates. 

By reading `/moqui-framework/framework/src/main/groovy/org/moqui/impl/entity/EntityFindBase.groovy`, we discover that before Moqui runs a query, it uses an internal class called `EntityFindBuilder` to generate the SQL string. Let's hijack it.

### Step 1: Import the Internal Engine Classes
```groovy
import org.moqui.impl.entity.EntityFindBase
import org.moqui.impl.entity.EntityDefinition
import org.moqui.impl.entity.condition.EntityConditionImplBase
import org.moqui.impl.entity.FieldInfo
import org.moqui.impl.entity.EntityFindBuilder
```
Notice we are bypassing the public `org.moqui.entity.*` packages and directly importing `org.moqui.impl.*`.

### Step 2: The Typecast
```groovy
def activeProducts = ec.entity.find("co.hotwax.upreti.A1Q01NewCustomers")
EntityFindBase efb = (EntityFindBase) activeProducts
```
We take the public `activeProducts` object and forcefully cast it to `EntityFindBase`. This unlocks access to internal variables like `efb.fieldsToSelect`, `efb.limit`, and `efb.offset`.

### Step 3: Extract Metadata and Conditions
```groovy
// Gets the XML metadata (Table names, Joins, Column names)
EntityDefinition ed = efb.getEntityDef()

// Compiles all Moqui conditions (<date-filter>, maps) into a Java Condition Tree
EntityConditionImplBase whereCondition = (EntityConditionImplBase) efb.getWhereEntityConditionInternal(ed)
```

### Step 4: Formatting the Field List
```groovy
List<String> selectFields = efb.fieldsToSelect
def fieldInfoList = []
if (selectFields) {
    selectFields.each { fname ->
        FieldInfo fi = ed.getFieldInfo(fname)
        if (fi) fieldInfoList << fi
    }
} else {
    fieldInfoList.addAll(ed.entityInfo.allFieldInfoArray.findAll { it != null })
}
FieldInfo[] fieldInfoArray = fieldInfoList.toArray(new FieldInfo[0])
```
The SQL Builder doesn't understand simple string lists `['partyId', 'firstName']`. It requires an array of `FieldInfo` objects, which contain the physical database column mappings. We translate the strings into `FieldInfo` objects here.

### Step 5: Manually Driving the SQL Engine
```groovy
// Instantiate the Engine just like the framework does internally
EntityFindBuilder efBuilder = new EntityFindBuilder(ed, efb, whereCondition, fieldInfoArray)

// Command the Engine to build the string chunk by chunk
efBuilder.makeSqlSelectFields(fieldInfoArray, null, "true" == efBuilder.efi.getDatabaseNode(ed.groupName)?.attribute("add-unique-as"))
efBuilder.makeSqlFromClause()
efBuilder.makeWhereClause()
efBuilder.makeGroupByClause()

// Handle HAVING, ORDER BY, and LIMIT manually
EntityConditionImplBase havingCondition = (EntityConditionImplBase) efb.havingEntityCondition
if (havingCondition) efBuilder.makeHavingClause(havingCondition)

List<String> orderBy = efb.orderByFields?:[]
boolean hasLimit = (efb.limit != null || efb.offset != null)
efBuilder.makeOrderByClause(orderBy, hasLimit)
if(hasLimit) efBuilder.addLimitOffset(efb.limit, efb.offset)
```
Instead of calling `.list()`, we take the steering wheel. We instantiate `EntityFindBuilder` and manually call its string-generation methods in exact order.

### Step 6: Extract the Payload
```groovy
// Extract the raw string from the StringBuilder
String sqlText = efBuilder.sqlTopLevel.toString()

// Extract the bind parameters that replace the '?' in the SQL
List paramValues = efBuilder.parameters.collect { p -> p.getValue() }

ec.logger.info("=== Rendered SQL ===\n${sqlText}")
```

---

## CHAPTER 6: Real Codebase Reference Guide

To see how senior engineers use these advanced concepts in the actual codebase, review the following files:

1. **`oms/service/co/hotwax/webhook/WebhookServices.xml`**
   - Use `find` or `ctrl+f` for `<script>` in this file. 
   - **Why it's important:** It shows how to use Groovy to import raw Java Cryptography (`javax.crypto.Mac`) to generate HMAC signatures for Shopify. This is a perfect example of doing something XML simply cannot do.
2. **`oms/service/co/hotwax/orderledger/order/TestTransferOrderServices.xml`**
   - **Why it's important:** It shows massive Groovy `<script>` blocks used for complex List and Map manipulation during automated testing.
3. **`mantle-usl` (If installed in your framework)**
   - The entire mantle component heavily relies on combining basic XML DSL with complex inline Groovy scripts for calculating tax, managing inventory reservations, and running financial ledgers.

## Final Summary
You are now armed with:
1. The standard **XML DSL** for 90% of your daily work.
2. The **`ec` object** for powerful Groovy scripting.
3. The **Hacking Methodology** (Typecasting to `org.moqui.impl.*`) for when you need to break the rules and access the framework's core engine.
