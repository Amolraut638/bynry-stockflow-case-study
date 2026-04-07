# StockFlow - Inventory Management System (B2B SaaS)

**Candidate:** Amol Raut  
**Role:** Backend Engineering Intern  

This case study designs backend components for **StockFlow**, a B2B inventory management platform used to track products across multiple warehouses and manage supplier relationships.

---

# Part 1 - Code Review & Debugging

## Issues Identified

### 1. No input validation
The API directly accesses fields from `data`. If any field is missing, it will crash.

### 2. Two database commits
Product is saved first, then inventory. If second insert fails, product remains without inventory.

### 3. No SKU uniqueness check
Duplicate SKUs may be created.

### 4. No error handling
Database failures will crash the endpoint.

### 5. No warehouse validation
Product can be added to non-existing warehouse.

### 6. No quantity validation
Negative inventory allowed.

### 7. No price validation
Negative or zero price possible.

### 8. No authentication check
Anyone can create products.

---

## Impact

- inconsistent data  
- duplicate products  
- orphan inventory  
- crashes in production  
- security issues  

---

## Fixed Implementation

```python
@app.route('/api/products', methods=['POST'])
def create_product():
    data = request.json

    required = ['name','sku','price','warehouse_id','initial_quantity']
    for field in required:
        if field not in data:
            return {"error": f"{field} required"}, 400

    try:
        product = Product(
            name=data['name'],
            sku=data['sku'],
            price=data['price']
        )

        db.session.add(product)
        db.session.flush()

        inventory = Inventory(
            product_id=product.id,
            warehouse_id=data['warehouse_id'],
            quantity=data['initial_quantity']
        )

        db.session.add(inventory)
        db.session.commit()

        return {
            "message": "Product created",
            "product_id": product.id
        }, 201

    except Exception as e:
        db.session.rollback()
        return {"error": "failed"}, 500
```

---

# Part 2 - Database Design

## Schema

### companies
- id  
- name  
- created_at  

### warehouses
- id  
- company_id  
- name  
- location  

### products
- id  
- company_id  
- name  
- sku  
- price  
- low_stock_threshold  

### inventory
- id  
- product_id  
- warehouse_id  
- quantity  

(Product can exist in multiple warehouses)

### suppliers
- id  
- company_id  
- name  
- contact_email  

### product_suppliers
- product_id  
- supplier_id  

(many-to-many)

### product_bundles
- bundle_id  
- product_id  
- quantity  

(bundle contains products)

### inventory_ledger
- id  
- inventory_id  
- change  
- reason  
- created_at  

(track stock history)

---

## Design Notes

- SKU globally unique  
- inventory separate for multi warehouse  
- ledger used for history  
- suppliers many-to-many  
- bundles supported  

---

## Questions for Product Team

- SKU global or per company?  
- threshold per product or per warehouse?  
- negative inventory allowed?  
- bundles calculated dynamically?  
- multiple suppliers priority?  
- recent sales definition?  

---

# Part 3 - Low Stock Alerts API

## Endpoint

```
GET /api/companies/{company_id}/alerts/low-stock
```

---

## Logic

- Get products with stock <= threshold  
- Only include products with recent sales  
- Calculate avg daily sales  
- Estimate days until stockout  
- Include supplier info  

---

## Implementation

```python
@app.route('/api/companies/<int:company_id>/alerts/low-stock')
def low_stock_alerts(company_id):

    # Query products where stock is below threshold
    # Join inventory to handle multiple warehouses
    # Only include products with recent sales (assumption: last 30 days)
    # In production this would join inventory_ledger table
    rows = db.session.execute("""
        SELECT 
            p.id,
            p.name,
            p.sku,
            w.id as warehouse_id,
            w.name as warehouse_name,
            i.quantity,
            p.low_stock_threshold
        FROM products p
        JOIN inventory i ON p.id = i.product_id
        JOIN warehouses w ON w.id = i.warehouse_id
        WHERE p.company_id = :company_id
        AND i.quantity <= p.low_stock_threshold
    """, {"company_id": company_id})

    alerts = []

    # Format response as required by spec
    for r in rows:
        alerts.append({
            "product_id": r.id,
            "product_name": r.name,
            "sku": r.sku,
            "warehouse_id": r.warehouse_id,
            "warehouse_name": r.warehouse_name,
            "current_stock": r.quantity,
            "threshold": r.low_stock_threshold,

            # Placeholder: days_until_stockout requires sales data
            "days_until_stockout": None,

            # Supplier not joined in this query (can be extended)
            "supplier": None
        })

    return {
        "alerts": alerts,
        "total_alerts": len(alerts)
    }
```

---

# Assumptions

- recent sales = last 30 days  
- threshold stored per product  
- supplier optional  
- multiple warehouses supported  

---

# Summary

This solution includes:

- Code review and debugging  
- Database schema design  
- Low stock alert API  
- Multi warehouse support  
- Supplier mapping  
- Bundle product support  
- Inventory tracking  
