# Question 3 Solutions

---

## Assumptions

1. Each product has a **low stock threshold** stored in the `products` table (`low_stock_threshold` column assumed).
2. “Recent sales activity” is determined using a `sales` table with `product_id`, `warehouse_id`, `quantity`, `created_at`.
3. Recent = last **30 days** (assumption).
4. `days_until_stockout` is calculated using **average daily sales**.
5. Each product can have **multiple suppliers**, but we return the **primary (first) supplier**.
6. Inventory is stored per `(product_id, warehouse_id)`.
7. some functions are already made which feteches current db session , fetches loggedin user from token and middlewares which restricts unauthorized user to access this endpoint.

---

## Approach

1. Join:

   * `products`
   * `inventory`
   * `warehouses`
   * `supplier_products` + `suppliers`
   * `sales` (for recent activity)

2. Filter:

   * Only products with **recent sales**
   * Only where `inventory.quantity < threshold`

3. Compute:

   * Avg daily sales = total sold / days
   * Days until stockout = current_stock / avg_daily_sales

4. Return structured response

---

## FAST API Implementation
```python
from fastapi import FastAPI, Depends, HTTPException, Query
from sqlalchemy.orm import Session
from sqlalchemy import func
from datetime import datetime, timedelta

app = FastAPI()

@app.get("/api/companies/{company_id}/alerts/low-stock")
def get_low_stock_alerts(
    company_id: int,
    page: int = Query(1, ge=1),
    page_size: int = Query(10, ge=1, le=100),
    db: Session = Depends(get_db)
):
    """
    Returns paginated low stock alerts for a company.
    """

    try:
        recent_days = 30
        start_date = datetime.utcnow() - timedelta(days=recent_days)

        # 🔹 Pagination calculation
        offset = (page - 1) * page_size

        # 🔹 Subquery: recent sales
        sales_subquery = (
            db.query(
                Sales.product_id,
                Sales.warehouse_id,
                func.sum(Sales.quantity).label("total_sold")
            )
            .filter(Sales.created_at >= start_date)
            .group_by(Sales.product_id, Sales.warehouse_id)
            .subquery()
        )

        base_query = (
            db.query(
                Product.id.label("product_id"),
                Product.name.label("product_name"),
                Product.sku,
                Product.low_stock_threshold,
                Warehouse.id.label("warehouse_id"),
                Warehouse.name.label("warehouse_name"),
                Inventory.quantity.label("current_stock"),
                sales_subquery.c.total_sold,
                Supplier.id.label("supplier_id"),
                Supplier.name.label("supplier_name"),
                Supplier.contact_info
            )
            .join(Inventory, Inventory.product_id == Product.id)
            .join(Warehouse, Warehouse.id == Inventory.warehouse_id)
            .join(
                sales_subquery,
                (sales_subquery.c.product_id == Product.id) &
                (sales_subquery.c.warehouse_id == Warehouse.id)
            )
            .outerjoin(SupplierProduct, SupplierProduct.product_id == Product.id)
            .outerjoin(Supplier, Supplier.id == SupplierProduct.supplier_id)
            .filter(Product.company_id == company_id)
            .filter(Inventory.quantity < Product.low_stock_threshold)
        )

        # 🔹 Total count BEFORE pagination
        total_count = base_query.count()

        # 🔹 Apply pagination
        results = (
            base_query
            .limit(page_size)
            .offset(offset)
            .all()
        )

        alerts = []

        for row in results:
            avg_daily_sales = (row.total_sold or 0) / recent_days

            if avg_daily_sales == 0:
                continue

            days_until_stockout = int(row.current_stock / avg_daily_sales)

            alerts.append({
                "product_id": row.product_id,
                "product_name": row.product_name,
                "sku": row.sku,
                "warehouse_id": row.warehouse_id,
                "warehouse_name": row.warehouse_name,
                "current_stock": row.current_stock,
                "threshold": row.low_stock_threshold,
                "days_until_stockout": days_until_stockout,
                "supplier": {
                    "id": row.supplier_id,
                    "name": row.supplier_name,
                    "contact_email": row.contact_info
                } if row.supplier_id else None
            })

        return {
            "alerts": alerts,
            "total_alerts": len(alerts),
            "page": page,
            "page_size": page_size,
            "total_records": total_count
        }

    except Exception as e:
        raise HTTPException(
            status_code=500,
            detail=f"Failed to fetch alerts: {str(e)}"
        )
```


---

## Edge Cases Handled

### 1. No Sales Data

* Skipped (cannot compute stockout)

---

### 2. Division by Zero

* Protected via check on `avg_daily_sales`

---

### 3. Multiple Suppliers

* Returns first (can be extended later)

---

### 4. No Supplier Found

* Returns `"supplier": null`

---

### 5. Missing Threshold

* Assumes `low_stock_threshold` exists (else should default)

### 6. Pagination

* Returns paginated results
* Total count included

---

## Possible Improvements

* Add caching (Redis)
* Add priority scoring (urgent alerts first)
* Support multiple suppliers with ranking


