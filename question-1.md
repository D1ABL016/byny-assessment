# Question 1 Solution

## 📌 Assumptions

1. If a SKU already exists, we should not create a new product but update the product and Inventory.
2. Inventory should be maintained per `(product, warehouse)` pair.
3. The API currently does not have a global exception handler.
4. The API is publicly accessible unless authentication is enforced.
5. Inventory should increase if the same SKU is added again for a warehouse.
6. Missing fields should not crash the API and should have safe fallbacks.

---

## 🚨 Potential Issues

1. Instead of directly accessing the keys using data[‘name’] , we should use .get(‘name’,None) so that it does not throw a key error.
2. If we won’t use .get if any key is optional and client did not gave an input for that it would not be having any fallback value.
3. Instead of storing warehouse  id in the product table , we should store it in the Inventory table only.
4. SKU should be a unique column in the inventory table.
5. No check that any product exists with the same sku or not . if exists , should throw an error.
6. We are committing 2 times in same api , due to which if any issue occur while saving inventory ,whole transaction wont be rollbacked as we already committed till product creation.
7. Missing Authentication and Authorization, due to which anyone with URL could create new Products and Inventory.
8. No try catch block is present inside the api.
9. Instead of creating or updating based on product and warehouse id in the Inventory table , it's always creating but it should update the count if SKU already exists.

---

## Corrected Code

```python
from flask import request, jsonify
from sqlalchemy.exc import SQLAlchemyError
from flask_jwt_extended import jwt_required

@app.route('/api/products', methods=['POST'])
@jwt_required()
def create_product():
    try:
        data = request.json or {}

        name = data.get('name')
        sku = data.get('sku')
        price = data.get('price')
        warehouse_id = data.get('warehouse_id')
        quantity = data.get('initial_quantity', 0)

        if not sku or not warehouse_id:
            return jsonify({"error": "sku and warehouse_id are required"}), 400

        product = Product.query.filter_by(sku=sku).first()

        if product:
            inventory = Inventory.query.filter_by(
                product_id=product.id,
                warehouse_id=warehouse_id
            ).first()

            if inventory:
                inventory.quantity += quantity
            else:
                inventory = Inventory(
                    product_id=product.id,
                    warehouse_id=warehouse_id,
                    quantity=quantity
                )
                db.session.add(inventory)

        else:
            if not name or price is None:
                return jsonify({"error": "name and price required"}), 400

            product = Product(
                name=name,
                sku=sku,
                price=price
            )

            db.session.add(product)
            db.session.flush()

            inventory = Inventory(
                product_id=product.id,
                warehouse_id=warehouse_id,
                quantity=quantity
            )
            db.session.add(inventory)

        db.session.commit()

        return jsonify({
            "message": "Success",
            "product_id": product.id
        }), 200

    except SQLAlchemyError as e:
        db.session.rollback()
        return jsonify({"error": str(e)}), 500

    except Exception as e:
        db.session.rollback()
        return jsonify({"error": str(e)}), 500
```

## CORRECTED DB SCHEMA

```python
class Product(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String, nullable=False)
    sku = db.Column(db.String, unique=True, nullable=False)
    price = db.Column(db.Numeric(10, 2), nullable=False)

class Inventory(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    product_id = db.Column(db.Integer, db.ForeignKey('product.id'), nullable=False)
    warehouse_id = db.Column(db.Integer, db.ForeignKey('warehouse.id'), nullable=False)
    quantity = db.Column(db.Integer, default=0)

    __table_args__ = (
        db.UniqueConstraint('product_id', 'warehouse_id', name='unique_product_warehouse'),
    )
```