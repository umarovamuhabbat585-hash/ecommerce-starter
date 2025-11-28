
const express = require('express');
const sqlite3 = require('sqlite3').verbose();
const cors = require('cors');
const bodyParser = require('body-parser');
const { v4: uuidv4 } = require('uuid');
const path = require('path');

const app = express();
app.use(cors());
app.use(bodyParser.json());

const DB_PATH = path.join(__dirname, 'db.sqlite');
const db = new sqlite3.Database(DB_PATH);

// Initialize tables
db.serialize(() => {
  db.run(`CREATE TABLE IF NOT EXISTS products (
    id TEXT PRIMARY KEY,
    title TEXT,
    description TEXT,
    price REAL,
    image TEXT,
    stock INTEGER
  )`);

  db.run(`CREATE TABLE IF NOT EXISTS orders (
    id TEXT PRIMARY KEY,
    created_at TEXT,
    items TEXT,
    total REAL,
    customer_name TEXT,
    customer_email TEXT
  )`);

  // Seed sample products if empty
  db.get("SELECT COUNT(*) as cnt FROM products", (err, row) => {
    if (!err && row.cnt === 0) {
      const stmt = db.prepare("INSERT INTO products (id, title, description, price, image, stock) VALUES (?, ?, ?, ?, ?, ?)");
      const sample = [
        [uuidv4(), "Кружка «Узбекистан»", "Керамическая кружка 350ml", 9.99, "https://via.placeholder.com/300x200?text=Mug", 50],
        [uuidv4(), "Футболка с принтом", "Мягкая хлопковая футболка", 19.99, "https://via.placeholder.com/300x200?text=T-shirt", 30],
        [uuidv4(), "Блокнот", "А5 блокнот в мягкой обложке", 4.5, "https://via.placeholder.com/300x200?text=Notebook", 100]
      ];
      sample.forEach(p => stmt.run(p));
      stmt.finalize();
      console.log("Seeded product data.");
    }
  });
});

// Routes
app.get('/api/products', (req, res) => {
  db.all("SELECT * FROM products", (err, rows) => {
    if (err) return res.status(500).json({ error: err.message });
    res.json(rows);
  });
});

app.get('/api/products/:id', (req, res) => {
  db.get("SELECT * FROM products WHERE id = ?", [req.params.id], (err, row) => {
    if (err) return res.status(500).json({ error: err.message });
    if (!row) return res.status(404).json({ error: 'Product not found' });
    res.json(row);
  });
});

// Place an order
app.post('/api/orders', (req, res) => {
  const { customer_name, customer_email, items, total } = req.body;
  if (!items || !Array.isArray(items) || items.length === 0) {
    return res.status(400).json({ error: 'Basket is empty' });
  }

  const id = uuidv4();
  const created_at = new Date().toISOString();
  const itemsJson = JSON.stringify(items);

  db.run(
    "INSERT INTO orders (id, created_at, items, total, customer_name, customer_email) VALUES (?, ?, ?, ?, ?, ?)",
    [id, created_at, itemsJson, total, customer_name || '', customer_email || ''],
    function(err) {
      if (err) return res.status(500).json({ error: err.message });
      // Decrease stock for each item (best-effort, no transaction for simplicity)
      items.forEach(it => {
        db.run("UPDATE products SET stock = stock - ? WHERE id = ? AND stock >= ?",
          [it.quantity, it.id, it.quantity]);
      });
      res.json({ id, created_at });
    }
  );
});

// Admin: create/update/delete product (very simple, no auth)
app.post('/api/products', (req, res) => {
  const { title, description, price, image, stock } = req.body;
  const id = uuidv4();
  db.run("INSERT INTO products (id, title, description, price, image, stock) VALUES (?, ?, ?, ?, ?, ?)",
    [id, title, description, price, image, stock || 0],
    function(err) {
      if (err) return res.status(500).json({ error: err.message });
      res.json({ id });
    });
});

app.put('/api/products/:id', (req, res) => {
  const { title, description, price, image, stock } = req.body;
  db.run("UPDATE products SET title=?, description=?, price=?, image=?, stock=? WHERE id=?",
    [title, description, price, image, stock, req.params.id],
    function(err) {
      if (err) return res.status(500).json({ error: err.message });
      res.json({ updated: this.changes });
    });
});

app.delete('/api/products/:id', (req, res) => {
  db.run("DELETE FROM products WHERE id=?", [req.params.id], function(err) {
    if (err) return res.status(500).json({ error: err.message });
    res.json({ deleted: this.changes });
  });
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`Backend listening on http://localhost:${PORT}`);
});
