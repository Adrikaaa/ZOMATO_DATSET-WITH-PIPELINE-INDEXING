# FoodHub API

A Zomato-style food ordering backend, built as a **teaching starter repo**.

Customers browse restaurants, place orders and leave reviews. Restaurant owners manage
their restaurants, menus and incoming orders, and (once you build it) see analytics
for their business.

The CRUD API and auth already work. Your job is to make it **fast** (indexes) and
**smart** (aggregation pipelines). To leave that work for you, this repo deliberately has:

- **no indexes** other than the default `_id` index (and `autoIndex: false` in the Mongoose connection)
- **no aggregation pipelines**. The analytics endpoints are stubs that return `501 Not Implemented`.

**Stack:** Node.js 20+, Express 5, MongoDB + Mongoose, JWT (access + refresh tokens), bcrypt, plain JavaScript (CommonJS).

---

## Table of contents

1. [Quick start](#quick-start)
2. [Architecture](#architecture)
3. [Data model](#data-model)
4. [Authentication](#authentication)
5. [API reference](#api-reference)
6. [Errors](#errors)
7. [Environment variables](#environment-variables)
8. [Seed data](#seed-data)
9. [Your tasks](#your-tasks)

---

## Quick start

Requirements: Node.js 20+ and a MongoDB server (local or Atlas).

```bash
npm install
cp .env.example .env      # then edit the secrets and MONGO_URI
npm run seed              # loads ~34 million documents, takes ~5 minutes (see "Seed data")
npm run dev               # starts the server with nodemon on http://localhost:3000
```

| Script         | What it does                      |
| -------------- | --------------------------------- |
| `npm run dev`  | Start the server with auto-reload |
| `npm start`    | Start the server                  |
| `npm run seed` | Drop all collections and reseed   |

Check it is running:

```bash
curl http://localhost:3000/
# { "message": "FoodHub API is running" }
```

### Demo accounts

Password for both: **`password123`** (all other seeded users have it too).

| Role     | Email                  | Notes                                                                           |
| -------- | ---------------------- | ------------------------------------------------------------------------------- |
| owner    | `owner@foodhub.com`    | Owns 10 restaurants, including the most popular one (in Bhopal, ~5,000 orders) |
| customer | `customer@foodhub.com` | Lives in Bhopal and has about 500 orders                                        |

The seed prints the demo restaurant's `_id` at the end. Use it in the restaurant routes below.

---

## Architecture

FoodHub is a classic **layered Express app**. A request passes through each layer
in turn, and each layer has one job.

```
                      HTTP request
                           │
┌──────────────────────────▼──────────────────────────┐
│ app.js            global middleware                 │
│                   express.json · cookieParser · morgan
├─────────────────────────────────────────────────────┤
│ routes/*.routes.js   match METHOD + path            │
│                      attach per-route middleware    │
├─────────────────────────────────────────────────────┤
│ middlewares/auth     authUser   → who are you?  (401)
│                      authorize  → right role?   (403)
├─────────────────────────────────────────────────────┤
│ controllers/*        validate input, check ownership│
│                      (utils/ownership.js), run query│
├─────────────────────────────────────────────────────┤
│ models/*             Mongoose schemas               │
└──────────────────────────┬──────────────────────────┘
                           ▼
                        MongoDB

   Anything thrown along the way ─► middlewares/error.middleware.js
   No route matched              ─► notFound (404)
```

### Layers

| Layer           | Folder / file                     | Responsibility                                                                                         |
| --------------- | --------------------------------- | ------------------------------------------------------------------------------------------------------ |
| Entry point     | `src/server.js`                   | Checks the JWT secrets exist, connects to MongoDB, then starts listening                               |
| App             | `src/app.js`                      | Registers global middleware and mounts every router under `/api`                                       |
| Config          | `src/config/env.js`               | **The only file that reads `process.env`.** Exports a frozen settings object                           |
|                 | `src/config/db.js`                | Opens the Mongoose connection with `autoIndex: false`                                                  |
| Routes          | `src/routes/`                     | Map URLs to controllers and declare who may call them. No business logic here                          |
| Middleware      | `src/middlewares/auth.middleware.js` | `authUser` verifies the Bearer token and sets `req.user = { id, role }`. `authorize(...roles)` checks the role |
|                 | `src/middlewares/error.middleware.js` | `notFound` (404) and the central `errorHandler`                                                    |
| Controllers     | `src/controllers/`                | Request handlers. Validate input, check ownership, talk to models directly, send JSON                  |
| Models          | `src/models/`                     | Mongoose schemas: User, Restaurant, MenuItem, Order, Review                                            |
| Utils           | `src/utils/token.js`              | Sign access/refresh JWTs, hash and compare refresh tokens                                              |
|                 | `src/utils/pagination.js`         | Turns `?page` and `?limit` into safe `{ page, limit, skip }` (default 10, max 50)                      |
|                 | `src/utils/ownership.js`          | `findOwnedRestaurant(id, userId)`: loads a restaurant and checks it belongs to the caller              |
| Seed            | `src/seed/`                       | Streams millions of fake documents into MongoDB                                                        |

### Design decisions worth knowing

- **Two levels of access control.** The route says *which role* may call it
  (`authorize("owner")`). The controller then checks *which resource*: an owner can
  only change their **own** restaurant, via `findOwnedRestaurant`. Both checks are needed.
  Without the second one, any owner could edit any restaurant.
- **No try/catch in controllers.** Express 5 forwards errors thrown in `async` handlers
  to the error middleware automatically, so a bad ObjectId becomes a clean `400` without
  extra code.
- **Order items are snapshots.** When an order is placed, the server copies each dish's
  `name` and `price` from the database into the order. Old orders stay correct when a menu
  changes, and the client can never set its own price.
- **Analytics router is mounted before the restaurant router.** Otherwise
  `/api/restaurants/search` would match `/api/restaurants/:id` with `id = "search"`.
- **Deleting a restaurant deletes its menu**, but keeps its orders and reviews for history.

### Project structure

```
src/
  server.js                    Entry point: connect DB, start server
  app.js                       Express app, global middleware, route mounting
  config/
    env.js                     Loads .env and exports every setting
    db.js                      Mongoose connection (autoIndex: false)
  routes/
    auth.routes.js             /api/auth/*
    restaurant.routes.js       /api/restaurants/* (restaurants, menu, orders, reviews, revenue)
    order.routes.js            /api/orders/*
    review.routes.js           /api/reviews/*
    analytics.routes.js        /api/analytics/*, /api/restaurants/search, /api/restaurants/nearby
  controllers/
    auth.controller.js
    restaurant.controller.js
    menu.controller.js
    order.controller.js
    review.controller.js
    analytics.controller.js    Stubs: your tasks
  middlewares/
    auth.middleware.js         authUser, authorize
    error.middleware.js        notFound, errorHandler
  models/                      user, restaurant, menuItem, order, review
  utils/                       token.js, pagination.js, ownership.js
  seed/                        seed.js and its static data (data.js)
```

---

## Data model

```
User (customer | owner)
 │ 1
 │ owns (owner)
 │ N
Restaurant ──1:N──► MenuItem
 │ 1
 │ N
Order ◄── placed by ── User (customer)
 │  items[] = snapshot { menuItem, name, price, quantity }
 │ 1
 │ 0..1
Review  (customer, restaurant, order, rating 1–5)
```

| Model        | Key fields                                                                                           | Notes                                                                   |
| ------------ | ---------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------- |
| `User`       | `name`, `email`, `password`, `role` (`customer` / `owner`), `city`, `refreshToken`                  | `password` and `refreshToken` are hashed and `select: false`           |
| `Restaurant` | `name`, `owner → User`, `city`, `area`, `cuisines[]`, `location` (GeoJSON Point), `isOpen`          | Coordinates are `[lng, lat]`, longitude first                           |
| `MenuItem`   | `restaurant → Restaurant`, `name`, `category`, `price`, `isVeg`, `isAvailable`                       | Separate collection (not embedded), ~20 per restaurant                  |
| `Order`      | `customer → User`, `restaurant → Restaurant`, `items[]`, `totalAmount`, `status`, `paymentMethod`   | `status`: `placed`, `preparing`, `delivered`, `cancelled`               |
| `Review`     | `customer → User`, `restaurant → Restaurant`, `order → Order`, `rating`, `comment`                  | One per delivered order                                                 |

All models have `createdAt` (no `updatedAt`).

### Order status flow

```
placed ──► preparing ──► delivered
  │            │
  └────────────┴──────► cancelled
```

`delivered` and `cancelled` are final. Any other move returns `400`.

---

## Authentication

FoodHub uses a short-lived **access token** and a long-lived **refresh token**.

| Token         | Lifetime | Where it lives                                                                                    | Payload        |
| ------------- | -------- | ------------------------------------------------------------------------------------------------- | -------------- |
| Access token  | 15 min   | JSON response body. Keep it in memory and send it as `Authorization: Bearer <token>`             | `{ id, role }` |
| Refresh token | 7 days   | `httpOnly` cookie named `refreshToken` (`sameSite: strict`, path `/api/auth`, `secure` in production) | `{ id }`   |

```
Client                                   Server
  │  POST /api/auth/login                  │
  │ ─────────────────────────────────────► │  check password (bcrypt)
  │ ◄───────────────────────────────────── │  body: accessToken
  │                                        │  cookie: refreshToken (hash saved on user)
  │                                        │
  │  GET /api/... + Bearer accessToken     │
  │ ─────────────────────────────────────► │  authUser verifies JWT
  │ ◄── 401 "Access token expired" ─────── │  (after 15 min)
  │                                        │
  │  POST /api/auth/refresh (cookie sent)  │
  │ ─────────────────────────────────────► │  verify JWT + compare with stored hash
  │ ◄───────────────────────────────────── │  new accessToken + NEW refreshToken
  │                                        │  (old refresh token stops working)
  │  POST /api/auth/logout                 │
  │ ─────────────────────────────────────► │  stored hash set to null, cookie cleared
```

Only a hash of the **current** refresh token is stored (`user.refreshToken`). Because
the token is rotated on every refresh, a stolen refresh token that has already been used
is rejected. (bcrypt only reads the first 72 bytes of its input, so the token is first
SHA-256-digested, see `utils/token.js`.)

Example with curl:

```bash
# Log in, save the refresh cookie, copy the accessToken from the response
curl -c cookies.txt -H "Content-Type: application/json" \
  -d '{"email":"owner@foodhub.com","password":"password123"}' \
  http://localhost:3000/api/auth/login

# Call a protected route
curl -H "Authorization: Bearer <accessToken>" http://localhost:3000/api/auth/me

# Get a new access token (also rotates the cookie)
curl -b cookies.txt -c cookies.txt -X POST http://localhost:3000/api/auth/refresh
```

---

## API reference

Base URL: `http://localhost:3000`

**Auth column:**
**public** = no token ·
**cookie** = uses the `refreshToken` cookie ·
**user** = any logged-in user ·
**owner** / **customer** = that role only.
Routes marked **owner** on a restaurant also check that **you own that restaurant** (`403` otherwise).

**Paginated lists** accept `?page` (default 1) and `?limit` (default 10, max 50) and respond with:

```json
{ "page": 1, "limit": 10, "total": 523, "totalPages": 53, "<items>": [ ... ] }
```

### Overview

| Method | Path                                         | Auth     | Purpose                              |
| ------ | -------------------------------------------- | -------- | ------------------------------------ |
| GET    | `/`                                          | public   | Health check                         |
| POST   | `/api/auth/register`                         | public   | Create an account                    |
| POST   | `/api/auth/login`                            | public   | Log in                               |
| POST   | `/api/auth/refresh`                          | cookie   | New access token                     |
| POST   | `/api/auth/logout`                           | cookie   | Log out                              |
| GET    | `/api/auth/me`                               | user     | Current user                         |
| GET    | `/api/restaurants`                           | public   | List restaurants                     |
| GET    | `/api/restaurants/:id`                       | public   | One restaurant                       |
| POST   | `/api/restaurants`                           | owner    | Create a restaurant                  |
| PATCH  | `/api/restaurants/:id`                       | owner    | Update your restaurant               |
| DELETE | `/api/restaurants/:id`                       | owner    | Delete your restaurant               |
| GET    | `/api/restaurants/:id/menu`                  | public   | A restaurant's menu                  |
| POST   | `/api/restaurants/:id/menu`                  | owner    | Add a menu item                      |
| PATCH  | `/api/restaurants/:id/menu/:itemId`          | owner    | Update a menu item                   |
| DELETE | `/api/restaurants/:id/menu/:itemId`          | owner    | Delete a menu item                   |
| GET    | `/api/restaurants/:id/orders`                | owner    | Orders for your restaurant           |
| GET    | `/api/restaurants/:id/reviews`               | public   | Reviews for a restaurant             |
| GET    | `/api/restaurants/:id/revenue`               | owner    | Daily revenue (stub, Task 1)         |
| POST   | `/api/orders`                                | customer | Place an order                       |
| GET    | `/api/orders/my`                             | customer | Your orders                          |
| PATCH  | `/api/orders/:id/status`                     | owner    | Move an order to its next status     |
| POST   | `/api/reviews`                               | customer | Review a delivered order             |
| GET    | `/api/analytics/...`                         | owner/user | Analytics (stubs, see below)       |
| GET    | `/api/restaurants/search`                    | public   | Search (stub, Task 9)                |
| GET    | `/api/restaurants/nearby`                    | public   | Nearby (stub, Task 10)               |

---

### Auth

#### `POST /api/auth/register` — public

Creates an account, logs it in and sets the refresh cookie.

| Body field | Type   | Required | Notes                                    |
| ---------- | ------ | -------- | ---------------------------------------- |
| `name`     | string | yes      |                                          |
| `email`    | string | yes      | Stored lowercase and trimmed             |
| `password` | string | yes      | At least 6 characters                    |
| `role`     | string | no       | `customer` (default) or `owner`          |
| `city`     | string | no       |                                          |

```json
// 201
{
  "message": "Registered successfully",
  "user": { "id": "...", "name": "Asha", "email": "asha@example.com", "role": "customer", "city": "Pune", "createdAt": "..." },
  "accessToken": "eyJ..."
}
```

Errors: `400` missing fields / short password / bad role · `409` email already registered.

#### `POST /api/auth/login` — public

Body: `{ "email", "password" }`. Returns the same shape as register (status `200`) and
sets the refresh cookie.
Errors: `400` missing fields · `401` invalid email or password (same message for both, so
attackers can't tell which emails exist).

#### `POST /api/auth/refresh` — cookie

No body. Reads the `refreshToken` cookie, checks it against the stored hash, and returns
`{ "accessToken": "..." }`. Also sets a **new** refresh cookie; the old one is now invalid.
Errors: `401` cookie missing, expired, invalid, or already rotated (the cookie is cleared).

#### `POST /api/auth/logout` — cookie

No body. Clears the stored refresh token hash and the cookie. Always returns
`200 { "message": "Logged out successfully" }`, even if the cookie was already invalid.

#### `GET /api/auth/me` — user

Returns `{ "user": { id, name, email, role, city, createdAt } }` for the token's owner.
Errors: `401` no/expired/invalid token · `404` user was deleted.

---

### Restaurants

#### `GET /api/restaurants` — public

Paginated list, sorted by name.

| Query   | Notes                          |
| ------- | ------------------------------ |
| `page`  | default 1                      |
| `limit` | default 10, max 50             |
| `city`  | exact match, e.g. `Bhopal`     |

Response: `{ page, limit, total, totalPages, restaurants: [...] }`

#### `GET /api/restaurants/:id` — public

Returns `{ "restaurant": {...} }` with `owner` populated as `{ _id, name }`.
Errors: `400` invalid id · `404` not found.

#### `POST /api/restaurants` — owner

The logged-in owner becomes the restaurant's owner.

| Body field | Type     | Required | Notes                             |
| ---------- | -------- | -------- | --------------------------------- |
| `name`     | string   | yes      |                                   |
| `city`     | string   | yes      |                                   |
| `area`     | string   | no       |                                   |
| `cuisines` | string[] | no       | e.g. `["Biryani", "North Indian"]` |
| `lng`      | number   | yes      | −180 to 180                       |
| `lat`      | number   | yes      | −90 to 90                         |
| `isOpen`   | boolean  | no       | default `true`                    |

Returns `201 { "restaurant": {...} }`. `lng`/`lat` are stored as a GeoJSON point.
Errors: `400` invalid coordinates or validation failed.

#### `PATCH /api/restaurants/:id` — owner (own restaurant)

Send any of `name`, `city`, `area`, `cuisines`, `isOpen`. To move the restaurant, send
**both** `lng` and `lat`. Other fields (like `owner`) are ignored.
Returns `{ "restaurant": {...} }`.
Errors: `400` bad coordinates · `403` not your restaurant · `404` not found.

#### `DELETE /api/restaurants/:id` — owner (own restaurant)

Deletes the restaurant **and all its menu items**. Orders and reviews are kept.
Returns `{ "message": "Restaurant deleted" }`.

---

### Menu

#### `GET /api/restaurants/:id/menu` — public

The full menu (not paginated), sorted by category then name.

| Query      | Notes                              |
| ---------- | ---------------------------------- |
| `category` | exact match, e.g. `Starters`       |
| `isVeg`    | `true` or `false`                  |

```json
{
  "restaurant": { "id": "...", "name": "Royal Biryani House" },
  "count": 18,
  "menuItems": [ { "_id": "...", "name": "Paneer Tikka", "category": "Starters", "price": 240, "isVeg": true, "isAvailable": true } ]
}
```

> Without an index on `menuitems.restaurant`, this scans ~20 million documents. A good first index to add.

#### `POST /api/restaurants/:id/menu` — owner (own restaurant)

Body: `name` (required), `price` (required, ≥ 0), `category` (default `Other`),
`isVeg` (default `true`), `isAvailable` (default `true`).
Returns `201 { "menuItem": {...} }`.

#### `PATCH /api/restaurants/:id/menu/:itemId` — owner (own restaurant)

Send any of `name`, `category`, `price`, `isVeg`, `isAvailable`.
Returns `{ "menuItem": {...} }`. `404` if the item doesn't belong to this restaurant.

#### `DELETE /api/restaurants/:id/menu/:itemId` — owner (own restaurant)

Returns `{ "message": "Menu item deleted" }`. Existing orders keep their snapshot of the dish.

---

### Orders

#### `POST /api/orders` — customer

```json
{
  "restaurantId": "665f...",
  "paymentMethod": "upi",
  "items": [
    { "menuItemId": "665f...01", "quantity": 2 },
    { "menuItemId": "665f...02", "quantity": 1 }
  ]
}
```

| Field           | Notes                                                       |
| --------------- | ----------------------------------------------------------- |
| `restaurantId`  | required                                                    |
| `items`         | at least one; `quantity` is a whole number from 1 to 20     |
| `paymentMethod` | `upi`, `card` or `cod` (default)                            |

The server loads the menu items, copies their **current name and price**, and computes
`totalAmount`. The new order starts as `placed`. Returns `201 { "order": {...} }`.

Errors: `400` bad body, restaurant closed, or an item that is unavailable / from another
restaurant · `404` restaurant not found.

#### `GET /api/orders/my` — customer

Your orders, newest first, paginated. `restaurant` is populated with `name city area`.
Response: `{ page, limit, total, totalPages, orders: [...] }`

#### `GET /api/restaurants/:id/orders` — owner (own restaurant)

Orders for your restaurant, newest first, paginated. `customer` is populated with `name`.
Optional `?status=placed|preparing|delivered|cancelled`.

#### `PATCH /api/orders/:id/status` — owner (order's restaurant must be yours)

Body: `{ "status": "preparing" }`. Allowed moves:

| From        | To                         |
| ----------- | -------------------------- |
| `placed`    | `preparing`, `cancelled`   |
| `preparing` | `delivered`, `cancelled`   |
| `delivered` | (none)                     |
| `cancelled` | (none)                     |

Returns `{ "order": {...} }`.
Errors: `400` move not allowed · `403` not your restaurant · `404` order not found.

---

### Reviews

#### `POST /api/reviews` — customer

Body: `{ "orderId": "...", "rating": 5, "comment": "Loved it!" }`

Rules: `rating` is a whole number 1–5, the order must be **yours**, it must be
**delivered**, and you can review each order **only once**.
Returns `201 { "review": {...} }`.
Errors: `400` bad rating / not delivered · `403` not your order · `404` order not found · `409` already reviewed.

#### `GET /api/restaurants/:id/reviews` — public

Reviews for a restaurant, newest first, paginated. `customer` is populated with `name`.
Response: `{ page, limit, total, totalPages, reviews: [...] }`

---

### Analytics (stubs, these are your tasks)

All of these currently return `501 { "message": "Not implemented yet" }`. Each stub in
[`src/controllers/analytics.controller.js`](src/controllers/analytics.controller.js) has
the exact spec and an example response.

| Method | Path                                                 | Auth   | Task | Returns                                                           |
| ------ | ---------------------------------------------------- | ------ | ---- | ----------------------------------------------------------------- |
| GET    | `/api/analytics/restaurants/:id/revenue?from=&to=`   | owner  | 1    | Revenue and order count per day (default last 30 days)            |
| GET    | `/api/restaurants/:id/revenue?from=&to=`             | owner  | 1    | Same handler as above, under the restaurant URL                   |
| GET    | `/api/analytics/restaurants/:id/top-dishes?limit=`   | owner  | 2    | Best-selling dishes by quantity and revenue (default 5)           |
| GET    | `/api/analytics/restaurants/:id/peak-hours`          | owner  | 3    | Orders for each hour 0–23 (IST) and the peak hour                 |
| GET    | `/api/analytics/restaurants/:id/status-breakdown`    | owner  | 4    | Count and percentage per status                                   |
| GET    | `/api/analytics/restaurants/:id/top-customers`       | owner  | 5    | Top 10 customers by amount spent, with name and email             |
| GET    | `/api/analytics/restaurants/:id/ratings`             | owner  | 6    | Average, per-star counts, 5 latest reviews                        |
| GET    | `/api/analytics/platform/city-revenue`               | user   | 7    | Revenue and orders per city, highest first                        |
| GET    | `/api/analytics/platform/monthly-trend`              | user   | 8    | Revenue and orders per month, last 12 months                      |
| GET    | `/api/restaurants/search?q=&city=&cuisine=&page=`    | public | 9    | Results + total + count per cuisine, in one response              |
| GET    | `/api/restaurants/nearby?lng=&lat=&radius=`          | public | 10   | Restaurants within `radius` metres (default 5000), nearest first  |

Platform analytics are open to any logged-in user because there is no admin role yet.

---

## Errors

Every error is JSON with a `message`:

```json
{ "message": "Restaurant not found" }
```

| Status | When                                                                                  |
| ------ | ------------------------------------------------------------------------------------- |
| `400`  | Bad input, invalid ObjectId (`Invalid _id: abc`), invalid JSON, or schema validation failed (adds an `errors` array) |
| `401`  | Token missing, invalid or expired (`"Access token expired"` means call `/api/auth/refresh`) |
| `403`  | Wrong role, or the resource isn't yours                                               |
| `404`  | Resource not found, or unknown route (`Route not found: GET /api/foo`)                |
| `409`  | Duplicate email, or order already reviewed                                            |
| `500`  | Anything unexpected (logged on the server)                                            |
| `501`  | Analytics stub not implemented yet                                                    |

---

## Environment variables

| Variable               | Example                             | Notes                                                    |
| ---------------------- | ----------------------------------- | -------------------------------------------------------- |
| `PORT`                 | `3000`                              |                                                          |
| `NODE_ENV`             | `development`                       | The refresh cookie is `secure` only in `production`      |
| `MONGO_URI`            | `mongodb://127.0.0.1:27017/foodhub` | Required                                                 |
| `ACCESS_TOKEN_SECRET`  | long random string                  | Required. Generate with `openssl rand -hex 32`           |
| `REFRESH_TOKEN_SECRET` | another long random string          | Required. Must differ from the access secret             |
| `ACCESS_TOKEN_EXPIRY`  | `15m`                               |                                                          |
| `REFRESH_TOKEN_EXPIRY` | `7d`                                |                                                          |
| `CUSTOMER_COUNT`       | `900000`                            | Seed only                                                |
| `OWNER_COUNT`          | `100000`                            | Seed only                                                |
| `RESTAURANT_COUNT`     | `1000000`                           | Seed only. Each gets 15 to 25 menu items                 |
| `ORDER_COUNT`          | `10000000`                          | Seed only. Lower it on a slow machine                    |
| `REVIEW_COUNT`         | `2000000`                           | Seed only. Must be below ~85% of `ORDER_COUNT`           |
| `SEED_END_DATE`        | `2026-06-30`                        | Seed only. Orders cover the 12 months before this date. Empty means today |

---

## Seed data

`npm run seed` uses Faker with a fixed seed, so everyone gets the same data. Orders are placed
relative to today, though, so if you want the whole class to have **byte-identical** data
(same dates, same `_id`s), set the same `SEED_END_DATE` for everyone.

| Collection  | Count                                         | Data size (approx.) |
| ----------- | --------------------------------------------- | ------------------- |
| users       | 1,000,000 (900,000 customers, 100,000 owners) | 0.25 GB             |
| restaurants | 1,000,000                                     | 0.27 GB             |
| menuitems   | ~20,000,000 (15 to 25 per restaurant)         | 2.5 GB              |
| orders      | 10,000,000                                    | 3.3 GB              |
| reviews     | 2,000,000                                     | 0.3 GB              |

That is about **34 million documents**, ~6.6 GB of data and ~2.2 GB on disk (WiredTiger
compresses it). On a modern laptop the seed takes about **5 minutes** and uses well under
2 GB of memory: documents are streamed to MongoDB in batches of 5,000. Progress is logged
roughly every 1% per collection.

You need **at least 5 GB of free disk space**. On a slow machine, or on the free Atlas tier
(512 MB limit), use a smaller data set by putting these lines in `.env`:

```bash
# Small preset: ~370,000 documents, takes a few seconds
CUSTOMER_COUNT=4700
OWNER_COUNT=300
RESTAURANT_COUNT=300
ORDER_COUNT=300000
REVIEW_COUNT=60000
```

**Why so big?** Without indexes, every query that filters or sorts has to scan the whole
collection. With millions of documents you will *feel* that (for example
`GET /api/restaurants/:id/menu` scans 20 million menu items and takes several seconds),
and you will see the difference once you add the right index. Use `.explain("executionStats")`
to compare `totalDocsExamined` before and after.

Restaurant names repeat (like branches of a chain), so never look a restaurant up by name.
Use its `_id`.

The data is shaped so that analytics give interesting answers:

- 8 Indian cities (Bhopal, Indore, Delhi, Mumbai, Bengaluru, Pune, Hyderabad, Jaipur). Restaurants are placed within about 10 km of the city centre as GeoJSON points.
- Orders cover the last 12 months, with busier weekends and slow growth over time.
- Order times peak at lunch (12 to 2 pm) and dinner (7 to 10 pm), in IST.
- About 20% of restaurants get about 60% of all orders.
- Customers mostly order from restaurants in their own city.
- Each order has 1 to 4 items, and `totalAmount` always equals the sum of `price × quantity`.
- About 85% of orders are delivered and 8% cancelled. The rest are placed or preparing.
- Reviews exist only for delivered orders, and ratings lean towards 4 and 5.

> ⚠️ Re-running the seed **drops every collection, including any indexes you created.**
> Keep your index definitions in code (or a script) so you can re-create them.

---

## Your tasks

Implement every stub in [`src/controllers/analytics.controller.js`](src/controllers/analytics.controller.js)
with **aggregation pipelines**, then add the **indexes** that make them fast.

1. **Daily revenue**: revenue and order count per day for a restaurant, between `from` and `to`.
2. **Top dishes**: best-selling dishes by quantity sold and by revenue.
3. **Peak hours**: order count for each hour of the day (0 to 23).
4. **Status breakdown**: count and percentage of orders in each status.
5. **Top customers**: the 10 customers who spent the most, with their name and email.
6. **Ratings summary**: average rating, count for each star, and the 5 latest reviews with the customer's name.
7. **City revenue (platform)**: revenue and order count for each city, sorted.
8. **Monthly trend (platform)**: revenue for each of the last 12 months.
9. **Search**: one response with the results, the total count, and the count per cuisine (hint: `$facet`, text index).
10. **Nearby (stretch)**: restaurants near a point, with their distance (hint: `2dsphere` index, `$geoNear`).

Tips:

- Usually only `delivered` orders count as revenue. Decide on a rule and stick to it.
- For restaurant analytics, check ownership with `findOwnedRestaurant` (see `src/utils/ownership.js`).
- Convert ids for `$match` with `new mongoose.Types.ObjectId(req.params.id)`. Aggregations don't cast for you.
- Group by date and hour in the `Asia/Kolkata` timezone (`$dateToString` / `$hour` accept a `timezone`).
- Check every query with `.explain("executionStats")` before and after adding an index, and compare
  `totalDocsExamined` with `nReturned`.
- Good candidates to think about: a unique index on `users.email`, `{ restaurant: 1 }` on menu items,
  `{ restaurant: 1, createdAt: -1 }` on orders and reviews, `{ customer: 1, createdAt: -1 }` on orders,
  `{ order: 1 }` on reviews, a text index on restaurants, and a `2dsphere` index on `restaurants.location`.
- Mongoose's `autoIndex` is `false`, so indexes you declare in a schema are **not** built automatically.
  Call `Model.syncIndexes()` / `createIndexes()`, or create them in `mongosh`.
