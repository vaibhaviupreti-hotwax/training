# How to Write a Moqui Service — Complete Guide

> Based on the `moqui-training-mentor` component at:
> `runtime/component/moqui-training-mentor/`

---

## 1. Component Structure (What Files You Need)

```
your-component/
├── component.xml              ← registers the component with Moqui
├── MoquiConf.xml              ← component-level config overrides (can be empty)
├── entity/
│   └── YourEntities.xml       ← entity definitions (your "tables")
├── service/
│   ├── YourServices.xml       ← XML service definitions (inline actions)
│   ├── YourGroovyService.xml  ← XML wrapper for Groovy script services
│   ├── YourService.groovy     ← Groovy implementation file
│   └── yourServices.rest.xml  ← REST API definitions
└── data/
    └── YourDemoData.xml       ← seed/demo data
```

---

## 2. Prerequisites Before Writing a Service

### a) `component.xml` — Register Your Component

```xml
<?xml version="1.0" encoding="UTF-8"?>
<component xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:noNamespaceSchemaLocation="http://moqui.org/xsd/moqui-conf-3.xsd"
           name="moqui-training" version="${moqui_version}">
</component>
```

> [!IMPORTANT]
> The `name` attribute here determines how Moqui discovers your component. It must match your directory name.

### b) `MoquiConf.xml` — Minimal Config

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<moqui-conf xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xsi:noNamespaceSchemaLocation="http://moqui.org/xsd/moqui-conf-3.xsd">
</moqui-conf>
```

### c) Entity Definition — You Need an Entity First

From [MoquiTraining.xml](file:///home/vaibhaviupreti/1-HW/all-here/6-local-maarg-new/moqui-framework/runtime/component/moqui-training-mentor/entity/MoquiTraining.xml#L1-L10):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<entities xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:noNamespaceSchemaLocation="http://moqui.org/xsd/entity-definition-3.xsd">
    <entity entity-name="MoquiTraining" package="moquitraining">
        <field name="trainingId" type="id" is-pk="true"/>
        <field name="trainingName" type="text-medium"/>
        <field name="trainingDate" type="date"/>
        <field name="trainingPrice" type="number-decimal"/>
        <field name="trainingDuration" type="number-integer"/>
    </entity>
</entities>
```

> [!NOTE]
> **Field types available**: `id`, `text-short`, `text-medium`, `text-long`, `text-very-long`, `date`, `date-time`, `number-integer`, `number-decimal`, `number-float`, `currency-amount`, `currency-precise`

---

## 3. The Service XML File — Root Structure

Every service file starts with:

```xml
<services xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:noNamespaceSchemaLocation="http://moqui.org/xsd/service-definition-3.xsd">

    <!-- your services go here -->

</services>
```

The XSD reference (`service-definition-3.xsd`) is **required** — it validates your XML.

---

## 4. Three Types of Services

### Type 1: Inline XML Service (Most Common)

Logic written directly in `<actions>` using Moqui's XML DSL.

From [TrainingServices.xml](file:///home/vaibhaviupreti/1-HW/all-here/6-local-maarg-new/moqui-framework/runtime/component/moqui-training-mentor/service/TrainingServices.xml#L2-L22):

```xml
<service verb="find" noun="MoquiTraining">
    <in-parameters>
        <parameter name="trainingName" />
        <parameter name="trainingId" />
    </in-parameters>
    <out-parameters>
        <parameter name="trainingList"/>
    </out-parameters>
    <actions>
        <entity-find entity-name="moquitraining.MoquiTraining" list="trainingList">
            <econditions combine="or">
                <econdition field-name="trainingId" value="${trainingId}" ignore-if-empty="true"/>
                <econdition field-name="trainingName" operator="like" value="%${trainingName}%"
                            ignore-if-empty="true"/>
            </econditions>
            <select-field field-name="trainingId"/>
            <select-field field-name="trainingName"/>
            <select-field field-name="trainingDate"/>
            <order-by field-name="trainingName"/>
        </entity-find>
    </actions>
</service>
```

### Type 2: Entity-Auto Service (Zero-Code CRUD)

Moqui generates the implementation automatically. Just specify `type="entity-auto"`:

```xml
<service verb="create" noun="Party" type="entity-auto">
    <in-parameters>
        <parameter name="partyType" required="true"/>
        <parameter name="firstName" required="true"/>
        <parameter name="lastName" required="true" />
    </in-parameters>
    <out-parameters>
        <auto-parameters entity-name="Party" required="true"/>
    </out-parameters>
</service>
```

> [!TIP]
> `entity-auto` works with verbs: `create`, `update`, `delete`, `store`. The entity name is derived from the service `noun`.

### Type 3: Groovy Script Service (External Logic)

Two files needed:

**a) XML wrapper** — [MoquiTrainingGroovyService.xml](file:///home/vaibhaviupreti/1-HW/all-here/6-local-maarg-new/moqui-framework/runtime/component/moqui-training-mentor/service/MoquiTrainingGroovyService.xml#L1-L16):

```xml
<services xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:noNamespaceSchemaLocation="http://moqui.org/xsd/service-definition-3.xsd">

    <service verb="create" noun="MoquiTraining" type="script"
             location="component://moqui-training/service/MoquiTrainingService.groovy">
        <in-parameters>
            <auto-parameters entity-name="moquitraining.MoquiTraining"/>
        </in-parameters>
        <out-parameters>
            <auto-parameters entity-name="moquitraining.MoquiTraining" include="pk" required="true"/>
        </out-parameters>
    </service>
</services>
```

**b) Groovy file** — [MoquiTrainingService.groovy](file:///home/vaibhaviupreti/1-HW/all-here/6-local-maarg-new/moqui-framework/runtime/component/moqui-training-mentor/service/MoquiTrainingService.groovy):

```groovy
if (true) {
    def call_service_result = ec.service.sync().name("create#moquitraining.MoquiTraining")
            .parameters(context).call()
    if (context != null) {
        if (call_service_result) context.putAll(call_service_result)
    } else {
        context = call_service_result
    }
    if (ec.message.hasError()) return
}
// make sure the last statement is not considered the return value
return;
```

> [!IMPORTANT]
> - `type="script"` tells Moqui to delegate to an external file
> - `location` uses the `component://component-name/...` URI format
> - In the Groovy file, `ec` (ExecutionContext) and `context` (parameter map) are automatically available

---

## 5. Service Naming Rules

### The `verb#noun` Convention

| Attribute | Rule | Examples |
|-----------|------|----------|
| `verb` | **Action being performed** — use standard verbs | `create`, `find`, `store`, `update`, `delete`, `get`, `test` |
| `noun` | **Entity or concept being acted on** — PascalCase | `MoquiTraining`, `Party`, `OrderHeader` |
| Full name | `verb#noun` | `create#MoquiTraining`, `find#MoquiTraining` |

### How Moqui Resolves the Service Name

The **file name** determines the service's namespace prefix. For a file named `TrainingServices.xml`:

```
TrainingServices.find#MoquiTraining
└── file name    └── verb#noun
```

When calling the service:
```xml
<service-call name="TrainingServices.create#MoquiTraining" .../>
```

For **implicit entity services** (no XML definition needed), use:
```xml
<service-call name="create#moquitraining.MoquiTraining" .../>
```
This calls Moqui's built-in CRUD on the entity directly.

---

## 6. Parameters — Rules & Patterns

### In-Parameters

```xml
<in-parameters>
    <!-- Manual parameter definition -->
    <parameter name="trainingDate" required="true"/>
    <parameter name="trainingName" required="true"/>

    <!-- Auto-pull ALL fields from entity -->
    <auto-parameters entity-name="moquitraining.MoquiTraining"/>

    <!-- Auto-pull only PK fields -->
    <auto-parameters entity-name="moquitraining.MoquiTraining" include="pk"/>

    <!-- Auto-pull only non-PK fields -->
    <auto-parameters entity-name="moquitraining.MoquiTraining" include="nonpk"/>

    <!-- With exclusions -->
    <auto-parameters include="nonpk" required="true">
        <exclude field-name="lastUpdatedStamp"/>
    </auto-parameters>

    <!-- Complex nested parameter (List of Maps) -->
    <parameter name="listOfItems" type="List" required="true">
        <parameter name="orderItem" type="Map">
            <parameter name="productId" required="true"/>
            <parameter name="quantity" type="Integer" default-value="1">
                <number-range min="1"/>
            </parameter>
            <parameter name="status" default-value="Placed"/>
        </parameter>
    </parameter>

    <!-- Pagination parameters -->
    <parameter name="pageSize" type="Integer" default-value="20"/>
    <parameter name="pageIndex" type="Integer" default-value="0"/>
</in-parameters>
```

### Out-Parameters

```xml
<out-parameters>
    <!-- Simple field -->
    <parameter name="trainingId"/>

    <!-- Return a list -->
    <parameter name="trainingList"/>

    <!-- Auto-parameters for entity -->
    <auto-parameters entity-name="moquitraining.MoquiTraining"/>

    <!-- Auto-parameters for PK only -->
    <auto-parameters entity-name="moquitraining.MoquiTraining" include="pk" required="true"/>
</out-parameters>
```

> [!WARNING]
> The `out-parameter` name must exactly match the variable name you `set` in `<actions>`. If the entity-find uses `list="trainingList"`, your out-parameter must be `name="trainingList"`.

---

## 7. Actions — The XML DSL Cheat Sheet

### Entity Operations

```xml
<!-- FIND multiple records -->
<entity-find entity-name="moquitraining.MoquiTraining" list="trainingList">
    <econdition field-name="trainingId" value="${trainingId}" ignore-if-empty="true"/>
    <econdition field-name="trainingName" operator="like" value="%${trainingName}%"
                ignore-if-empty="true"/>
    <select-field field-name="trainingId"/>
    <order-by field-name="trainingName"/>
</entity-find>

<!-- FIND ONE record by PK -->
<entity-find-one entity-name="MoquiTraining" value-field="existingRecord">
    <field-map field-name="trainingId" from="trainingId"/>
</entity-find-one>

<!-- UPDATE a record -->
<set field="existingRecord.trainingName" from="trainingName"/>
<entity-update value-field="existingRecord"/>

<!-- DELETE a record -->
<entity-delete value-field="product"/>

<!-- DELETE related records -->
<entity-delete-related value-field="orderHeader" relationship-name="items"/>
```

### Combining Conditions

```xml
<!-- OR conditions -->
<econditions combine="or">
    <econdition field-name="trainingId" value="${trainingId}" ignore-if-empty="true"/>
    <econdition field-name="trainingName" operator="like" value="%${name}%" ignore-if-empty="true"/>
</econditions>

<!-- AND is the default -->
<econditions>
    <econdition field-name="partyId" from="partyId"/>
    <econdition field-name="contactMechId" from="contactMechId"/>
</econditions>
```

### Condition Operators

| Operator | Usage |
|----------|-------|
| `equals` (default) | `<econdition field-name="statusId" value="ACTIVE"/>` |
| `like` | `<econdition ... operator="like" value="%search%"/>` |
| `not-equals` | `<econdition ... operator="not-equals" value="SHIPPED"/>` |
| `greater-equals` | `<econdition ... operator="greater-equals" value="2023-01-01"/>` |
| `less-equals` | `<econdition ... operator="less-equals" value="2023-12-31"/>` |
| `is-not-null` | `<econdition field-name="changeReason" operator="is-not-null"/>` |

### Control Flow

```xml
<!-- If-Then-Else -->
<if condition="existingRecord">
    <then>
        <set field="existingRecord.trainingDate" from="trainingDate"/>
        <entity-update value-field="existingRecord"/>
    </then>
    <else>
        <service-call name="create#MoquiTraining" in-map="context" out-map="context"/>
    </else>
</if>

<!-- Iterate over a list -->
<iterate list="orderItems" entry="orderItem">
    <service-call name="delete#OrderItem" in-map="orderItem"/>
</iterate>

<!-- Set a field -->
<set field="trainingId" from="existingRecord.trainingId"/>

<!-- Return early with a message -->
<return message="No such Contact exists"/>

<!-- Log for debugging -->
<log level="warn" message="Debug: ${contactMech}"/>
```

### Calling Other Services

```xml
<!-- Call an implicit entity service -->
<service-call name="create#moquitraining.MoquiTraining" in-map="context" out-map="context"/>

<!-- Call a defined service -->
<service-call name="TrainingServices.create#Party" in-map="context" out-map="mp"/>

<!-- Use returned values -->
<set field="item.orderId" value="${mp.orderId}"/>
```

### Pagination

```xml
<entity-find entity-name="co.hotwax.party.party.PartyNameAndRoleDetail"
             list="parties" limit="pageSize" offset="pageSize * pageIndex">
    <econdition field-name="partyId" ignore-if-empty="true"/>
</entity-find>
<set field="count" from="parties_xafind.count()"/>
```

> [!NOTE]
> `_xafind` suffix gives you access to the find object for `.count()` and other metadata.

---

## 8. Interface Services (Reusable Contracts)

Define a common output contract that multiple services can implement:

```xml
<!-- Define the interface -->
<service verb="test" noun="SqlAssignment" type="interface">
    <out-parameters>
        <parameter name="entityValueList" type="List" />
    </out-parameters>
</service>

<!-- Implement it -->
<service verb="get" noun="A1Q01NewCustomers">
    <implements service="TrainingServices.test#SqlAssignment"/>
    <out-parameters>
        <parameter name="firstName"/>
        <parameter name="lastName"/>
    </out-parameters>
    <actions>
        <!-- ... -->
    </actions>
</service>
```

---

## 9. Exposing Services via REST API

From [moqui-trainingServices.rest.xml](file:///home/vaibhaviupreti/1-HW/all-here/6-local-maarg-new/moqui-framework/runtime/component/moqui-training-mentor/service/moqui-trainingServices.rest.xml):

```xml
<resource xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:noNamespaceSchemaLocation="http://moqui.org/xsd/rest-api-3.xsd"
          name="assignment" displayName="REST API Assignment" version="${moqui_version}">

    <resource name="parties" require-authentication="anonymous-all">
        <!-- GET /rest/s1/assignment/parties → list all -->
        <method type="get">
            <entity name="Party" operation="list"/>
        </method>

        <!-- POST /rest/s1/assignment/parties → create -->
        <method type="post">
            <service name="TrainingServices.create#Party"/>
        </method>

        <!-- GET/PUT/DELETE /rest/s1/assignment/parties/{partyId} -->
        <id name="partyId">
            <method type="get">
                <entity name="Party" operation="one"/>
            </method>
            <method type="put">
                <service name="TrainingServices.update#Party"/>
            </method>
            <method type="delete">
                <service name="TrainingServices.delete#Party"/>
            </method>

            <!-- Nested: /rest/s1/assignment/parties/{partyId}/contacts -->
            <resource name="contacts">
                <method type="get">
                    <entity name="ContactMech" operation="list"/>
                </method>
            </resource>
        </id>
    </resource>
</resource>
```

**REST URL Pattern**: `http://localhost:8080/rest/s1/{resource-name}/{sub-resource}`

| Method | URL | Action |
|--------|-----|--------|
| `GET` | `/rest/s1/assignment/parties` | List all parties |
| `POST` | `/rest/s1/assignment/parties` | Create a party |
| `GET` | `/rest/s1/assignment/parties/123` | Get party 123 |
| `PUT` | `/rest/s1/assignment/parties/123` | Update party 123 |
| `DELETE` | `/rest/s1/assignment/parties/123` | Delete party 123 |

---

## 10. Demo/Seed Data

From [MoquiTrainingDemoData.xml](file:///home/vaibhaviupreti/1-HW/all-here/6-local-maarg-new/moqui-framework/runtime/component/moqui-training-mentor/data/MoquiTrainingDemoData.xml):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<entity-facade-xml type="demo">
    <moquitraining.MoquiTraining trainingId="1" trainingName="Java"
        trainingDate="2024-12-12" trainingPrice="12300.00" trainingDuration="3"/>
    <moquitraining.MoquiTraining trainingId="2" trainingName="Spring Boot"
        trainingDate="2024-12-30" trainingPrice="1200.00" trainingDuration="7"/>
</entity-facade-xml>
```

> [!TIP]
> - Use `type="demo"` for test data, `type="seed"` for required configuration data
> - Entity records are written as `<package.EntityName field1="val1" field2="val2"/>`

---

## 11. The store Verb Pattern — Create-or-Update

The `store` verb is a special pattern for "upsert" logic. From [TrainingServices.xml](file:///home/vaibhaviupreti/1-HW/all-here/6-local-maarg-new/moqui-framework/runtime/component/moqui-training-mentor/service/TrainingServices.xml#L39-L80):

```xml
<service verb="store" noun="MoquiTraining">
    <in-parameters>
        <auto-parameters entity-name="MoquiTraining"/>
        <parameter name="trainingId" required="true"/>
        <parameter name="trainingDate" required="true"/>
    </in-parameters>
    <out-parameters>
        <auto-parameters entity-name="MoquiTraining"/>
    </out-parameters>
    <actions>
        <!-- Check if record exists -->
        <entity-find-one entity-name="MoquiTraining" value-field="existingRecord">
            <field-map field-name="trainingId" from="trainingId"/>
        </entity-find-one>

        <if condition="existingRecord">
            <then>
                <!-- Update existing -->
                <set field="existingRecord.trainingDate" from="trainingDate"/>
                <if condition="trainingName">
                    <set field="existingRecord.trainingName" from="trainingName"/>
                </if>
                <entity-update value-field="existingRecord"/>
                <set field="trainingId" from="existingRecord.trainingId"/>
            </then>
            <else>
                <!-- Create new -->
                <service-call name="create#MoquiTraining" in-map="context" out-map="context"/>
            </else>
        </if>
    </actions>
</service>
```

---

## 12. Common Rules & Gotchas

| Rule | Detail |
|------|--------|
| **Entity name must be fully qualified** | Use `moquitraining.MoquiTraining` (package.EntityName) in service actions |
| **XSD is required** | Every XML file needs the correct `xsi:noNamespaceSchemaLocation` |
| **File goes in `service/` dir** | Moqui auto-discovers XML files in the `service/` directory |
| **`in-map="context"`** | Passes all current in-parameters to a sub-service call |
| **`out-map="context"`** | Merges the sub-service's output back into the current context |
| **`ignore-if-empty="true"`** | Makes a condition optional — skipped if the field is null/empty |
| **Variable from `list`** | If `entity-find list="myList"`, the out-parameter must match `myList` |
| **Groovy `return;`** | Always end Groovy scripts with `return;` to avoid accidental return values |
| **REST file naming** | REST XML files should end with `.rest.xml` for Moqui to recognize them |
| **`entity-auto` verb matching** | For `entity-auto`, the verb must be `create`, `update`, `store`, or `delete` |
| **No `<actions>` for `entity-auto`** | Entity-auto services generate their own actions; don't add `<actions>` |
| **No `<actions>` for `interface`** | Interface services define contracts only; no implementation |

---

## 13. Quick-Start Template

Here's a minimal complete example to get you started:

```xml
<services xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:noNamespaceSchemaLocation="http://moqui.org/xsd/service-definition-3.xsd">

    <!-- CREATE -->
    <service verb="create" noun="YourEntity">
        <in-parameters>
            <auto-parameters entity-name="yourpackage.YourEntity" include="nonpk"/>
            <parameter name="requiredField" required="true"/>
        </in-parameters>
        <out-parameters>
            <parameter name="yourEntityId"/>
        </out-parameters>
        <actions>
            <service-call name="create#yourpackage.YourEntity" in-map="context" out-map="context"/>
        </actions>
    </service>

    <!-- FIND -->
    <service verb="find" noun="YourEntity">
        <in-parameters>
            <parameter name="yourEntityId"/>
            <parameter name="searchField"/>
        </in-parameters>
        <out-parameters>
            <parameter name="resultList"/>
        </out-parameters>
        <actions>
            <entity-find entity-name="yourpackage.YourEntity" list="resultList">
                <econdition field-name="yourEntityId" ignore-if-empty="true"/>
                <econdition field-name="searchField" operator="like"
                            value="%${searchField}%" ignore-if-empty="true"/>
            </entity-find>
        </actions>
    </service>

    <!-- UPDATE -->
    <service verb="update" noun="YourEntity">
        <in-parameters>
            <parameter name="yourEntityId" required="true"/>
            <parameter name="fieldToUpdate"/>
        </in-parameters>
        <out-parameters>
            <parameter name="updatedRecord"/>
        </out-parameters>
        <actions>
            <entity-find-one entity-name="yourpackage.YourEntity" value-field="record">
                <field-map field-name="yourEntityId" from="yourEntityId"/>
            </entity-find-one>
            <if condition="fieldToUpdate">
                <set field="record.fieldToUpdate" from="fieldToUpdate"/>
            </if>
            <entity-update value-field="record"/>
            <set field="updatedRecord" from="record"/>
        </actions>
    </service>

    <!-- DELETE -->
    <service verb="delete" noun="YourEntity">
        <in-parameters>
            <auto-parameters entity-name="yourpackage.YourEntity" include="pk" required="true"/>
        </in-parameters>
        <actions>
            <entity-find-one entity-name="yourpackage.YourEntity" value-field="record"/>
            <entity-delete value-field="record"/>
        </actions>
    </service>
</services>
```

---

## Summary Checklist

- [ ] Entity defined in `entity/` directory
- [ ] Service XML in `service/` directory with correct XSD
- [ ] Service has `verb` + `noun` attributes
- [ ] `<in-parameters>` and `<out-parameters>` defined
- [ ] `<actions>` contains the business logic (unless `entity-auto` or `interface`)
- [ ] Entity names fully qualified with package (`package.EntityName`)
- [ ] Out-parameter names match variable names used in actions
- [ ] Demo data in `data/` directory (optional but recommended)
- [ ] REST API in `*.rest.xml` file (if you need HTTP endpoints)
