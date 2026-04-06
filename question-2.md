# Question 2 Solutions

---

## Database Schemas

### Company

```sql
CREATE TABLE company (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_company_name ON company(name);
```

---

### Warehouses

```sql
CREATE TABLE warehouse (
    id INT AUTO_INCREMENT PRIMARY KEY,
    company_id INT NOT NULL,
    name VARCHAR(255) NOT NULL,
    max_capacity INT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (company_id) REFERENCES company(id) ON DELETE CASCADE
);

CREATE INDEX idx_warehouse_company ON warehouse(company_id);
```

---

### Products

```sql
CREATE TABLE product (
    id INT AUTO_INCREMENT PRIMARY KEY,
    company_id INT NOT NULL,
    name VARCHAR(255) NOT NULL,
    sku VARCHAR(100) NOT NULL,
    price NUMERIC(10,2),
    is_bundle BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (company_id) REFERENCES company(id) ON DELETE CASCADE,
    UNIQUE (company_id, sku)
);

CREATE INDEX idx_products_sku ON product(sku);
```

---

### Bundle Mapping (Product → Sub-products)

```sql
CREATE TABLE product_bundles (
    id INT AUTO_INCREMENT PRIMARY KEY,
    parent_product_id INT NOT NULL,
    child_product_id INT NOT NULL,
    quantity INT NOT NULL CHECK (quantity > 0),

    FOREIGN KEY (parent_product_id) REFERENCES product(id) ON DELETE CASCADE,
    FOREIGN KEY (child_product_id) REFERENCES product(id) ON DELETE CASCADE,

    UNIQUE (parent_product_id, child_product_id)
);

CREATE INDEX idx_bundle_parent ON product_bundles(parent_product_id);
```

---

### Suppliers

```sql
CREATE TABLE suppliers (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    contact_info TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_suppliers_name ON suppliers(name);
```

---

### Inventory

```sql
CREATE TABLE inventory (
    id INT AUTO_INCREMENT PRIMARY KEY,
    product_id INT NOT NULL,
    warehouse_id INT NOT NULL,
    quantity INT NOT NULL DEFAULT 0,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (product_id) REFERENCES product(id) ON DELETE CASCADE,
    FOREIGN KEY (warehouse_id) REFERENCES warehouse(id) ON DELETE CASCADE,
    UNIQUE (product_id, warehouse_id)
);

CREATE INDEX idx_inventory_product_warehouse 
ON inventory(product_id, warehouse_id);
```

---

### Supplier-Product Mapping

```sql
CREATE TABLE supplier_products (
    id INT AUTO_INCREMENT PRIMARY KEY,
    supplier_id INT NOT NULL,
    product_id INT NOT NULL,
    supplier_price NUMERIC(10,2),
    lead_time_days INT,
    FOREIGN KEY (supplier_id) REFERENCES suppliers(id) ON DELETE CASCADE,
    FOREIGN KEY (product_id) REFERENCES product(id) ON DELETE CASCADE,
    UNIQUE (supplier_id, product_id)
);

CREATE INDEX idx_supplier_products_supplier 
ON supplier_products(supplier_id);
```

---

### Inventory Audit

```sql
CREATE TABLE inventory_audit (
    id INT AUTO_INCREMENT PRIMARY KEY,
    product_id INT,
    warehouse_id INT,
    old_quantity INT,
    new_quantity INT,
    change_type VARCHAR(20),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_inventory_audit_product 
ON inventory_audit(product_id);
```

---

## Trigger for Inventory Audit

```sql
DELIMITER $$

CREATE TRIGGER trg_inventory_update
AFTER UPDATE ON inventory
FOR EACH ROW
BEGIN
    INSERT INTO inventory_audit (
        product_id,
        warehouse_id,
        old_quantity,
        new_quantity,
        change_type
    )
    VALUES (
        OLD.product_id,
        OLD.warehouse_id,
        OLD.quantity,
        NEW.quantity,
        CASE 
            WHEN NEW.quantity > OLD.quantity THEN 'INCREASE'
            WHEN NEW.quantity < OLD.quantity THEN 'DECREASE'
            ELSE 'NO_CHANGE'
        END
    );
END$$

DELIMITER ;
```

---

## Assumptions

### Core Assumptions

1. Each company owns its own products and warehouses.
2. SKU is unique **per company**, not globally.
3. A product can exist in multiple warehouses.
4. Inventory is tracked per `(product_id, warehouse_id)`.
5. Inventory updates happen via UPDATE, not repeated INSERT.
6. Suppliers can supply multiple products and vice versa.
7. Inventory quantity is assumed to be non-negative.
8. Audit logs are only created on UPDATE.
9. No user tracking for inventory changes.
10. No reservation system is implemented.

---

### Bundle-Specific Assumptions

11. A product can be a bundle containing multiple products.
12. A bundle can contain the **same product multiple times**.
13. A product can technically reference itself (no restriction added).
14. Bundle expansion logic (deducting child inventory) is handled at application level.
15. No limit on bundle size or nesting depth.

---

## Index Justification

| Table             | Index                      | Reason                                     |
| ----------------- | -------------------------- | ------------------------------------------ |
| company           | name                       | Fast lookup/filtering by company name      |
| warehouse         | company_id                 | Fetch warehouse per company efficiently   |
| product           | sku                        | Frequent lookup by SKU                     |
| product_bundles   | parent_product_id          | Quickly fetch all sub-products of a bundle |
| suppliers         | name                       | Search suppliers by name                   |
| inventory         | (product_id, warehouse_id) | Most common query: stock lookup            |
| supplier_products | supplier_id                | Fetch all products for a supplier          |
| inventory_audit   | product_id                 | Retrieve audit history for a product       |

---

## Relationship Summary

* Company → Warehouses → One-to-Many
* Company → Products → One-to-Many
* Products ↔ Warehouses → Many-to-Many (Inventory)
* Suppliers ↔ Products → Many-to-Many
* Products ↔ Products → Self-referencing (Bundles)

