// TravelCompany Fullstack Template
// Contains backend (Node/Express + SQLite) and frontend (React single-file) samples.

/*
Project structure (suggested):

travel-company/
├─ backend/
│  ├─ package.json
│  ├─ server.js
│  ├─ db.js
│  └─ migrations.sql
└─ frontend/
   ├─ package.json
   ├─ public/index.html
   └─ src/App.jsx

Instructions: run backend on port 4000, frontend on 3000.

*/

// =========================
// backend/package.json
// =========================
{
  "name": "travel-backend",
  "version": "1.0.0",
  "main": "server.js",
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "sqlite3": "^5.1.6",
    "better-sqlite3": "^8.1.0",
    "cors": "^2.8.5",
    "body-parser": "^1.20.2",
    "uuid": "^9.0.0"
  }
}

// =========================
// backend/migrations.sql
// =========================
/*
CREATE TABLE tours (
  id TEXT PRIMARY KEY,
  title TEXT NOT NULL,
  description TEXT,
  price INTEGER NOT NULL,
  duration_days INTEGER,
  available INTEGER DEFAULT 1
);

CREATE TABLE bookings (
  id TEXT PRIMARY KEY,
  tour_id TEXT,
  customer_name TEXT,
  customer_email TEXT,
  seats INTEGER,
  created_at TEXT,
  FOREIGN KEY (tour_id) REFERENCES tours(id)
);
*/

// =========================
// backend/db.js
// =========================
const Database = require('better-sqlite3');
const path = require('path');
const db = new Database(path.join(__dirname, 'travel.db'));

// initialize simple schema if not exists
const init = () => {
  db.exec(`
    CREATE TABLE IF NOT EXISTS tours (
      id TEXT PRIMARY KEY,
      title TEXT NOT NULL,
      description TEXT,
      price INTEGER NOT NULL,
      duration_days INTEGER,
      available INTEGER DEFAULT 1
    );
    CREATE TABLE IF NOT EXISTS bookings (
      id TEXT PRIMARY KEY,
      tour_id TEXT,
      customer_name TEXT,
      customer_email TEXT,
      seats INTEGER,
      created_at TEXT,
      FOREIGN KEY (tour_id) REFERENCES tours(id)
    );
  `);

  // seed some tours if empty
  const row = db.prepare('SELECT COUNT(*) as c FROM tours').get();
  if (row.c === 0) {
    const insert = db.prepare('INSERT INTO tours (id,title,description,price,duration_days,available) VALUES (?,?,?,?,?,?)');
    insert.run('tour-1','Pyramids Day Trip','Visit the Great Pyramids of Giza with guide', 30, 1, 1);
    insert.run('tour-2','Nile Sunset Cruise','Relaxing sunset cruise on the Nile', 45, 0.5, 1);
    insert.run('tour-3','Alexandria City Tour','Historical tour of Alexandria', 55, 1, 1);
  }
};

module.exports = { db, init };

// =========================
// backend/server.js
// =========================
const express = require('express');
const bodyParser = require('body-parser');
const cors = require('cors');
const { v4: uuidv4 } = require('uuid');
const { db, init } = require('./db');

init();

const app = express();
app.use(cors());
app.use(bodyParser.json());

// GET /api/tours - list tours
app.get('/api/tours', (req, res) => {
  const rows = db.prepare('SELECT * FROM tours WHERE available=1').all();
  res.json(rows);
});

// GET /api/tours/:id - tour detail
app.get('/api/tours/:id', (req, res) => {
  const row = db.prepare('SELECT * FROM tours WHERE id=?').get(req.params.id);
  if (!row) return res.status(404).json({ error: 'Not found' });
  res.json(row);
});

// POST /api/bookings - create booking
app.post('/api/bookings', (req, res) => {
  const { tour_id, customer_name, customer_email, seats } = req.body;
  if (!tour_id || !customer_name || !customer_email || !seats) {
    return res.status(400).json({ error: 'Missing fields' });
  }
  const tour = db.prepare('SELECT * FROM tours WHERE id=?').get(tour_id);
  if (!tour) return res.status(400).json({ error: 'Invalid tour_id' });

  const id = uuidv4();
  const now = new Date().toISOString();
  db.prepare('INSERT INTO bookings (id,tour_id,customer_name,customer_email,seats,created_at) VALUES (?,?,?,?,?,?)')
    .run(id,tour_id,customer_name,customer_email,seats,now);
  res.status(201).json({ id, tour, created_at: now });
});

// GET /api/bookings - list bookings (basic admin)
app.get('/api/bookings', (req, res) => {
  const rows = db.prepare('SELECT * FROM bookings ORDER BY created_at DESC LIMIT 100').all();
  res.json(rows);
});

const port = process.env.PORT || 4000;
app.listen(port, () => console.log('Travel backend listening on', port));

// =========================
// frontend/package.json (for Vite React)
// =========================
{
  "name": "travel-frontend",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "react-router-dom": "^6.12.1"
  },
  "devDependencies": {
    "vite": "^5.0.0"
  }
}

// =========================
// frontend/public/index.html
// =========================
<!doctype html>
<html>
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Travel Company</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.jsx"></script>
  </body>
</html>

// =========================
// frontend/src/main.jsx
// =========================
import React from 'react'
import { createRoot } from 'react-dom/client'
import App from './App'

createRoot(document.getElementById('root')).render(<App />)

// =========================
// frontend/src/App.jsx
// =========================
import React, { useEffect, useState } from 'react'

const API = import.meta.env.VITE_API_URL || 'http://localhost:4000/api'

export default function App(){
  const [tours, setTours] = useState([])
  const [selected, setSelected] = useState(null)
  const [form, setForm] = useState({name:'',email:'',seats:1})
  const [message, setMessage] = useState('')

  useEffect(()=>{ fetchTours() },[])

  async function fetchTours(){
    const res = await fetch(`${API}/tours`)
    const data = await res.json()
    setTours(data)
  }

  async function viewTour(id){
    const res = await fetch(`${API}/tours/${id}`)
    const data = await res.json()
    setSelected(data)
  }

  async function submitBooking(e){
    e.preventDefault()
    if(!selected) return setMessage('Select a tour first')
    const payload = { tour_id:selected.id, customer_name:form.name, customer_email:form.email, seats: parseInt(form.seats) }
    const res = await fetch(`${API}/bookings`, { method:'POST', headers:{'Content-Type':'application/json'}, body: JSON.stringify(payload) })
    if(res.ok){
      const data = await res.json()
      setMessage('Booking confirmed! ID: '+data.id)
      setForm({name:'',email:'',seats:1})
    } else {
      const err = await res.json()
      setMessage('Error: '+(err.error||'unknown'))
    }
  }

  return (
    <div style={{fontFamily:'Arial', padding:20}}>
      <h1>Travel Company</h1>
      <div style={{display:'flex',gap:20}}>
        <div style={{flex:1}}>
          <h2>Tours</h2>
          {tours.map(t=> (
            <div key={t.id} style={{border:'1px solid #ddd', padding:10, marginBottom:10}}>
              <h3>{t.title} - ${t.price}</h3>
              <p>{t.description}</p>
              <button onClick={()=>viewTour(t.id)}>View</button>
            </div>
          ))}
        </div>

        <div style={{width:360}}>
          <h2>Details</h2>
          {selected ? (
            <div>
              <h3>{selected.title}</h3>
              <p>{selected.description}</p>
              <p>Price: ${selected.price} | Duration: {selected.duration_days} days</p>

              <h4>Book this tour</h4>
              <form onSubmit={submitBooking}>
                <div>
                  <label>Name</label><br/>
                  <input value={form.name} onChange={e=>setForm({...form,name:e.target.value})} required />
                </div>
                <div>
                  <label>Email</label><br/>
                  <input value={form.email} onChange={e=>setForm({...form,email:e.target.value})} required />
                </div>
                <div>
                  <label>Seats</label><br/>
                  <input type="number" min={1} value={form.seats} onChange={e=>setForm({...form,seats:e.target.value})} />
                </div>
                <button type="submit">Book</button>
              </form>
            </div>
          ) : <p>Select a tour to see details</p>}

          <div style={{marginTop:20}}>
            <strong>{message}</strong>
          </div>
        </div>
      </div>
    </div>
  )
}

/*
Notes & next steps:
- To run backend: cd backend && npm install && npm start
- To run frontend (Vite): cd frontend && npm install && npm run dev
- Set VITE_API_URL in .env if backend not localhost.
- Add validations, authentication, admin pages, payment integration for production.
- Use HTTPS, rate-limiting, input sanitization, and CORS best practices.
*/
