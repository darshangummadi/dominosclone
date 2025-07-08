```markdown
# Domino's Clone: A 7-Microservice Backend Architecture

This document outlines a potential backend architecture for a Domino's clone, built with 7 microservices. This design promotes separation of concerns, allowing your team of 8 to develop, deploy, and scale different parts of the application independently.

## Overall Architecture Overview

The frontend application (web or mobile) will not call each microservice directly. Instead, it will communicate with a single **API Gateway**. The gateway is responsible for routing requests to the appropriate internal microservice, handling authentication, and aggregating responses. This simplifies the frontend code and adds a layer of security.

```

[Frontend App] \<--\> [API Gateway] \<--\> [Microservices]

```

---

## Communication Strategy

**Synchronous (Direct API Calls):** For requests that need an immediate response (e.g., "Get me the menu"), services will call each other's REST APIs directly.

**Asynchronous (Message Queue):** For events that can be processed in the background (e.g., "An order was placed"), services will publish messages to a message queue (like RabbitMQ or Apache Kafka). Other services can then subscribe to these messages and react accordingly. This decouples services and improves resilience.

---

## Part 1: Java / Spring Boot Microservices (2)

These services handle the core business logic of user management and order orchestration.

### 1. User Service (Java / Spring Boot)

This service is the source of truth for all user-related information.

**Core Responsibilities:**
* User registration and login (password hashing).
* JWT (JSON Web Token) generation for authentication.
* Managing user profiles (name, phone number).
* Storing and managing customer addresses.
* Viewing order history (by fetching data from the Order Service).

**Example API Endpoints:**
```

POST /api/auth/register
POST /api/auth/login
GET /api/users/me
POST /api/users/me/addresses
GET /api/users/me/orders

```

**Primary Data Models:**
```

User { userId, name, email, passwordHash }
Address { addressId, userId, street, city, state, zipCode }

```

### 2. Order Service (Java / Spring Boot)

This service acts as the central orchestrator for placing and managing an order.

**Core Responsibilities:**
* Managing the user's shopping cart (add/remove/update items).
* Validating the cart's contents before checkout.
* Coordinating the checkout process:
    * Calls **Promotion Service** to validate coupons.
    * Calls **Payment Service** to process payment.
    * Calls **Store Locator Service** to assign the order to a local store.
* Persisting the final order details to its database.
* Publishing an `OrderPlaced` event to the message queue so the **Tracking Service** can take over.

**Example API Endpoints:**
```

GET /api/cart
POST /api/cart/items
POST /api/orders/checkout
GET /api/orders/{orderId}

```

**Primary Data Models:**
```

Order { orderId, userId, storeId, items, total, status, createdAt }
Cart { cartId, userId, items, subtotal }

```

---

## Part 2: C# / .NET Microservices (5)

These services handle more specialized, self-contained functionalities.

### 3. Menu Service (C# / .NET)

This service provides all information about the products available for purchase.

**Core Responsibilities:**
* Manages all product information: pizzas, crusts, toppings, sides, drinks.
* Provides pricing information.
* Handles categories and product details.
* Mostly a read-heavy service.

**Example API Endpoints:**
```

GET /api/menu/products
GET /api/menu/products/{productId}
GET /api/menu/categories
GET /api/menu/toppings

```

**Primary Data Models:**
```

Product { productId, name, description, price, category }
Topping { toppingId, name, price }
Category { categoryId, name }

```

### 4. Store Locator Service (C# / .NET)

This service handles all information related to physical store locations.

**Core Responsibilities:**
* Manages a list of all store locations, including addresses and hours.
* Determines which store is closest to a user's address for delivery.
* Manages store-specific menus or availability.

**Example API Endpoints:**
```

GET /api/stores
GET /api/stores/nearby?address={userAddress}

```

**Primary Data Models:**
```

Store { storeId, name, address, openingHours, deliveryRadius }

```

### 5. Payment Service (C# / .NET)

A dedicated, secure service for handling all financial transactions.

**Core Responsibilities:**
* Integrates with a third-party payment provider (e.g., Stripe, Braintree).
* Processes credit card payments.
* Handles payment authorizations and captures.
* Manages refunds.
* This service should be highly secure and isolated.

**Example API Endpoints:**
```

POST /api/payments/process
POST /api/payments/refund

```

**Primary Data Models:**
```

Transaction { transactionId, orderId, amount, status, providerId }

```

### 6. Promotion Service (C# / .NET)

Manages all discounts, deals, and coupon codes.

**Core Responsibilities:**
* Creates and manages coupon codes (e.g., "25OFF").
* Validates a coupon code against a shopping cart.
* Calculates the discount amount.
* Manages special deals (e.g., "2 Medium Pizzas for $20").

**Example API Endpoints:**
```

POST /api/promotions/validate
GET /api/promotions/deals

```

**Primary Data Models:**
```

Coupon { couponCode, discountType, value, expirationDate, rules }

```

### 7. Order Tracking Service (C# / .NET)

Provides the real-time "pizza tracker" functionality.

**Core Responsibilities:**
* Subscribes to the `OrderPlaced` event from the message queue.
* Manages the status of an order after it's been paid for (e.g., Prep -> Baking -> Quality Check -> Out for Delivery).
* Provides real-time updates to the frontend, likely via WebSockets.

**Example API Endpoints:**
```

GET /api/tracking/{orderId} (for initial status)
WebSocket endpoint for real-time updates: ws://[api.yourdomain.com/tracking](https://www.google.com/search?q=https://api.yourdomain.com/tracking)

```

**Primary Data Models:**
```

OrderTrackingStatus { orderId, status, estimatedTime, lastUpdated }

```
```
