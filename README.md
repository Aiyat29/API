# API
Разработка API для управления инвентарём
pip install Flask SQLAlchemy Flask-Marshmallow
from flask import Flask, request, jsonify
from flask_sqlalchemy import SQLAlchemy
from flask_marshmallow import Marshmallow

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///inventory.db'
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False
db = SQLAlchemy(app)
ma = Marshmallow(app)

# Модели
class Category(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(100), nullable=False)
    description = db.Column(db.String(255))

    def __init__(self, name, description):
        self.name = name
        self.description = description

class Product(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(100), nullable=False)
    description = db.Column(db.String(255))
    price = db.Column(db.Float, nullable=False)
    in_stock = db.Column(db.Integer, nullable=False)
    category_id = db.Column(db.Integer, db.ForeignKey('category.id'), nullable=False)
    category = db.relationship('Category', backref=db.backref('products', lazy=True))

    def __init__(self, name, description, price, in_stock, category_id):
        self.name = name
        self.description = description
        self.price = price
        self.in_stock = in_stock
        self.category_id = category_id

# Схемы Marshmallow
class CategorySchema(ma.SQLAlchemyAutoSchema):
    class Meta:
        model = Category

class ProductSchema(ma.SQLAlchemyAutoSchema):
    class Meta:
        model = Product

# Создание базы данных
@app.before_first_request
def create_tables():
    db.create_all()

# Эндпоинты для категорий
@app.route('/categories', methods=['POST'])
def add_category():
    name = request.json.get('name')
    description = request.json.get('description')
    new_category = Category(name, description)
    db.session.add(new_category)
    db.session.commit()
    return category_schema.jsonify(new_category), 201

@app.route('/categories', methods=['GET'])
def get_categories():
    categories = Category.query.all()
    return categories_schema.jsonify(categories)

@app.route('/categories/<int:id>', methods=['GET'])
def get_category(id):
    category = Category.query.get(id)
    if category:
        return category_schema.jsonify(category)
    return jsonify({'message': 'Category not found'}), 404

@app.route('/categories/<int:id>', methods=['PUT'])
def update_category(id):
    category = Category.query.get(id)
    if category:
        category.name = request.json.get('name', category.name)
        category.description = request.json.get('description', category.description)
        db.session.commit()
        return category_schema.jsonify(category)
    return jsonify({'message': 'Category not found'}), 404

@app.route('/categories/<int:id>', methods=['DELETE'])
def delete_category(id):
    category = Category.query.get(id)
    if category:
        db.session.delete(category)
        db.session.commit()
        return '', 204
    return jsonify({'message': 'Category not found'}), 404

# Эндпоинты для товаров
@app.route('/products', methods=['POST'])
def add_product():
    name = request.json.get('name')
    description = request.json.get('description')
    price = request.json.get('price')
    in_stock = request.json.get('in_stock')
    category_id = request.json.get('category_id')

    new_product = Product(name, description, price, in_stock, category_id)
    db.session.add(new_product)
    db.session.commit()
    return product_schema.jsonify(new_product), 201

@app.route('/products', methods=['GET'])
def get_products():
    filters = []
    min_price = request.args.get('min_price', type=float)
    max_price = request.args.get('max_price', type=float)
    in_stock = request.args.get('in_stock', type=int)

    if min_price:
        filters.append(Product.price >= min_price)
    if max_price:
        filters.append(Product.price <= max_price)
    if in_stock is not None:
        filters.append(Product.in_stock > 0 if in_stock else Product.in_stock == 0)

    products = Product.query.filter(*filters).all()
    return products_schema.jsonify(products)

@app.route('/products/<int:id>', methods=['GET'])
def get_product(id):
    product = Product.query.get(id)
    if product:
        return product_schema.jsonify(product)
    return jsonify({'message': 'Product not found'}), 404

@app.route('/products/<int:id>', methods=['PUT'])
def update_product(id):
    product = Product.query.get(id)
    if product:
        product.name = request.json.get('name', product.name)
        product.description = request.json.get('description', product.description)
        product.price = request.json.get('price', product.price)
        product.in_stock = request.json.get('in_stock', product.in_stock)
        product.category_id = request.json.get('category_id', product.category_id)
        db.session.commit()
        return product_schema.jsonify(product)
    return jsonify({'message': 'Product not found'}), 404

@app.route('/products/<int:id>', methods=['DELETE'])
def delete_product(id):
    product = Product.query.get(id)
    if product:
        db.session.delete(product)
        db.session.commit()
        return '', 204
    return jsonify({'message': 'Product not found'}), 404

# Схемы Marshmallow
category_schema = CategorySchema()
categories_schema = CategorySchema(many=True)
product_schema = ProductSchema()
products_schema = ProductSchema(many=True)

if __name__ == '__main__':
    app.run(debug=True)
python app.py
POST /categories
{
  "name": "Electronics",
  "description": "Electronic devices"
}
POST /products
{
  "name": "Smartphone",
  "description": "Latest model",
  "price": 599.99,
  "in_stock": 100,
  "category_id": 1
}
GET /products?min_price=100&max_price=500
