# Roxiler_system_Assignment
from flask import Flask, jsonify, request, render_template
import requests
import sqlite3
import os

app = Flask(__name__)

# Database setup
DATABASE = 'transactions.db'

def init_db():
    with sqlite3.connect(DATABASE) as conn:
        cursor = conn.cursor()
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS transactions (
                id INTEGER PRIMARY KEY,
                title TEXT,
                description TEXT,
                price REAL,
                dateOfSale TEXT,
                category TEXT
            )
        ''')
        conn.commit()

@app.route('/api/init-db', methods=['GET'])
def initialize_db():
    response = requests.get("https://s3.amazonaws.com/roxiler.com/product_transaction.json")
    data = response.json()
    
    with sqlite3.connect(DATABASE) as conn:
        cursor = conn.cursor()
        cursor.execute('DELETE FROM transactions')  # Clear existing data
        for item in data:
            cursor.execute('''
                INSERT INTO transactions (title, description, price, dateOfSale, category)
                VALUES (?, ?, ?, ?, ?)
            ''', (item['title'], item['description'], item['price'], item['dateOfSale'], item['category']))
        conn.commit()
    return jsonify({"message": "Database initialized successfully!"})

@app.route('/api/transactions', methods=['GET'])
def get_transactions():
    month = request.args.get('month')
    page = int(request.args.get('page', 1))
    per_page = int(request.args.get('per_page', 10))
    search = request.args.get('search', '')

    with sqlite3.connect(DATABASE) as conn:
        cursor = conn.cursor()
        query = '''
            SELECT * FROM transactions
            WHERE strftime('%m', dateOfSale) = ?
            AND (title LIKE ? OR description LIKE ? OR price LIKE ?)
            LIMIT ? OFFSET ?
        '''
        cursor.execute(query, (month, f'%{search}%', f'%{search}%', f'%{search}%', per_page, (page - 1) * per_page))
        transactions = cursor.fetchall()
        
    return jsonify(transactions)

@app.route('/api/statistics', methods=['GET'])
def get_statistics():
    month = request.args.get('month')

    with sqlite3.connect(DATABASE) as conn:
        cursor = conn.cursor()
        cursor.execute('''
            SELECT SUM(price) AS total_sales, COUNT(*) AS total_sold_items 
            FROM transactions WHERE strftime('%m', dateOfSale) = ?
        ''', (month,))
        
        total_sales, total_sold_items = cursor.fetchone() or (0, 0)
        
        cursor.execute('''
            SELECT COUNT(*) FROM transactions 
            WHERE strftime('%m', dateOfSale) = ? AND price = 0
        ''', (month,))
        
        total_not_sold_items = cursor.fetchone()[0]
        
    return jsonify({
        "total_sales": total_sales,
        "total_sold_items": total_sold_items,
        "total_not_sold_items": total_not_sold_items
    })

@app.route('/api/bar-chart', methods=['GET'])
def get_bar_chart():
    month = request.args.get('month')
    ranges = {
        '0-100': 0,
        '101-200': 0,
        '201-300': 0,
        '301-400': 0,
        '401-500': 0,
        '501-600': 0,
        '601-700': 0,
        '701-800': 0,
        '801-900': 0,
        '901-above': 0
    }

    with sqlite3.connect(DATABASE) as conn:
        cursor = conn.cursor()
        cursor.execute('''
            SELECT price FROM transactions WHERE strftime('%m', dateOfSale) = ?
        ''', (month,))
        
        prices = cursor.fetchall()
        
        for (price,) in prices:
            if price <= 100:
                ranges['0-100'] += 1
            elif price <= 200:
                ranges['101-200'] += 1
            elif price <= 300:
                ranges['201-300'] += 1
            elif price <= 400:
                ranges['301-400'] += 1
            elif price <= 500:
                ranges['401-500'] += 1
            elif price <= 600:
                ranges['501-600'] += 1
            elif price <= 700:
                ranges['601-700'] += 1
            elif price <= 800:
                ranges['701-800'] += 1
            elif price <= 900:
                ranges['801-900'] += 1
            else:
                ranges['901-above'] += 1
    
    return jsonify(ranges)

@app.route('/api/pie-chart', methods=['GET'])
def get_pie_chart():
    month = request.args.get('month')
    
    with sqlite3.connect(DATABASE) as conn:
        cursor = conn.cursor()
        cursor.execute('''
            SELECT category, COUNT(*) FROM transactions 
            WHERE strftime('%m', dateOfSale) = ? GROUP BY category
        ''', (month,))
        
        categories = cursor.fetchall()
    
    return jsonify(dict(categories))

@app.route('/api/combined-data', methods=['GET'])
def get_combined_data():
    month = request.args.get('month')
    
    transactions = get_transactions()
    statistics = get_statistics()
    bar_chart = get_bar_chart()
    pie_chart = get_pie_chart()
    
    combined_response = {
        "transactions": transactions.json,
        "statistics": statistics.json,
        "bar_chart": bar_chart.json,
        "pie_chart": pie_chart.json
    }
    
    return jsonify(combined_response)

@app.route('/')
def index():
    return render_template('index.html')

if __name__ == '_main_':
    init_db()
    app.run(debug=True)
