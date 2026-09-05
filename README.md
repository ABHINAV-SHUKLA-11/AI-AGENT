# Retail AI Agent — Abhinav Shukla

> Rule-based + Gemini-powered retail management agent for local shopkeepers, with Razorpay payment links.
> Built for **Razorpay Buildathon**. Running locally (MySQL) for demo.

## What it does

A chat-style agent (`/query`) that understands plain-language retail commands and falls back to
Gemini 2.5 Flash for anything it doesn't recognize. Backed by MySQL, not MongoDB — every "collection"
operation (`find`, `insert_one`, `update_one`, etc.) is actually translated to SQL under the hood via a
small compatibility layer in `main.py` (`Table` / `Database` classes), so the agent logic can stay
Mongo-style without touching a real MongoDB instance.

### Features
- Product search, price filter, rating filter, category filter
- Create / delete / update orders via natural language
- Stock auto-decrement on order, auto-restore on delete/cancel
- GST invoice generation (18%)
- Revenue report & store dashboard
- Low stock alerts, top-rated products, "who bought X"
- **Razorpay test-mode payment links** on order creation, with webhook-based payment confirmation
- Gemini 2.5 Flash fallback for anything the regex rules don't catch

## Tech stack (actual, as run locally)

| Layer | Technology |
|-------|-----------|
| Backend | Python Flask (`main.py`) |
| Database | MySQL (via PyMySQL — a Mongo-style shim sits on top, see `Table`/`Database` in `main.py`) |
| AI fallback | Gemini 2.5 Flash (`google-genai` SDK) |
| Payments | Razorpay Payment Links + Webhooks (test mode) |
| Frontend | Static `index.html` served at `/ui` |
| Server | Flask dev server locally; Gunicorn in `Dockerfile` (not used for this local demo) |

> Note: `src/app.py`, `src/config.py`, `src/mongodb_mcp_server.py`, `src/models.py`, `src/deploy.py`
> are leftovers from an earlier MongoDB + Google Agent Builder version and are **not used** by the
> running app. `src/config.py` in particular is not valid Python right now (it has stray Markdown in
> it) — don't import it. Only `main.py` matters for this build.

## Setup (local)

1. Create a MySQL database and note its host/user/password/db name.
2. Create a `.env` file in the project root:
   ```
   MYSQL_HOST=localhost
   MYSQL_PORT=3306
   MYSQL_USER=your_user
   MYSQL_PASSWORD=your_password
   MYSQL_DATABASE=retail_db
   MYSQL_SSL=false

   GEMINI_API_KEY=your_gemini_key
   GEMINI_MODEL=gemini-2.5-flash

   RAZORPAY_KEY_ID=your_test_key_id
   RAZORPAY_KEY_SECRET=your_test_key_secret
   RAZORPAY_WEBHOOK_SECRET=your_webhook_secret

   PORT=8080
   ```
   All three integrations (MySQL, Gemini, Razorpay) degrade gracefully if their env vars are missing —
   the agent still runs rule-based / Cash-on-Delivery only.
3. Install dependencies:
   ```bash
   pip install -r requirements.txt
   pip install razorpay
   ```
   **`razorpay` is used in `main.py` but is missing from `requirements.txt` — install it manually, or
   add it to the file, or every order-creation call will throw on a fresh install.**
4. Run:
   ```bash
   python main.py
   ```
   Tables (`products`, `orders`, `payments`) are auto-created on first run if they don't exist.
5. Open `http://localhost:8080/ui` for the chat UI, or POST to `http://localhost:8080/query`.

## Example commands

```
show all products
products under $100
top rated products
low stock alerts
revenue report
store dashboard
orders for John Doe
bill for ORD-001
create order for Amit product Nike Air Max qty 2
delete order Rahul Kumar
mark ORD-001 as delivered
who bought Nike Air Max
payment status ORD-001
```

## API endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/ui` | Chat frontend |
| GET | `/health` | Health check (includes MySQL connectivity) |
| POST | `/query` | Send a message to the agent |
| GET/POST | `/products` | List / add products |
| GET/POST | `/orders` | List / create orders |
| DELETE/PUT | `/orders/<order_id>` | Delete / update an order |
| POST | `/razorpay/webhook` | Razorpay payment confirmation webhook |
| GET | `/payments/<order_id>` | Payment attempt audit trail for an order |
| GET | `/mcp/tools` | Tool manifest (descriptive only) |

## Known issues / TODO before shipping beyond local demo

- Add `razorpay` to `requirements.txt`.
- Remove or clearly archive the dead `src/` files (old MongoDB/Agent Builder version) so they don't
  confuse contributors or judges reading the repo.
- Fix `src/config.py` — currently contains non-Python Markdown content, will fail to import as-is.
- `vercel.json` and `Dockerfile` describe two different, currently-unused deployment paths — this
  README covers local-only for now; pick one when you actually deploy.
