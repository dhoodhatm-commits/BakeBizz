// backend/server.js
import express from "express";
import sqlite3 from "sqlite3";
import { open } from "sqlite";
import cors from "cors";

const app = express();
app.use(cors());
app.use(express.json());

// --- Database Setup ---
let db;
async function initDB() {
  db = await open({
    filename: "./db.sqlite",
    driver: sqlite3.Database,
  });

  // Create tables if they don’t exist
  await db.exec(`
    CREATE TABLE IF NOT EXISTS customers (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      name TEXT,
      type TEXT CHECK(type IN ('direct', 'consignment'))
    );
  `);

  await db.exec(`
    CREATE TABLE IF NOT EXISTS products (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      name TEXT,
      price REAL,
      stock INTEGER DEFAULT 0
    );
  `);

  await db.exec(`
    CREATE TABLE IF NOT EXISTS invoices (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      customerId INTEGER,
      date TEXT,
      FOREIGN KEY(customerId) REFERENCES customers(id)
    );
  `);

  await db.exec(`
    CREATE TABLE IF NOT EXISTS invoice_items (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      invoiceId INTEGER,
      productId INTEGER,
      quantity INTEGER,
      FOREIGN KEY(invoiceId) REFERENCES invoices(id),
      FOREIGN KEY(productId) REFERENCES products(id)
    );
  `);
}

// --- API Routes ---

// Customers
app.get("/customers", async (req, res) => {
  const customers = await db.all("SELECT * FROM customers");
  res.json(customers);
});

app.post("/customers", async (req, res) => {
  const { name, type } = req.body;
  const result = await db.run("INSERT INTO customers (name, type) VALUES (?, ?)", [name, type]);
  res.json({ id: result.lastID, name, type });
});

// Products
app.get("/products", async (req, res) => {
  const products = await db.all("SELECT * FROM products");
  res.json(products);
});

app.post("/products", async (req, res) => {
  const { name, price, stock } = req.body;
  const result = await db.run("INSERT INTO products (name, price, stock) VALUES (?, ?, ?)", [
    name,
    price,
    stock || 0,
  ]);
  res.json({ id: result.lastID, name, price, stock });
});

// Update stock manually
app.post("/products/:id/stock", async (req, res) => {
  const { adjustment } = req.body; // +10 for new batch, -5 for spoilage, etc.
  await db.run("UPDATE products SET stock = stock + ? WHERE id = ?", [adjustment, req.params.id]);
  const updated = await db.get("SELECT * FROM products WHERE id = ?", [req.params.id]);
  res.json(updated);
});

// Invoices
app.get("/invoices", async (req, res) => {
  const invoices = await db.all("SELECT * FROM invoices");
  res.json(invoices);
});

app.post("/invoices", async (req, res) => {
  const { customerId, items } = req.body;
  const date = new Date().toISOString();

  const result = await db.run("INSERT INTO invoices (customerId, date) VALUES (?, ?)", [
    customerId,
    date,
  ]);
  const invoiceId = result.lastID;

  for (let item of items) {
    await db.run("INSERT INTO invoice_items (invoiceId, productId, quantity) VALUES (?, ?, ?)", [
      invoiceId,
      item.productId,
      item.quantity,
    ]);

    // Reduce stock
    await db.run("UPDATE products SET stock = stock - ? WHERE id = ?", [
      item.quantity,
      item.productId,
    ]);
  }

  res.json({ invoiceId, customerId, date, items });
});

// Reports
app.get("/reports/sales-by-customer", async (req, res) => {
  const rows = await db.all(`
    SELECT c.name as customer, SUM(ii.quantity * p.price) as total
    FROM invoices i
    JOIN customers c ON i.customerId = c.id
    JOIN invoice_items ii ON ii.invoiceId = i.id
    JOIN products p ON ii.productId = p.id
    GROUP BY c.name
  `);
  res.json(rows);
});

app.get("/reports/sales-by-product", async (req, res) => {
  const rows = await db.all(`
    SELECT p.name as product, SUM(ii.quantity) as qtySold, SUM(ii.quantity * p.price) as total
    FROM invoice_items ii
    JOIN products p ON ii.productId = p.id
    GROUP BY p.name
  `);
  res.json(rows);
});

// --- Start server ---
const PORT = process.env.PORT || 3001;

initDB().then(() => {
  app.listen(PORT, () => {
    console.log(`✅ Backend running on http://localhost:${PORT}`);
  });
});
