# Grocery Store API

**Version:** 1.0.0 · **OpenAPI:** 3.0.3

API contract for the 2-hour MERN grocery store (DDD-lite, API-first).

> **Contract rule:** frozen at 0:15. Changes go through the lead and are announced to the team.

## Server

| URL | Description |
|---|---|
| `http://localhost:5000/api` | Local development |

## Conventions

- All request and response bodies are JSON.
- IDs are 24-character hex strings (MongoDB ObjectIds).
- **Money is an integer in minor units (paisa).** `25000` means Rs 250.00. Never use floats for money.
- Quantities are numbers because some products are sold by weight (for example `0.5` kg).
- Authenticated routes need `Authorization: Bearer <JWT>`.
- Every error uses the [`Error`](#error) schema with a machine-readable `code`.
- Lists are paginated with `page` and `limit` and return a `pagination` object.

## Authentication

Security scheme `bearerAuth`: HTTP bearer, format JWT. Applied globally; endpoints marked **Public** below override it with no auth.

## Tags and owners

| Tag | Description | Owner |
|---|---|---|
| Identity | Registration, login, current user | P2 |
| Catalog | Browse and manage grocery products | P3 |
| Cart | The logged-in user's cart | P2 |
| Ordering | Checkout, mock payment, order history | P4 |

## Endpoint summary

| Method | Path | Operation ID | Auth | Summary |
|---|---|---|---|---|
| POST | `/auth/register` | registerUser | Public | Create a customer account |
| POST | `/auth/login` | loginUser | Public | Log in with email and password |
| GET | `/auth/me` | getCurrentUser | Bearer | Get the logged-in user |
| GET | `/products` | listProducts | Public | List and search products |
| POST | `/products` | createProduct | Bearer (admin) | Create a product |
| GET | `/products/{id}` | getProduct | Public | Get one product |
| PUT | `/products/{id}` | updateProduct | Bearer (admin) | Replace a product |
| GET | `/cart` | getCart | Bearer | Get the current user's cart |
| PUT | `/cart/items/{productId}` | setCartItem | Bearer | Add a product or set its quantity |
| DELETE | `/cart/items/{productId}` | removeCartItem | Bearer | Remove a product from the cart |
| GET | `/orders` | listOrders | Bearer | List the current user's orders |
| POST | `/orders` | placeOrder | Bearer | Check out (PlaceOrder) |
| GET | `/orders/{id}` | getOrder | Bearer | Get one of the current user's orders |
| POST | `/orders/{id}/pay` | payOrder | Bearer | Pay for an order (mock card payment) |

---

# Identity (P2)

## POST `/auth/register`

**Create a customer account** · `registerUser` · Public

**Request body** (required): [`RegisterRequest`](#registerrequest)

| Status | Description | Schema |
|---|---|---|
| 201 | Account created. Returns a token so the user is logged in immediately. | [`AuthResponse`](#authresponse) |
| 400 | Validation error (`VALIDATION_ERROR`) | [`Error`](#error) |
| 409 | Email already registered (`EMAIL_TAKEN`) | [`Error`](#error) |

Example 409:

```json
{
  "code": "EMAIL_TAKEN",
  "message": "An account with this email already exists."
}
```

## POST `/auth/login`

**Log in with email and password** · `loginUser` · Public

**Request body** (required): [`LoginRequest`](#loginrequest)

| Status | Description | Schema |
|---|---|---|
| 200 | Login successful. | [`AuthResponse`](#authresponse) |
| 400 | Validation error (`VALIDATION_ERROR`) | [`Error`](#error) |
| 401 | Wrong email or password. Use one message for both cases so accounts cannot be probed. | [`Error`](#error) |

Example 401:

```json
{
  "code": "UNAUTHORIZED",
  "message": "Invalid email or password."
}
```

## GET `/auth/me`

**Get the logged-in user** · `getCurrentUser` · Bearer

Called by the frontend on page load to restore the session from a stored token.

| Status | Description | Schema |
|---|---|---|
| 200 | The current user. | [`User`](#user) |
| 401 | Missing, invalid or expired token (`UNAUTHORIZED`) | [`Error`](#error) |

---

# Catalog (P3)

## GET `/products`

**List and search products** · `listProducts` · Public

**Query parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `q` | string (max 100) | No | Text search on product name and description. Example: `tomato` |
| `category` | [`Category`](#category) | No | Filter by category. |
| `page` | integer (min 1, default 1) | No | Page number, starting at 1. |
| `limit` | integer (1–50, default 12) | No | Items per page. |

| Status | Description | Schema |
|---|---|---|
| 200 | A page of products. | [`ProductPage`](#productpage) |
| 400 | Validation error (`VALIDATION_ERROR`) | [`Error`](#error) |

## POST `/products`

**Create a product (admin)** · `createProduct` · Bearer

**Request body** (required): [`ProductInput`](#productinput)

| Status | Description | Schema |
|---|---|---|
| 201 | Product created. | [`Product`](#product) |
| 400 | Validation error (`VALIDATION_ERROR`) | [`Error`](#error) |
| 401 | Missing, invalid or expired token (`UNAUTHORIZED`) | [`Error`](#error) |
| 403 | The user is logged in but is not an admin (`FORBIDDEN`) | [`Error`](#error) |
| 409 | A product with this SKU already exists (`SKU_TAKEN`) | [`Error`](#error) |

Example 409:

```json
{
  "code": "SKU_TAKEN",
  "message": "A product with SKU PRD-TOM-001 already exists."
}
```

## GET `/products/{id}`

**Get one product** · `getProduct` · Public

**Path parameters**

| Name | Type | Description |
|---|---|---|
| `id` | [`ObjectId`](#objectid) | Product ID. |

| Status | Description | Schema |
|---|---|---|
| 200 | The product. | [`Product`](#product) |
| 404 | The resource does not exist, or does not belong to the user (`NOT_FOUND`) | [`Error`](#error) |

## PUT `/products/{id}`

**Replace a product (admin)** · `updateProduct` · Bearer

Full replacement. Send every field of `ProductInput`.

**Path parameters**

| Name | Type | Description |
|---|---|---|
| `id` | [`ObjectId`](#objectid) | Product ID. |

**Request body** (required): [`ProductInput`](#productinput)

| Status | Description | Schema |
|---|---|---|
| 200 | Product updated. | [`Product`](#product) |
| 400 | Validation error (`VALIDATION_ERROR`) | [`Error`](#error) |
| 401 | Missing, invalid or expired token (`UNAUTHORIZED`) | [`Error`](#error) |
| 403 | Not an admin (`FORBIDDEN`) | [`Error`](#error) |
| 404 | Not found (`NOT_FOUND`) | [`Error`](#error) |

---

# Cart (P2)

## GET `/cart`

**Get the current user's cart** · `getCart` · Bearer

Returns an empty cart (no items, zero subtotal) if the user has not added anything yet.

| Status | Description | Schema |
|---|---|---|
| 200 | The cart, with current product names, images and prices. | [`Cart`](#cart-1) |
| 401 | Missing, invalid or expired token (`UNAUTHORIZED`) | [`Error`](#error) |

## PUT `/cart/items/{productId}`

**Add a product or set its quantity** · `setCartItem` · Bearer

Sets the **absolute** quantity of a product in the cart (adds it if missing). The quantity must be a multiple of the product's `quantityStep` and must not exceed its `stock`.

**Path parameters**

| Name | Type | Description |
|---|---|---|
| `productId` | [`ObjectId`](#objectid) | ID of the product in the catalog. |

**Request body** (required): [`SetCartItemRequest`](#setcartitemrequest)

| Status | Description | Schema |
|---|---|---|
| 200 | The updated cart. | [`Cart`](#cart-1) |
| 400 | Validation error (`VALIDATION_ERROR`) | [`Error`](#error) |
| 401 | Missing, invalid or expired token (`UNAUTHORIZED`) | [`Error`](#error) |
| 404 | Not found (`NOT_FOUND`) | [`Error`](#error) |
| 409 | Requested quantity is more than the available stock (`INSUFFICIENT_STOCK`) | [`Error`](#error) |

## DELETE `/cart/items/{productId}`

**Remove a product from the cart** · `removeCartItem` · Bearer

Idempotent. Returns the updated cart even if the product was not in it.

**Path parameters**

| Name | Type | Description |
|---|---|---|
| `productId` | [`ObjectId`](#objectid) | ID of the product in the catalog. |

| Status | Description | Schema |
|---|---|---|
| 200 | The updated cart. | [`Cart`](#cart-1) |
| 401 | Missing, invalid or expired token (`UNAUTHORIZED`) | [`Error`](#error) |

---

# Ordering (P4)

## GET `/orders`

**List the current user's orders (newest first)** · `listOrders` · Bearer

**Query parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `page` | integer (min 1, default 1) | No | Page number, starting at 1. |
| `limit` | integer (1–50, default 12) | No | Items per page. |

| Status | Description | Schema |
|---|---|---|
| 200 | A page of orders. | [`OrderPage`](#orderpage) |
| 401 | Missing, invalid or expired token (`UNAUTHORIZED`) | [`Error`](#error) |

## POST `/orders`

**Check out (PlaceOrder)** · `placeOrder` · Bearer

Turns the current cart into an order with status `PENDING`.

In one use case the server: loads the cart, checks stock, snapshots current prices into the order lines, decrements stock, creates the order and empties the cart.

Stock is held by unpaid orders (no automatic release in the MVP). A declined payment can be retried on the same order.

**Request body** (required): [`CheckoutRequest`](#checkoutrequest)

| Status | Description | Schema |
|---|---|---|
| 201 | Order placed, awaiting payment. | [`Order`](#order) |
| 400 | Validation error (`VALIDATION_ERROR`) | [`Error`](#error) |
| 401 | Missing, invalid or expired token (`UNAUTHORIZED`) | [`Error`](#error) |
| 409 | The cart is empty (`EMPTY_CART`) or an item is out of stock (`INSUFFICIENT_STOCK`) | [`Error`](#error) |

Example 409 (empty cart):

```json
{
  "code": "EMPTY_CART",
  "message": "Your cart is empty."
}
```

Example 409 (out of stock):

```json
{
  "code": "INSUFFICIENT_STOCK",
  "message": "Only 2.5 kg of Fresh Tomatoes is available."
}
```

## GET `/orders/{id}`

**Get one of the current user's orders** · `getOrder` · Bearer

Returns 404 (not 403) for another user's order so order IDs cannot be probed.

**Path parameters**

| Name | Type | Description |
|---|---|---|
| `id` | [`ObjectId`](#objectid) | Order ID. |

| Status | Description | Schema |
|---|---|---|
| 200 | The order. | [`Order`](#order) |
| 401 | Missing, invalid or expired token (`UNAUTHORIZED`) | [`Error`](#error) |
| 404 | Not found (`NOT_FOUND`) | [`Error`](#error) |

## POST `/orders/{id}/pay`

**Pay for an order (mock card payment)** · `payOrder` · Bearer

Mock gateway. **Card data is never stored or logged.**

- Any valid-format card succeeds, except card numbers ending in `0000`, which are declined (`PAYMENT_DECLINED`).
- Success moves the order `PENDING -> PAID`.
- A declined payment leaves the order `PENDING`, so the user can try again.

**Path parameters**

| Name | Type | Description |
|---|---|---|
| `id` | [`ObjectId`](#objectid) | Order ID. |

**Request body** (required): [`PaymentRequest`](#paymentrequest)

| Status | Description | Schema |
|---|---|---|
| 200 | Payment accepted. The order is now `PAID`. | [`Order`](#order) |
| 400 | Validation error (`VALIDATION_ERROR`) | [`Error`](#error) |
| 401 | Missing, invalid or expired token (`UNAUTHORIZED`) | [`Error`](#error) |
| 402 | Card declined (`PAYMENT_DECLINED`). The order stays `PENDING`. | [`Error`](#error) |
| 404 | Not found (`NOT_FOUND`) | [`Error`](#error) |
| 409 | The order is not `PENDING` (`INVALID_STATUS_TRANSITION`), for example already paid. | [`Error`](#error) |

Example 402:

```json
{
  "code": "PAYMENT_DECLINED",
  "message": "Your card was declined. Please try another card."
}
```

Example 409:

```json
{
  "code": "INVALID_STATUS_TRANSITION",
  "message": "Only PENDING orders can be paid."
}
```

---

# Common responses

| Name | Status | Code | Description | Example message |
|---|---|---|---|---|
| ValidationError | 400 | `VALIDATION_ERROR` | The request body or parameters are invalid. | Request validation failed. |
| Unauthorized | 401 | `UNAUTHORIZED` | Missing, invalid or expired token. | Authentication required. |
| Forbidden | 403 | `FORBIDDEN` | The user is logged in but is not an admin. | Admin access required. |
| NotFound | 404 | `NOT_FOUND` | The resource does not exist, or does not belong to the user. | Resource not found. |
| InsufficientStock | 409 | `INSUFFICIENT_STOCK` | Requested quantity is more than the available stock. | Only 2.5 kg of Fresh Tomatoes is available. |

Example `ValidationError` body:

```json
{
  "code": "VALIDATION_ERROR",
  "message": "Request validation failed.",
  "details": [
    { "field": "quantity", "message": "Must be a multiple of 0.5." }
  ]
}
```

---

# Schemas

## Shared

### ObjectId

`string`, pattern `^[a-f0-9]{24}$`. Example: `665f1c2e8a1b2c3d4e5f6a7b`

### Money

| Field | Type | Required | Description |
|---|---|---|---|
| `amount` | integer (min 0) | Yes | Amount in minor units (paisa). `25000` = Rs 250.00. |
| `currency` | string, enum `PKR` | Yes | Currency code. Example: `PKR` |

### Pagination

| Field | Type | Required | Description |
|---|---|---|---|
| `page` | integer | Yes | Current page. Example: `1` |
| `limit` | integer | Yes | Items per page. Example: `12` |
| `total` | integer | Yes | Total number of items across all pages. Example: `24` |
| `totalPages` | integer | Yes | Example: `2` |

### Error

| Field | Type | Required | Description |
|---|---|---|---|
| `code` | string (enum) | Yes | Machine-readable error code (see below). |
| `message` | string | Yes | Human-readable message safe to show to the user. |
| `details` | array of `{ field: string, message: string }` | No | Field-level problems, present for `VALIDATION_ERROR`. Each item requires `field` and `message`. |

**Error codes:** `VALIDATION_ERROR`, `UNAUTHORIZED`, `FORBIDDEN`, `NOT_FOUND`, `EMAIL_TAKEN`, `SKU_TAKEN`, `INSUFFICIENT_STOCK`, `EMPTY_CART`, `INVALID_STATUS_TRANSITION`, `PAYMENT_DECLINED`, `INTERNAL_ERROR`

## Identity

### Role

`string`, enum: `customer`, `admin`

### User

Never includes the password or its hash.

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | [`ObjectId`](#objectid) | Yes | |
| `name` | string | Yes | Example: `Aiman Khan` |
| `email` | string (email) | Yes | Example: `aiman@example.com` |
| `role` | [`Role`](#role) | Yes | |

### RegisterRequest

| Field | Type | Required | Constraints / Example |
|---|---|---|---|
| `name` | string | Yes | 2–80 chars. `Aiman Khan` |
| `email` | string (email) | Yes | `aiman@example.com` |
| `password` | string (password) | Yes | 8–72 chars. `Str0ngPassw0rd` |

### LoginRequest

| Field | Type | Required | Example |
|---|---|---|---|
| `email` | string (email) | Yes | `aiman@example.com` |
| `password` | string (password) | Yes | `Str0ngPassw0rd` |

### AuthResponse

| Field | Type | Required | Description |
|---|---|---|---|
| `token` | string | Yes | JWT to send as `Authorization: Bearer <token>`. |
| `user` | [`User`](#user) | Yes | |

## Catalog

### Category

`string`, enum: `produce`, `dairy`, `bakery`, `pantry`, `beverages`

### Unit

How the product is sold and priced. `string`, enum: `kg`, `piece`, `dozen`, `litre`, `pack`

### ProductInput

| Field | Type | Required | Description |
|---|---|---|---|
| `sku` | string (3–32) | Yes | Unique stock-keeping unit. Example: `PRD-TOM-001` |
| `name` | string (2–120) | Yes | Example: `Fresh Tomatoes` |
| `description` | string (max 1000) | No | Example: `Vine-ripened tomatoes, washed and sorted.` |
| `category` | [`Category`](#category) | Yes | |
| `unit` | [`Unit`](#unit) | Yes | |
| `unitPrice` | [`Money`](#money) | Yes | |
| `quantityStep` | number (> 0) | Yes | Smallest increment a customer can add. `0.5` for kg items, `1` for pieces. |
| `stock` | number (min 0) | Yes | Available quantity in the product's unit. Never negative. |
| `imageUrl` | string (uri) | Yes | Example: `https://example.com/images/tomatoes.jpg` |

### Product

[`ProductInput`](#productinput) plus:

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | [`ObjectId`](#objectid) | Yes | |
| `createdAt` | string (date-time) | Yes | Example: `2026-10-08T09:30:00Z` |

### ProductPage

| Field | Type | Required |
|---|---|---|
| `items` | array of [`Product`](#product) | Yes |
| `pagination` | [`Pagination`](#pagination) | Yes |

## Cart

### SetCartItemRequest

| Field | Type | Required | Description |
|---|---|---|---|
| `quantity` | number (> 0) | Yes | Absolute quantity. A multiple of the product's `quantityStep`, at most its `stock`. Example: `1.5` |

### CartItem

Name, image and price are read from the catalog at request time.

| Field | Type | Required | Description |
|---|---|---|---|
| `productId` | [`ObjectId`](#objectid) | Yes | |
| `name` | string | Yes | Example: `Fresh Tomatoes` |
| `imageUrl` | string (uri) | Yes | |
| `unit` | [`Unit`](#unit) | Yes | |
| `unitPrice` | [`Money`](#money) | Yes | |
| `quantity` | number | Yes | Example: `1.5` |
| `lineTotal` | [`Money`](#money) | Yes | |

### Cart

| Field | Type | Required | Description |
|---|---|---|---|
| `items` | array of [`CartItem`](#cartitem) | Yes | |
| `subtotal` | [`Money`](#money) | Yes | |
| `itemCount` | integer | Yes | Number of distinct products in the cart (for the navbar badge). |

## Ordering

### Address

| Field | Type | Required | Constraints / Example |
|---|---|---|---|
| `fullName` | string | Yes | Max 80. `Aiman Khan` |
| `phone` | string | Yes | Pattern `^\+?[0-9]{10,13}$`. `+923001234567` |
| `line1` | string | Yes | Max 120. `House 12, Street 4, F-7` |
| `line2` | string, nullable | No | Max 120. |
| `city` | string | Yes | Max 60. `Islamabad` |
| `postalCode` | string, nullable | No | Max 10. `44000` |

### DeliverySlot

| Field | Type | Required | Description |
|---|---|---|---|
| `date` | string (date) | Yes | Delivery date. Must be today or later. Example: `2026-10-09` |
| `window` | string, enum `MORNING` / `AFTERNOON` / `EVENING` | Yes | MORNING 9am–12pm, AFTERNOON 12pm–4pm, EVENING 5pm–8pm. Example: `EVENING` |

### CheckoutRequest

| Field | Type | Required |
|---|---|---|
| `deliveryAddress` | [`Address`](#address) | Yes |
| `deliverySlot` | [`DeliverySlot`](#deliveryslot) | Yes |

### PaymentRequest

Mock payment. The server must never store or log these fields.

| Field | Type | Required | Description |
|---|---|---|---|
| `cardholderName` | string | Yes | Example: `Aiman Khan` |
| `cardNumber` | string | Yes | Pattern `^[0-9]{16}$`. 16 digits, no spaces. Numbers ending in `0000` are declined. Example: `4242424242424242` |
| `expiryMonth` | integer (1–12) | Yes | Example: `12` |
| `expiryYear` | integer (min 2000) | Yes | Four-digit year. Must not be in the past. Example: `2030` |
| `cvc` | string | Yes | Pattern `^[0-9]{3,4}$`. Example: `123` |

### OrderStatus

`string`, enum: `PENDING`, `PAID`, `SHIPPED`, `CANCELLED`

Allowed transitions: `PENDING -> PAID -> SHIPPED`, or `PENDING -> CANCELLED`. The MVP implements `PENDING -> PAID` only.

### OrderLine

Immutable snapshot taken when the order was placed. Later catalog changes do not affect it.

| Field | Type | Required | Description |
|---|---|---|---|
| `productId` | [`ObjectId`](#objectid) | Yes | |
| `name` | string | Yes | Example: `Fresh Tomatoes` |
| `unit` | [`Unit`](#unit) | Yes | |
| `unitPrice` | [`Money`](#money) | Yes | |
| `quantity` | number | Yes | Example: `1.5` |
| `lineTotal` | [`Money`](#money) | Yes | |

### Order

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | [`ObjectId`](#objectid) | Yes | |
| `status` | [`OrderStatus`](#orderstatus) | Yes | |
| `lines` | array of [`OrderLine`](#orderline) | Yes | |
| `subtotal` | [`Money`](#money) | Yes | |
| `deliveryFee` | [`Money`](#money) | Yes | |
| `total` | [`Money`](#money) | Yes | `subtotal + deliveryFee`. |
| `deliveryAddress` | [`Address`](#address) | Yes | |
| `deliverySlot` | [`DeliverySlot`](#deliveryslot) | Yes | |
| `createdAt` | string (date-time) | Yes | Example: `2026-10-08T10:15:00Z` |
| `paidAt` | string (date-time), nullable | Yes | Null until the order is paid. |

### OrderPage

| Field | Type | Required |
|---|---|---|
| `items` | array of [`Order`](#order) | Yes |
| `pagination` | [`Pagination`](#pagination) | Yes |
