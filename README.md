{
  "name": "hylton-hangout",
  "version": "1.5.0",
  "description": "The Hylton Hangout app with login, scratch ticket, points, menu, and cashier-confirmed purchases",
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "bcryptjs": "^2.4.3",
    "body-parser": "^1.20.2",
    "connect-pg-simple": "^8.0.0",
    "dotenv": "^16.1.4",
    "express": "^4.18.2",
    "express-session": "^1.17.3",
    "pg": "^8.11.0",
    "uuid": "^9.0.0"
  }
}
require('dotenv').config();

const express = require('express');
const bodyParser = require('body-parser');
const session = require('express-session');
const pg = require('pg');
const PgSession = require('connect-pg-simple')(session);
const bcrypt = require('bcryptjs');
const path = require('path');
const { v4: uuidv4 } = require('uuid');

const PORT = process.env.PORT || 3000;
const DATABASE_URL = process.env.DATABASE_URL || 'postgres://postgres:postgres@localhost:5432/hylton';
const SESSION_SECRET = process.env.SESSION_SECRET || 'replace-with-secure-secret';
const CASHIER_PASSCODE = process.env.CASHIER_PASSCODE || '198759';

const POINTS_PER_PURCHASE = parseInt(process.env.POINTS_PER_PURCHASE || '100', 10);
const DAILY_POINTS_CAP = parseInt(process.env.DAILY_POINTS_CAP || '500', 10);
const MIN_SECONDS_BETWEEN_CONFIRMS = parseInt(process.env.MIN_SECONDS_BETWEEN_CONFIRMS || '10', 10);

const pool = new pg.Pool({
  connectionString: DATABASE_URL,
  ssl: process.env.DB_SSL === 'true' ? { rejectUnauthorized: false } : false
});

const app = express();
app.use(bodyParser.json());
app.use(bodyParser.urlencoded({ extended: true }));

app.use(session({
  store: new PgSession({
    pool,
    tableName: 'session',
    createTableIfMissing: true
  }),
  secret: SESSION_SECRET,
  resave: false,
  saveUninitialized: false,
  cookie: {
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'lax',
    maxAge: 7 * 24 * 60 * 60 * 1000
  }
}));

app.use(express.static(path.join(__dirname, 'public')));

async function initDb() {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');

    await client.query(`
      CREATE TABLE IF NOT EXISTS users (
        id SERIAL PRIMARY KEY,
        username TEXT UNIQUE NOT NULL,
        password_hash TEXT NOT NULL,
        points INTEGER NOT NULL DEFAULT 0,
        last_scratch_date TEXT,
        created_at TIMESTAMPTZ DEFAULT (now() AT TIME ZONE 'UTC')
      );
    `);

    await client.query(`
      CREATE TABLE IF NOT EXISTS scratch_logs (
        id SERIAL PRIMARY KEY,
        user_id INTEGER REFERENCES users(id),
        date TEXT,
        win BOOLEAN DEFAULT FALSE
      );
    `);

    await client.query(`
      CREATE TABLE IF NOT EXISTS menu (
        id SERIAL PRIMARY KEY,
        name TEXT,
        description TEXT,
        price_cents INTEGER
      );
    `);

    await client.query(`
      CREATE TABLE IF NOT EXISTS coupons (
        id SERIAL PRIMARY KEY,
        user_id INTEGER REFERENCES users(id),
        code TEXT UNIQUE,
        type TEXT,
        value INTEGER,
        created_at TIMESTAMPTZ DEFAULT (now() AT TIME ZONE 'UTC'),
        expires_at TIMESTAMPTZ,
        redeemed BOOLEAN DEFAULT FALSE,
        redeemed_at TIMESTAMPTZ
      );
    `);

    await client.query(`
      CREATE TABLE IF NOT EXISTS purchases (
        id SERIAL PRIMARY KEY,
        user_id INTEGER NOT NULL REFERENCES users(id),
        menu_id INTEGER,
        amount_cents INTEGER,
        status TEXT NOT NULL DEFAULT 'pending',
        confirm_code TEXT UNIQUE,
        created_at TIMESTAMPTZ DEFAULT (now() AT TIME ZONE 'UTC'),
        confirmed_at TIMESTAMPTZ,
        cashier_name TEXT,
        awarded_points INTEGER DEFAULT 0
      );
    `);

    const menuCheck = await client.query('SELECT COUNT(*)::int AS count FROM menu');
    if (menuCheck.rows[0].count === 0) {
      const items = [
        ['Espresso', 'Single shot espresso', 300],
        ['Americano', 'Espresso with hot water', 350],
        ['Latte', 'Espresso with steamed milk', 450],
        ['Cappuccino', 'Espresso with milk foam', 450],
        ['Mocha', 'Chocolate flavored latte', 500],
        ['Cold Brew', 'Slow-steeped cold coffee', 450],
        ['Iced Latte', 'Espresso over milk and ice', 450],
        ['Tea', 'Black or green tea', 250]
      ];
      for (const row of items) {
        await client.query('INSERT INTO menu (name, description, price_cents) VALUES ($1, $2, $3)', row);
      }
    }

    await client.query('COMMIT');
    console.log('Database initialized');
  } catch (err) {
    await client.query('ROLLBACK');
    console.error('Database init failed', err);
    throw err;
  } finally {
    client.release();
  }
}

function requireAuth(req, res, next) {
  if (!req.session || !req.session.userId) {
    return res.status(401).json({ error: 'Unauthorized' });
  }
  next();
}

function todayUTCDateString() {
  const d = new Date();
  const year = d.getUTCFullYear();
  const month = String(d.getUTCMonth() + 1).padStart(2, '0');
  const day = String(d.getUTCDate()).padStart(2, '0');
  return `${year}-${month}-${day}`;
}

function makeConfirmCode() {
  return Math.floor(100000 + Math.random() * 900000).toString();
}

async function createCouponForUser(clientOrPool, userId, type, value = null, daysValid = 30) {
  const code = (type === 'free_drink' ? 'FREE-' : 'OFF-') + uuidv4().slice(0, 8).toUpperCase();
  const expiresAt = new Date(Date.now() + daysValid * 24 * 60 * 60 * 1000).toISOString();
  const r = await clientOrPool.query(
    `INSERT INTO coupons (user_id, code, type, value, expires_at)
     VALUES ($1, $2, $3, $4, $5)
     RETURNING id, code, type, value, expires_at`,
    [userId, code, type, value, expiresAt]
  );
  return r.rows[0];
}

app.post('/api/register', async (req, res) => {
  const { username, password } = req.body || {};
  if (!username || !password) {
    return res.status(400).json({ error: 'username and password required' });
  }

  try {
    const hash = bcrypt.hashSync(password, 10);
    const result = await pool.query(
      'INSERT INTO users (username, password_hash, points) VALUES ($1, $2, $3) RETURNING id',
      [username, hash, 100]
    );

    req.session.userId = result.rows[0].id;
    return res.json({ success: true, message: 'Account created. You received 100 points.' });
  } catch (err) {
    if (err && err.code === '23505') {
      return res.status(409).json({ error: 'username already exists' });
    }
    console.error('register error', err);
    return res.status(500).json({ error: 'Could not create account' });
  }
});

app.post('/api/login', async (req, res) => {
  const { username, password } = req.body || {};
  if (!username || !password) {
    return res.status(400).json({ error: 'username and password required' });
  }

  try {
    const r = await pool.query('SELECT id, password_hash FROM users WHERE username = $1', [username]);
    if (r.rows.length === 0) return res.status(401).json({ error: 'Invalid credentials' });

    const user = r.rows[0];
    const ok = bcrypt.compareSync(password, user.password_hash);
    if (!ok) return res.status(401).json({ error: 'Invalid credentials' });

    req.session.userId = user.id;
    return res.json({ success: true });
  } catch (err) {
    console.error('login error', err);
    return res.status(500).json({ error: 'Login failed' });
  }
});

app.post('/api/logout', (req, res) => {
  req.session.destroy(() => res.json({ success: true }));
});

app.get('/api/user', requireAuth, async (req, res) => {
  try {
    const user = await pool.query(
      'SELECT id, username, points, last_scratch_date, created_at FROM users WHERE id = $1',
      [req.session.userId]
    );

    if (user.rows.length === 0) return res.status(404).json({ error: 'User not found' });

    const userData = user.rows[0];
    const canScratch = userData.last_scratch_date !== todayUTCDateString();

    const coupons = await pool.query(`
      SELECT id, code, type, value, created_at, expires_at, redeemed
      FROM coupons
      WHERE user_id = $1
      ORDER BY created_at DESC
    `, [req.session.userId]);

    return res.json({
      id: userData.id,
      username: userData.username,
      points: userData.points,
      last_scratch_date: userData.last_scratch_date,
      created_at: userData.created_at,
      canScratch,
      coupons: coupons.rows
    });
  } catch (err) {
    console.error('get user error', err);
    return res.status(500).json({ error: 'Could not get user info' });
  }
});

app.post('/api/scratch', requireAuth, async (req, res) => {
  try {
    const user = await pool.query('SELECT id, last_scratch_date FROM users WHERE id = $1', [req.session.userId]);
    if (user.rows.length === 0) return res.status(404).json({ error: 'User not found' });

    const today = todayUTCDateString();
    const current = user.rows[0];

    if (current.last_scratch_date === today) {
      return res.status(403).json({ error: 'Already scratched today' });
    }

    const roll = Math.floor(Math.random() * 100) + 1;
    const win = roll === 1;

    await pool.query('UPDATE users SET last_scratch_date = $1 WHERE id = $2', [today, req.session.userId]);

    await pool.query(
      'INSERT INTO scratch_logs (user_id, date, win) VALUES ($1, $2, $3)',
      [req.session.userId, today, win]
    );

    if (win) {
      const coupon = await createCouponForUser(pool, req.session.userId, 'free_drink', null, 30);
      return res.json({
        win: true,
        message: 'You win a free drink.',
        coupon
      });
    }

    return res.json({
      win: false,
      message: 'Not today sorry. Try again tomorrow'
    });
  } catch (err) {
    console.error('scratch error', err);
    return res.status(500).json({ error: 'Could not process scratch' });
  }
});

app.post('/api/add-points', requireAuth, async (req, res) => {
  const amount = parseInt(req.body.amount, 10) || 0;
  if (amount <= 0) {
    return res.status(400).json({ error: 'amount must be positive integer' });
  }

  try {
    const user = await pool.query('SELECT id, points FROM users WHERE id = $1', [req.session.userId]);
    if (user.rows.length === 0) return res.status(404).json({ error: 'User not found' });

    let newPoints = user.rows[0].points + amount;
    let couponCreated = null;

    if (newPoints >= 1000) {
      newPoints = 0;
      couponCreated = await createCouponForUser(pool, req.session.userId, 'percent_off', 10, 60);
    }

    await pool.query('UPDATE users SET points = $1 WHERE id = $2', [newPoints, req.session.userId]);

    return res.json({
      points: newPoints,
      coupon: couponCreated
    });
  } catch (err) {
    console.error('add points error', err);
    return res.status(500).json({ error: 'Could not add points' });
  }
});

app.get('/api/menu', async (req, res) => {
  try {
    const result = await pool.query('SELECT id, name, description, price_cents FROM menu ORDER BY id');
    return res.json(result.rows);
  } catch (err) {
    console.error('menu error', err);
    return res.status(500).json({ error: 'Could not load menu' });
  }
});

app.post('/api/coupons/redeem', requireAuth, async (req, res) => {
  const { code } = req.body || {};
  if (!code) return res.status(400).json({ error: 'coupon code required' });

  try {
    const r = await pool.query('SELECT * FROM coupons WHERE code = $1 AND user_id = $2', [code, req.session.userId]);
    if (r.rows.length === 0) return res.status(404).json({ error: 'Coupon not found' });

    const coupon = r.rows[0];
    if (coupon.redeemed) return res.status(400).json({ error: 'Coupon already redeemed' });
    if (coupon.expires_at && new Date(coupon.expires_at) < new Date()) return res.status(400).json({ error: 'Coupon expired' });

    await pool.query(
      'UPDATE coupons SET redeemed = TRUE, redeemed_at = NOW() WHERE id = $1',
      [coupon.id]
    );

    return res.json({
      success: true,
      message: 'Coupon redeemed',
      coupon: {
        id: coupon.id,
        code: coupon.code,
        type: coupon.type,
        value: coupon.value
      }
    });
  } catch (err) {
    console.error('redeem coupon error', err);
    return res.status(500).json({ error: 'Could not redeem coupon' });
  }
});

app.post('/api/coupons/discard', async (req, res) => {
  const { code } = req.body || {};
  if (!code) return res.status(400).json({ error: 'coupon code required' });

  try {
    const r = await pool.query('SELECT * FROM coupons WHERE code = $1', [code]);
    if (r.rows.length === 0) return res.status(404).json({ error: 'Coupon not found' });

    const coupon = r.rows[0];
    if (coupon.redeemed) return res.status(400).json({ error: 'Coupon already redeemed' });
    if (coupon.expires_at && new Date(coupon.expires_at) < new Date()) return res.status(400).json({ error: 'Coupon expired' });

    await pool.query(
      'UPDATE coupons SET redeemed = TRUE, redeemed_at = NOW() WHERE id = $1',
      [coupon.id]
    );

    return res.json({
      success: true,
      message: 'Coupon discarded (redeemed) by cashier',
      coupon: {
        code: coupon.code,
        type: coupon.type,
        user_id: coupon.user_id
      }
    });
  } catch (err) {
    console.error('discard coupon error', err);
    return res.status(500).json({ error: 'Could not discard coupon' });
  }
});

app.post('/api/purchase', requireAuth, async (req, res) => {
  const menuId = parseInt(req.body.menuId, 10);
  if (!menuId) return res.status(400).json({ error: 'menuId required' });

  try {
    const menu = await pool.query('SELECT id, name, price_cents FROM menu WHERE id = $1', [menuId]);
    if (menu.rows.length === 0) return res.status(404).json({ error: 'Menu item not found' });

    let confirmCode;
    let created = false;

    for (let i = 0; i < 10; i++) {
      confirmCode = makeConfirmCode();
      try {
        const result = await pool.query(
          `INSERT INTO purchases (user_id, menu_id, amount_cents, status, confirm_code)
           VALUES ($1, $2, $3, 'pending', $4)
           RETURNING id`,
          [req.session.userId, menuId, menu.rows[0].price_cents, confirmCode]
        );

        created = true;
        return res.json({
          success: true,
          pending: true,
          message: 'Purchase recorded. Show the confirm code to the cashier.',
          purchaseId: result.rows[0].id,
          menuName: menu.rows[0].name,
          confirmCode
        });
      } catch (innerErr) {
        if (innerErr && innerErr.code === '23505') {
          continue;
        }
        throw innerErr;
      }
    }

    if (!created) return res.status(500).json({ error: 'Could not generate a unique confirm code' });
  } catch (err) {
    console.error('purchase create error', err);
    return res.status(500).json({ error: 'Could not create purchase' });
  }
});

app.post('/api/purchase/confirm', async (req, res) => {
  const { confirmCode, cashierPass, cashierName } = req.body || {};
  if (!confirmCode) return res.status(400).json({ error: 'confirmCode required' });

  if (cashierPass && cashierPass !== CASHIER_PASSCODE) {
    return res.status(403).json({ error: 'Invalid cashier passcode' });
  }

  try {
    const purchaseR = await pool.query('SELECT * FROM purchases WHERE confirm_code = $1', [confirmCode]);
    if (purchaseR.rows.length === 0) return res.status(404).json({ error: 'Purchase not found' });

    const purchase = purchaseR.rows[0];
    if (purchase.status !== 'pending') {
      return res.status(400).json({ error: 'Purchase already confirmed or invalid' });
    }

    const todayAwarded = await pool.query(`
      SELECT COALESCE(SUM(awarded_points), 0)::int AS total
      FROM purchases
      WHERE user_id = $1 AND status = 'confirmed' AND confirmed_at >= date_trunc('day', now())
    `, [purchase.user_id]);

    const alreadyAwardedToday = Number(todayAwarded.rows[0].total || 0);

    if (alreadyAwardedToday >= DAILY_POINTS_CAP) {
      await pool.query(
        `UPDATE purchases
         SET status = 'confirmed',
             confirmed_at = NOW(),
             cashier_name = $1,
             awarded_points = 0
         WHERE id = $2`,
        [cashierName || null, purchase.id]
      );

      return res.json({
        success: true,
        message: 'Purchase confirmed, but the user reached the daily points cap. No points awarded.'
      });
    }

    const lastConfirmed = await pool.query(`
      SELECT confirmed_at
      FROM purchases
      WHERE user_id = $1 AND status = 'confirmed'
      ORDER BY confirmed_at DESC
      LIMIT 1
    `, [purchase.user_id]);

    if (lastConfirmed.rows.length > 0 && lastConfirmed.rows[0].confirmed_at) {
      const lastTime = new Date(lastConfirmed.rows[0].confirmed_at).getTime();
      const now = Date.now();
      if (now - lastTime < MIN_SECONDS_BETWEEN_CONFIRMS * 1000) {
        return res.status(429).json({ error: 'Too many confirmations too quickly. Please wait a moment.' });
      }
    }

    let award = POINTS_PER_PURCHASE;
    const remainingCap = DAILY_POINTS_CAP - alreadyAwardedToday;
    if (award > remainingCap) award = remainingCap;

    if (award <= 0) {
      await pool.query(
        `UPDATE purchases
         SET status = 'confirmed',
             confirmed_at = NOW(),
             cashier_name = $1,
             awarded_points = 0
         WHERE id = $2`,
        [cashierName || null, purchase.id]
      );

      return res.json({
        success: true,
        message: 'Purchase confirmed, but the user reached the daily cap. No points awarded.'
      });
    }

    await pool.query('UPDATE users SET points = points + $1 WHERE id = $2', [award, purchase.user_id]);

    await pool.query(
      `UPDATE purchases
       SET status = 'confirmed',
           confirmed_at = NOW(),
           cashier_name = $1,
           awarded_points = $2
       WHERE id = $3`,
      [cashierName || null, award, purchase.id]
    );

    const userAfter = await pool.query('SELECT id, points FROM users WHERE id = $1', [purchase.user_id]);

    if ((userAfter.rows[0].points || 0) >= 1000) {
      const coupon = await createCouponForUser(pool, purchase.user_id, 'percent_off', 10, 60);
      await pool.query('UPDATE users SET points = 0 WHERE id = $1', [purchase.user_id]);

      return res.json({
        success: true,
        message: `Purchase confirmed. ${award} points awarded. 10% coupon created: ${coupon.code}`,
        awarded: award,
        coupon
      });
    }

    return res.json({
      success: true,
      message: `Purchase confirmed. ${award} points awarded.`,
      awarded: award
    });
  } catch (err) {
    console.error('confirm purchase error', err);
    return res.status(500).json({ error: 'Could not confirm purchase' });
  }
});

app.listen(PORT, () => {
  console.log('Server listening on http://localhost:' + PORT);
  initDb().catch((err) => {
    console.error('DB bootstrap failed', err);
    process.exit(1);
  });
});
<!doctype html>
<html>
<head>
  <meta charset="utf-8" />
  <title>The Hylton Hangout</title>
  <link rel="stylesheet" href="/styles.css" />
</head>
<body>
  <main class="card login-card">
    <h1>The Hylton Hangout</h1>
    <h2>Log in</h2>

    <form id="loginForm">
      <label>
        Username
        <input name="username" required />
      </label>

      <label>
        Password
        <input name="password" type="password" required />
      </label>

      <button type="submit">Log in</button>
    </form>

    <p>Need an account? <a href="/register.html">Create one</a></p>
    <div id="message"></div>
  </main>

  <script>
    document.getElementById('loginForm').addEventListener('submit', async (e) => {
      e.preventDefault();
      const form = e.target;
      const payload = {
        username: form.username.value,
        password: form.password.value
      };

      const res = await fetch('/api/login', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(payload)
      });

      const json = await res.json();

      if (res.ok) {
        window.location.href = '/dashboard.html';
      } else {
        document.getElementById('message').innerText = json.error || 'Login failed';
      }
    });
  </script>
</body>
</html>
<!doctype html>
<html>
<head>
  <meta charset="utf-8" />
  <title>Create Account | The Hylton Hangout</title>
  <link rel="stylesheet" href="/styles.css" />
</head>
<body>
  <main class="card login-card">
    <h1>The Hylton Hangout</h1>
    <h2>Create account</h2>

    <form id="registerForm">
      <label>
        Username
        <input name="username" required />
      </label>

      <label>
        Password
        <input name="password" type="password" required />
      </label>

      <button type="submit">Create account</button>
    </form>

    <p>Already have an account? <a href="/">Log in</a></p>
    <div id="message"></div>
  </main>

  <script>
    document.getElementById('registerForm').addEventListener('submit', async (e) => {
      e.preventDefault();
      const form = e.target;
      const payload = {
        username: form.username.value,
        password: form.password.value
      };

      const res = await fetch('/api/register', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(payload)
      });

      const json = await res.json();

      if (res.ok) {
        window.location.href = '/dashboard.html';
      } else {
        document.getElementById('message').innerText = json.error || 'Registration failed';
      }
    });
  </script>
</body>
</html>
<!doctype html>
<html>
<head>
  <meta charset="utf-8" />
  <title>Dashboard | The Hylton Hangout</title>
  <link rel="stylesheet" href="/styles.css" />
</head>
<body>
  <header class="topbar">
    <h1>The Hylton Hangout</h1>
    <button id="logoutBtn">Logout</button>
  </header>

  <main class="dashboard">
    <section class="card">
      <h2>Account</h2>
      <p><strong>Username:</strong> <span id="username"></span></p>
      <p><strong>Points:</strong> <span id="points"></span></p>

      <div id="couponBanner" class="coupon-banner" style="display:none;"></div>
      <button id="addPointsBtn">Add 100 points (demo)</button>
    </section>

    <section class="card">
      <h2>Daily Scratch Ticket</h2>
      <p id="scratchStatus"></p>
      <button id="openScratchBtn">Open Scratch Card</button>

      <div id="scratchModal" class="modal" style="display:none;">
        <div class="modal-content">
          <h3>Scratch to reveal</h3>
          <div class="scratch-area">
            <div id="scratchMessage" class="scratch-message">Hidden prize</div>
            <canvas id="scratchCanvas" width="320" height="100"></canvas>
          </div>
          <button id="closeScratch">Close</button>
        </div>
      </div>

      <div id="scratchResult" class="scratch-result"></div>
    </section>

    <section class="card">
      <h2>Drink Menu</h2>
      <ul id="menuList"></ul>

      <hr />

      <h3>Purchase</h3>
      <form id="purchaseForm">
        <label>
          Menu item
          <select id="menuSelect" name="menuId"></select>
        </label>

        <button type="submit">Purchase</button>
      </form>

      <div id="purchaseMessage" class="purchase-message" style="display:none;"></div>

      <h3>Your Coupons</h3>
      <ul id="couponList"></ul>
      <p class="small-note">
        For a 10% coupon earned by reaching 1000 points: show it to the cashier. When accepted, the cashier clicks "Discard" to mark it used.
      </p>
    </section>
  </main>

  <script>
    async function fetchUser() {
      const res = await fetch('/api/user');
      if (!res.ok) {
        window.location.href = '/';
        return;
      }
      const user = await res.json();

      document.getElementById('username').innerText = user.username;
      document.getElementById('points').innerText = user.points;

      const statusEl = document.getElementById('scratchStatus');
      const openBtn = document.getElementById('openScratchBtn');

      if (user.canScratch) {
        statusEl.innerText = 'You can scratch the daily ticket!';
        openBtn.disabled = false;
      } else {
        statusEl.innerText = 'You already scratched today. Come back tomorrow.';
        openBtn.disabled = true;
      }

      renderCoupons(user.coupons || []);
    }

    function renderCoupons(coupons) {
      const ul = document.getElementById('couponList');
      ul.innerHTML = '';

      if (!coupons.length) {
        ul.innerHTML = '<li>No active coupons.</li>';
        return;
      }

      coupons.forEach((coupon) => {
        const li = document.createElement('li');
        const expires = new Date(coupon.expires_at).toLocaleDateString();

        li.innerText = `${coupon.code} — ${coupon.type}${coupon.value ? ' ' + coupon.value + '%' : ''} — expires ${expires}${coupon.redeemed ? ' (redeemed)' : ''}`;

        if (!coupon.redeemed) {
          const redeemBtn = document.createElement('button');
          redeemBtn.innerText = 'Redeem';
          redeemBtn.addEventListener('click', async () => {
            const res = await fetch('/api/coupons/redeem', {
              method: 'POST',
              headers: { 'Content-Type': 'application/json' },
              body: JSON.stringify({ code: coupon.code })
            });
            const json = await res.json();

            if (res.ok) {
              alert('Coupon redeemed.');
              fetchUser();
            } else {
              alert(json.error || 'Unable to redeem coupon');
            }
          });

          li.appendChild(document.createTextNode(' '));
          li.appendChild(redeemBtn);

          if (coupon.type === 'percent_off') {
            const note = document.createElement('div');
            note.className = 'coupon-note';
            note.innerText = 'Show this coupon to the cashier. The cashier should click "Discard" after accepting it.';
            li.appendChild(note);

            const discardBtn = document.createElement('button');
            discardBtn.innerText = 'Discard (for cashier)';
            discardBtn.addEventListener('click', async () => {
              const res = await fetch('/api/coupons/discard', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({ code: coupon.code })
              });
              const json = await res.json();

              if (res.ok) {
                alert('Coupon discarded by cashier.');
                fetchUser();
              } else {
                alert(json.error || 'Unable to discard coupon');
              }
            });

            li.appendChild(document.createTextNode(' '));
            li.appendChild(discardBtn);
          }
        }

        ul.appendChild(li);
      });
    }

    async function fetchMenu() {
      const res = await fetch('/api/menu');
      const items = await res.json();

      const menuList = document.getElementById('menuList');
      const menuSelect = document.getElementById('menuSelect');
      menuList.innerHTML = '';
      menuSelect.innerHTML = '';

      items.forEach((item) => {
        const li = document.createElement('li');
        const price = (item.price_cents / 100).toFixed(2);
        li.innerText = `${item.name} — $${price} — ${item.description}`;
        menuList.appendChild(li);

        const opt = document.createElement('option');
        opt.value = item.id;
        opt.innerText = `${item.name} — $${price}`;
        menuSelect.appendChild(opt);
      });
    }

    document.getElementById('purchaseForm').addEventListener('submit', async (e) => {
      e.preventDefault();

      const menuId = document.getElementById('menuSelect').value;
      const msg = document.getElementById('purchaseMessage');

      const res = await fetch('/api/purchase', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ menuId })
      });

      const json = await res.json();

      if (res.ok) {
        msg.style.display = 'block';
        msg.className = 'purchase-message success';
        msg.innerText = `Purchase created. Show code ${json.confirmCode} to the cashier.`;
        alert(`Show this code to the cashier: ${json.confirmCode}`);
      } else {
        msg.style.display = 'block';
        msg.className = 'purchase-message error';
        msg.innerText = json.error || 'Could not create purchase';
      }
    });

    document.getElementById('addPointsBtn').addEventListener('click', async () => {
      const res = await fetch('/api/add-points', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ amount: 100 })
      });

      const json = await res.json();

      if (res.ok) {
        document.getElementById('points').innerText = json.points;
        if (json.coupon) {
          alert('You earned a 10% off coupon: ' + json.coupon.code);
        }
        fetchUser();
      } else {
        alert(json.error || 'Could not add points');
      }
    });

    document.getElementById('logoutBtn').addEventListener('click', async () => {
      await fetch('/api/logout', { method: 'POST' });
      window.location.href = '/';
    });

    const scratchModal = document.getElementById('scratchModal');
    const openScratchBtn = document.getElementById('openScratchBtn');
    const closeScratchBtn = document.getElementById('closeScratch');
    const scratchCanvas = document.getElementById('scratchCanvas');
    const scratchMessage = document.getElementById('scratchMessage');
    const scratchResult = document.getElementById('scratchResult');

    openScratchBtn.addEventListener('click', () => {
      scratchResult.innerText = '';
      scratchMessage.innerText = 'Scratch to reveal...';
      scratchModal.style.display = 'block';
      initScratchCanvas();
    });

    closeScratchBtn.addEventListener('click', () => {
      scratchModal.style.display = 'none';
    });

    function initScratchCanvas() {
      const canvas = scratchCanvas;
      const ctx = canvas.getContext('2d');

      ctx.clearRect(0, 0, canvas.width, canvas.height);
      ctx.globalCompositeOperation = 'source-over';
      ctx.fillStyle = '#d9d9d9';
      ctx.fillRect(0, 0, canvas.width, canvas.height);

      ctx.fillStyle = '#a6a6a6';
      ctx.fillRect(0, 0, canvas.width, canvas.height);

      ctx.globalCompositeOperation = 'destination-out';
      ctx.lineWidth = 20;
      ctx.lineCap = 'round';

      let drawing = false;

      function draw(event) {
        const rect = canvas.getBoundingClientRect();
        const clientX = (event.touches && event.touches[0]) ? event.touches[0].clientX : event.clientX;
        const clientY = (event.touches && event.touches[0]) ? event.touches[0].clientY : event.clientY;

        const x = clientX - rect.left;
        const y = clientY - rect.top;

        ctx.beginPath();
        ctx.arc(x, y, 12, 0, Math.PI * 2);
        ctx.fill();
      }

      function pointerDown(event) {
        drawing = true;
        draw(event);
      }

      function pointerMove(event) {
        if (!drawing) return;
        draw(event);
      }

      function pointerUp() {
        drawing = false;
        checkRevealed();
      }

      canvas.onpointerdown = pointerDown;
      canvas.onpointermove = pointerMove;
      canvas.onpointerup = pointerUp;
      canvas.ontouchstart = pointerDown;
      canvas.ontouchmove = pointerMove;
      canvas.ontouchend = pointerUp;
    }

    async function checkRevealed() {
      const canvas = scratchCanvas;
      const ctx = canvas.getContext('2d');
      const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height);
      let clearedPixels = 0;

      for (let i = 3; i < imageData.data.length; i += 4) {
        if (imageData.data[i] === 0) clearedPixels++;
      }

      const totalPixels = canvas.width * canvas.height;
      const percentCleared = (clearedPixels / totalPixels) * 100;

      if (percentCleared > 30) {
        canvas.onpointerdown = null;
        canvas.onpointermove = null;
        canvas.onpointerup = null;
        canvas.ontouchstart = null;
        canvas.ontouchmove = null;
        canvas.ontouchend = null;

        const res = await fetch('/api/scratch', { method: 'POST' });
        const json = await res.json();

        if (res.ok) {
          scratchMessage.innerText = json.message;
          scratchResult.innerText = json.message;
          if (json.coupon) {
            alert('Coupon created: ' + json.coupon.code);
          }
          fetchUser();
        } else {
          scratchMessage.innerText = json.error || 'Could not scratch';
          scratchResult.innerText = json.error || 'Could not scratch';
        }
      }
    }

    fetchMenu();
    fetchUser();
  </script>
</body>
</html>
<!doctype html>
<html>
<head>
  <meta charset="utf-8" />
  <title>Cashier | The Hylton Hangout</title>
  <link rel="stylesheet" href="/styles.css" />
</head>
<body>
  <main class="card login-card">
    <h1>Cashier</h1>
    <h2>Confirm Purchase</h2>

    <form id="cashierForm">
      <label>
        Confirm code
        <input id="confirmCode" required />
      </label>

      <label>
        Cashier passcode
        <input id="cashierPass" type="password" required value="198759" />
      </label>

      <label>
        Cashier name (optional)
        <input id="cashierName" />
      </label>

      <button type="submit">Confirm Purchase</button>
    </form>

    <div id="message"></div>
  </main>

  <script>
    document.getElementById('cashierForm').addEventListener('submit', async (e) => {
      e.preventDefault();

      const payload = {
        confirmCode: document.getElementById('confirmCode').value.trim(),
        cashierPass: document.getElementById('cashierPass').value,
        cashierName: document.getElementById('cashierName').value.trim()
      };

      const res = await fetch('/api/purchase/confirm', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(payload)
      });

      const json = await res.json();

      if (res.ok) {
        document.getElementById('message').innerText = json.message;
      } else {
        document.getElementById('message').innerText = json.error || 'Confirmation failed';
      }
    });
  </script>
</body>
</html>
:root {
  --bg: #f3efe8;
  --surface: #fffdf9;
  --primary: #5f3f2d;
  --secondary: #d7a86e;
  --text: #2b211d;
  --muted: #685a52;
  --success: #dff3df;
  --warning: #fff3c7;
  --danger: #fce1df;
  --shadow: rgba(30, 20, 15, 0.08);
}

* { box-sizing: border-box; }

body {
  margin: 0;
  font-family: Arial, Helvetica, sans-serif;
  background: var(--bg);
  color: var(--text);
}

a { color: var(--primary); }

.topbar {
  display: flex;
  justify-content: space-between;
  align-items: center;
  background: linear-gradient(90deg, #f8e8d4, #fff);
  padding: 16px 28px;
  box-shadow: 0 2px 8px var(--shadow);
}

.topbar h1 {
  margin: 0;
  font-size: 1.5rem;
}

.dashboard {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
  gap: 20px;
  padding: 24px;
}

.card {
  background: var(--surface);
  border-radius: 12px;
  padding: 20px;
  box-shadow: 0 4px 12px var(--shadow);
}

.login-card {
  max-width: 430px;
  margin: 80px auto;
}

h1, h2, h3 {
  margin-top: 0;
}

label {
  display: block;
  margin: 12px 0;
  font-weight: 600;
}

input, select, button {
  width: 100%;
  padding: 10px 12px;
  border-radius: 8px;
  border: 1px solid #d9d0c6;
  font-size: 1rem;
}

input, select {
  background: #fff;
  color: var(--text);
}

button {
  background: var(--primary);
  color: #fff;
  border: none;
  cursor: pointer;
  transition: 0.2s ease;
  font-weight: 700;
}

button:hover {
  opacity: 0.95;
}

button:disabled {
  opacity: 0.65;
  cursor: not-allowed;
}

#message {
  margin-top: 12px;
  color: var(--primary);
  font-weight: 600;
}

.coupon-banner {
  margin-top: 12px;
  background: var(--success);
  border: 1px solid #acc7ad;
  border-radius: 8px;
  padding: 10px 12px;
  font-weight: 600;
}

.scratch-result {
  margin-top: 12px;
  background: var(--warning);
  border-radius: 8px;
  padding: 10px 12px;
  font-weight: 600;
}

.purchase-message {
  margin-top: 12px;
  padding: 10px 12px;
  border-radius: 8px;
  font-weight: 600;
  display: none;
}

.purchase-message.success {
  background: var(--success);
  color: #1c4d2d;
}

.purchase-message.error {
  background: var(--danger);
  color: #7a2d2d;
}

.modal {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.45);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}

.modal-content {
  background: #fff;
  width: 360px;
  max-width: 90vw;
  padding: 18px;
  border-radius: 12px;
  box-shadow: 0 12px 28px rgba(0,0,0,0.2);
}

.scratch-area {
  position: relative;
  width: 320px;
  max-width: 100%;
  height: 100px;
  margin: 0 auto 12px;
  border-radius: 8px;
  overflow: hidden;
  background: linear-gradient(90deg, #fff7d9, #fff);
  display: flex;
  align-items: center;
  justify-content: center;
}

.scratch-message {
  position: absolute;
  z-index: 0;
  font-weight: 700;
  color: var(--primary);
}

#scratchCanvas {
  position: absolute;
  z-index: 1;
  left: 0;
  top: 0;
  width: 320px;
  height: 100px;
  max-width: 100%;
  border-radius: 8px;
}

#menuList, #couponList {
  padding-left: 20px;
  margin-top: 12px;
}

#menuList li, #couponList li {
  margin-bottom: 10px;
}

#couponList button {
  width: auto;
  margin-left: 8px;
}

.small-note {
  font-size: 0.9rem;
  color: var(--muted);
}

.coupon-note {
  margin-top: 8px;
  font-size: 0.9rem;
  color: var(--muted);
}
SESSION_SECRET=replace-with-secure-secret
NODE_ENV=development
DATABASE_URL=postgres://postgres:postgres@localhost:5432/hylton
DB_SSL=false
CASHIER_PASSCODE=198759
POINTS_PER_PURCHASE=100
DAILY_POINTS_CAP=500
MIN_SECONDS_BETWEEN_CONFIRMS=10
FROM node:18-alpine

WORKDIR /app

COPY package.json ./
RUN npm install --production

COPY . .

ENV NODE_ENV=production
ENV PORT=3000

EXPOSE 3000
CMD ["node", "server.js"]
version: "3.8"

services:
  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - SESSION_SECRET=replace-with-secure-secret
      - DATABASE_URL=postgres://postgres:postgres@db:5432/hylton
      - CASHIER_PASSCODE=198759
      - POINTS_PER_PURCHASE=100
      - DAILY_POINTS_CAP=500
      - MIN_SECONDS_BETWEEN_CONFIRMS=10
    depends_on:
      - db

  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: hylton
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
