# Complete Product Sync Story

## The mental model
```
Shopify Product JSON
        │
        ▼
Create Virtual Product
        │
        ▼
Create Variants
        │
        ▼
Link Parent ↔ Variant
(ProductAssoc)
        │
        ▼
Store External IDs
(GoodIdentification)
        │
        ▼
Create Features
(Size, Color, Brand, Tags)
        │
        ▼
Apply Features
(ProductFeatureAppl)
        │
        ▼
Assign Categories
(ProductCategoryMember)
        │
        ▼
Create Prices
(ProductPrice)
        │
        ▼
Create Inventory
(InventoryItem)
        │
        ▼
Store Shop Mapping
(ShopifyShopProduct)
        │
        ▼
Product Ready for Selling
```
---

## Step 1. Shopify sends Product data

For one product, Shopify sends:
- Product
- Variants
- SKU
- Barcode
- Price
- Inventory
- Collections
- Options (Color, Size)
- Vendor
- Tags
- Images

---

## Step 2. Create the Virtual Product

The first record created is the Virtual Product.

**Why?**
Because every variant needs a parent.

----------------------------

This is not the item customers buy.
It is simply the parent product.

**Stored in Product**

```sql
SELECT * FROM product
WHERE IS_VIRTUAL='Y'
```

- productId
- productName
- internalName
- description
- isVirtual = Y
- productTypeId = FINISHED_GOOD

```sql
SELECT * FROM Good_Identification WHERE GOOD_IDENTIFICATION_TYPE_ID='SHOPIFY_PROD_ID'
```

- SHOPIFY_PROD_ID

---

## Step 3. Create every Variant

```sql
SELECT * FROM product
WHERE IS_VARIANT='Y'
```

- Inventory
- Price
- Orders
- Shipments

# Everything points to these variants

## Step 4. Link Parent and Child

```sql
SELECT * FROM product_assoc ;
```

- Link Parent and Child

---

## Step 5. Store External Identifiers

```sql
SELECT * FROM good_identification ;
```

- Shopify Product ID
- Shopify Variant ID
- SKU
- Barcode
- Handle
- Legacy Shopify ID

```sql
SELECT DISTINCT GOOD_IDENTIFICATION_TYPE_ID FROM GOOD_IDENTIFICATION
```

```sql
SELECT * FROM good_identification 
WHERE good_identification_type_ID='SKU';
```

- ERP_ID
- GTIN
- NETSUITE_PRODUCT_ID
- NETSUITE_SALES_DESC
- SHOPIFY_PROD_ID
- SHOPIFY_PROD_SKU
- SKU
- UPCA

Without this table,
future updates would be impossible.

---

## Step 6. Create Features

```sql
SELECT * FROM product_feature;
```

------SH..-----
- Options
- Size
- Color

-------OMS---------
- ProductFeatureType
- ↓
- SIZE
- COLOR

--------AND---------
- ProductFeature
- ↓
- Red
- Blue
- Small
- Medium  // These values are reusable. //Thousands of products can reference it.

---

## Step 7. Apply Features

Now features are attached.

```sql
SELECT * from product_feature_appl 
```

- STANDARD, OR SELECTABLE(DROPDOWN)
- Blue Medium Variant

---

## Step 8. Create Categories

Collections from Shopify become
ProductCategory

```sql
SELECT * FROM product_category ;
SELECT * FROM product_category_member;
```

- LINKS
- Product // Winter Collection
- ↓
- Category //Organic Hoodie

---

## Step 9. Create Prices

Each variant receives:
- DEFAULT_PRICE
- LIST_PRICE
- AVERAGE_COST

```sql
SELECT * FROM PRODUCT_PRICE  -- Prices belong to variants**
```

---

## Step 10. Create Inventory

Inventory belongs only to physical variants.

```sql
SELECT * FROM inventory_item 
```

- Inventory Item ID
- Quantity
- Requires Shipping
- Tracked

Then links inventory to the warehouse.
No inventory is ever maintained for Virtual Product**

---

## Step 11. Save Store Mapping

```sql
SELECT * FROM shopify_shop_product;
```

TELLS OMS:
- This Shopify Store owns
- this HotWax Product.

---

## One more thing I'd prepare

one-line purpose.. For each entity:

| Entity | Why does it exist? |
|---|---|
| Product | Stores the core product/variant record. |
| ProductAssoc | Builds the parent–variant relationship. |
| GoodIdentification | Maps external identifiers (Shopify IDs, SKU, barcode) to internal products. |
| ProductFeature | Defines reusable characteristics like Red or XL. |
| ProductFeatureAppl | Attaches features to products or variants. |
| ProductCategoryMember | Places a product into one or more categories. |
| ProductPrice | Stores time-based pricing information. |
| InventoryItem | Tracks physical stock for sellable variants. |
| ShopifyShopProduct | Maps a Shopify store/product to the corresponding OMS product. |

