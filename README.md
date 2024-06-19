from flask import Flask, request, jsonify
from flask_sqlalchemy import SQLAlchemy
import requests

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///transactions.db'
db = SQLAlchemy(app)

class Transaction(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    title = db.Column(db.String(200))
    description = db.Column(db.String(500))
    price = db.Column(db.Float)
    dateOfSale = db.Column(db.String(50))
    category = db.Column(db.String(100))
    sold = db.Column(db.Boolean)

# Initialize the database and create tables
@app.before_first_request
def create_tables():
    db.create_all()

@app.route('/initialize_db', methods=['GET'])
def initialize_db():
    url = 'https://s3.amazonaws.com/roxiler.com/product_transaction.json'
    response = requests.get(url)
    data = response.json()
    
    for item in data:
        transaction = Transaction(
            title=item['title'],
            description=item['description'],
            price=item['price'],
            dateOfSale=item['dateOfSale'],
            category=item['category'],
            sold=item['sold']
        )
        db.session.add(transaction)
    db.session.commit()
    
    return jsonify({"message": "Database initialized with seed data"}), 201

# API to list all transactions with search and pagination
@app.route('/transactions', methods=['GET'])
def get_transactions():
    month = request.args.get('month')
    search = request.args.get('search', '')
    page = int(request.args.get('page', 1))
    per_page = int(request.args.get('per_page', 10))

    query = Transaction.query
    if month:
        query = query.filter(db.func.strftime('%B', Transaction.dateOfSale) == month)
    if search:
        search = f"%{search}%"
        query = query.filter(
            db.or_(
                Transaction.title.like(search),
                Transaction.description.like(search),
                Transaction.price.like(search)
            )
        )
    transactions = query.paginate(page, per_page, False).items
    result = [t.__dict__ for t in transactions]
    for r in result:
        r.pop('_sa_instance_state', None)
    return jsonify(result)

# API for statistics
@app.route('/statistics', methods=['GET'])
def get_statistics():
    month = request.args.get('month')
    query = Transaction.query.filter(db.func.strftime('%B', Transaction.dateOfSale) == month)

    total_sales = query.with_entities(db.func.sum(Transaction.price)).scalar()
    total_sold = query.filter(Transaction.sold == True).count()
    total_not_sold = query.filter(Transaction.sold == False).count()

    return jsonify({
        "total_sales": total_sales,
        "total_sold_items": total_sold,
        "total_not_sold_items": total_not_sold
    })

# API for bar chart
@app.route('/bar_chart', methods=['GET'])
def get_bar_chart():
    month = request.args.get('month')
    query = Transaction.query.filter(db.func.strftime('%B', Transaction.dateOfSale) == month)
    
    price_ranges = [
        (0, 100), (101, 200), (201, 300), (301, 400), 
        (401, 500), (501, 600), (601, 700), (701, 800),
        (801, 900), (901, float('inf'))
    ]
    
    result = []
    for price_range in price_ranges:
        count = query.filter(
            Transaction.price >= price_range[0],
            Transaction.price <= price_range[1]
        ).count()
        result.append({
            "price_range": f"{price_range[0]}-{price_range[1]}",
            "count": count
        })

    return jsonify(result)

# API for pie chart
@app.route('/pie_chart', methods=['GET'])
def get_pie_chart():
    month = request.args.get('month')
    query = Transaction.query.filter(db.func.strftime('%B', Transaction.dateOfSale) == month)
    
    categories = query.with_entities(Transaction.category, db.func.count(Transaction.id)).group_by(Transaction.category).all()
    
    result = [{"category": category, "count": count} for category, count in categories]

    return jsonify(result)

# API to fetch combined data
@app.route('/combined_data', methods=['GET'])
def get_combined_data():
    month = request.args.get('month')
    
    transactions_res = get_transactions().get_json()
    statistics_res = get_statistics().get_json()
    bar_chart_res = get_bar_chart().get_json()
    pie_chart_res = get_pie_chart().get_json()
    
    return jsonify({
        "transactions": transactions_res,
        "statistics": statistics_res,
        "bar_chart": bar_chart_res,
        "pie_chart": pie_chart_res
    })

if __name__ == '__main__':
    app.run(debug=True)
