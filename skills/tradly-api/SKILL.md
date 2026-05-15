---
name: tradly-api
description: Build marketplace and commerce applications using the Tradly headless API platform. Use this skill when creating new marketplace projects from scratch, integrating Tradly API into existing JavaScript applications, or building multi-vendor platforms, booking systems, storefronts, and commerce solutions. Focuses on direct API integration without SDK dependencies, providing practical patterns for authentication, product management, user flows, and payment processing.
license: Apache 2.0
---

# Tradly API Development Guide

## When To Use

- Build a new Tradly-powered marketplace, storefront, booking, or directory app.
- Integrate Tradly APIs into an existing JavaScript/TypeScript codebase.
- Implement direct API integration without depending on a Tradly SDK.

## When Not To Use

- You need a no-code setup only (use Tradly dashboard/admin configuration flows instead).
- You are building on a non-Tradly backend where replacing core commerce APIs is out of scope.
- You cannot securely manage API credentials/server-side routes.

## Implementation Workflow

When an agent uses this skill, it should:

1. Confirm required credentials and runtime env vars.
2. Implement a reusable API client first.
3. Build features in this order: auth -> catalog/layers -> cart/checkout -> orders.
4. Add robust error handling and response normalization.
5. Validate flows with runnable API examples before finalizing.

## Required Inputs

- `TRADLY_API_URL` (default: `https://api.tradly.app`)
- `TRADLY_TENANT_ID` (tenant/account identifier)
- `TRADLY_API_KEY` or `VITE_TRADLY_AUTH_KEY` depending on integration mode
- Optional: user auth token for user-scoped endpoints

## Recommended Deliverables

Deliverables from this skill should include:

- Env setup (`.env.example` and safe secret usage)
- A shared Tradly API client module
- Feature modules with clear responsibilities (auth, products, orders, layers)
- Error and retry strategy for network/auth/rate-limit failures
- Minimal test or verification steps for critical user journeys

## Quick Build Recipe

1. Setup environment variables and `.gitignore` secret rules.
2. Create `tradlyApi` client with base headers and centralized error handling.
3. Implement authentication and token persistence strategy.
4. Implement product/layers read flows with normalized response mapping.
5. Implement cart/checkout/order write flows with server-side protection.
6. Add lead capture flow (`/v1/users/register/{type}`) where needed.
7. Smoke test using cURL and one frontend flow end-to-end.

## Overview

Build marketplace and commerce applications using Tradly's headless API platform. This guide helps developers and product managers create production-ready applications with direct API integration.

**What is Tradly?**
Tradly is a headless, API-first marketplace and commerce platform that enables rapid development of:
- Multi-vendor marketplaces (C2C, B2C, B2B)
- Single-store storefronts
- Booking and appointment platforms
- Service marketplaces
- Directory and partner portals

**Core Philosophy:**
- Direct API calls (no SDK dependency)
- JavaScript-first approach (works with any framework)
- Production-ready code patterns
- Security and error handling built-in

---

# Quick Start

## Prerequisites Check

Before starting, ensure you have:
1. **Tradly Account** - Sign up at https://tradly.app
2. **API Access**
   - Production API: `https://api.tradly.app`
   - SuperAdmin Panel: `https://superadmin.tradly.app/`
3. **API Credentials**
   - Tenant ID (Account ID)
   - API Key (from SuperAdmin panel)

---

# Process

## Phase 1: Project Setup & Configuration

### 1.1 Environment Configuration

**CRITICAL: Secure API Key Management**

Always use environment variables for sensitive credentials:

```bash
# .env file (NEVER commit this)
TRADLY_API_URL=https://api.tradly.app
TRADLY_TENANT_ID=your_tenant_id
TRADLY_API_KEY=your_api_key
```

Add to `.gitignore`:
```
.env
.env.local
.env.*.local
```

For different frameworks:

**Node.js/Express:**
```bash
npm install dotenv
```

**Next.js:**
Built-in support - use `.env.local`

**React (Create React App):**
Use `REACT_APP_` prefix for env vars

**Vue/Vite:**
Use `VITE_` prefix for env vars

---

# Layers API (Blogs/Pages) + Markdown Handling

Use this pattern when your content is stored in Tradly `layers` and you need SEO-friendly routes like `/b/:slug` (blog) and `/p/:slug` (pages).

## Required Headers

Tradly deployments can differ. Use this safe baseline:

```js
const headers = {
  Accept: "application/json",
  "Content-Type": "application/json",
  "x-agent": "1",
  "x-auth-key": process.env.VITE_TRADLY_AUTH_KEY,
  ...(process.env.VITE_TRADLY_TOKEN
    ? { Authorization: `Bearer ${process.env.VITE_TRADLY_TOKEN}` }
    : {}),
  ...(process.env.VITE_TRADLY_TENANT_ID
    ? {
        "x-tenant-id": process.env.VITE_TRADLY_TENANT_ID,
        "x-tenant": process.env.VITE_TRADLY_TENANT_ID,
      }
    : {}),
};
```

## List Layers By Type

Always filter by `type`:

```js
// blog list
GET /v1/layers?type=blog&page=1&per_page=100

// pages list
GET /v1/layers?type=page&page=1&per_page=100
```

Example:

```js
async function fetchLayersByType(type, page = 1, perPage = 100) {
  const qs = new URLSearchParams({
    type,
    page: String(page),
    per_page: String(perPage),
  });
  const res = await fetch(`${API_BASE}/v1/layers?${qs}`, { headers });
  if (!res.ok) throw new Error(`Layers fetch failed: ${res.status}`);
  const payload = await res.json();
  return payload?.data?.layers || payload?.data || [];
}
```

## Detail Page Strategy (Slug First, ID Fallback)

For route params that are slugs, do not call only `/v1/layers/:id`.

```js
async function fetchLayerDetail(idOrSlug, expectedType) {
  // 1) Try by slug first
  const bySlug = await fetch(`${API_BASE}/v1/layers/by_slug/${encodeURIComponent(idOrSlug)}`, { headers });
  if (bySlug.ok) {
    const payload = await bySlug.json();
    const layer = payload?.data?.layer || payload?.data || payload;
    if (!expectedType || layer?.type === expectedType) return layer;
  }

  // 2) Fallback to direct layer endpoint (older links may use id)
  const byId = await fetch(`${API_BASE}/v1/layers/${encodeURIComponent(idOrSlug)}`, { headers });
  if (!byId.ok) throw new Error(`Layer fetch failed: ${byId.status}`);
  const payload = await byId.json();
  const layer = payload?.data || payload;
  if (expectedType && layer?.type !== expectedType) {
    throw new Error(`Type mismatch: expected ${expectedType}, got ${layer?.type || "unknown"}`);
  }
  return layer;
}
```

## Normalize Field Differences

Different endpoints may return slightly different keys. Normalize once:

```js
function normalizeLayer(raw = {}) {
  return {
    id: raw.id || raw.uid || raw._id || raw.uuid || "",
    slug: raw.slug || raw.handle || "",
    type: raw.type || "",
    title: raw.title || raw.name || raw.heading || "",
    description: raw.description || raw.short_description || raw.summary || raw.excerpt || "",
    content: raw.content || raw.html || raw.content_html || raw.body || "",
    createdAt: raw.created_at || raw.createdAt || raw.published_at || raw.publishedAt || "",
    featuredImage: raw.featured_image || raw.featuredImage || raw.cover_image || raw.coverImage || raw.image || "",
    author: raw.author || raw.created_by || raw.createdBy || "",
  };
}
```

## Markdown Listicles + HTML Fallback

Many teams store listicles in Markdown while others store full HTML. Support both:

```bash
npm install marked
```

```js
import { marked } from "marked";

function renderLayerContent(content = "") {
  const hasHtml = /<[^>]+>/.test(content);
  if (hasHtml) return content; // already HTML
  return marked.parse(content, { gfm: true, breaks: true });
}
```

React usage:

```jsx
const html = renderLayerContent(layer.content || "");
<div dangerouslySetInnerHTML={{ __html: html }} />
```

This ensures:
- Markdown bullets/numbered lists render correctly.
- Existing HTML posts keep working unchanged.

## Common Production Checks

1. If list works but detail fails, verify detail route is using `by_slug` first.
2. If you get `401 tenant`, add tenant headers (`x-tenant-id`, `x-tenant`) and/or token.
3. Ensure blog routes only query `type=blog`, pages routes only query `type=page`.
4. For SEO routes, generate static paths from `slug || id`.

---

# User Lead Collection API (`/v1/users/register/{type}`)

Use this endpoint when collecting lightweight leads without full account onboarding.

## Endpoint

```http
POST /v1/users/register/{type}
```

Where `{type}` is a collection bucket, for example:
- `newsletter`
- `waiting-list`
- `private-beta`

## Required Headers

```http
Content-Type: application/json
Authorization: Bearer <pk_key_or_auth_key>
```

Use env var for auth:

```bash
VITE_TRADLY_AUTH_KEY=pk_live_or_pk_test_key
```

## Request Body

```json
{
  "user": {
    "first_name": "Bod",
    "last_name": "Kl",
    "email": "bod@gmail.com"
  }
}
```

## cURL Examples

### 1) Newsletter signup
```bash
curl --location 'https://api.tradly.app/v1/users/register/newsletter' \
  --header 'Content-Type: application/json' \
  --header "Authorization: Bearer $VITE_TRADLY_AUTH_KEY" \
  --data-raw '{
    "user": {
      "first_name": "Bod",
      "last_name": "Kl",
      "email": "bod@gmail.com"
    }
  }'
```

### 2) Waiting list
```bash
curl --location 'https://api.tradly.app/v1/users/register/waiting-list' \
  --header 'Content-Type: application/json' \
  --header "Authorization: Bearer $VITE_TRADLY_AUTH_KEY" \
  --data-raw '{
    "user": {
      "first_name": "Ava",
      "last_name": "Lee",
      "email": "ava@example.com"
    }
  }'
```

### 3) Private beta
```bash
curl --location 'https://api.tradly.app/v1/users/register/private-beta' \
  --header 'Content-Type: application/json' \
  --header "Authorization: Bearer $VITE_TRADLY_AUTH_KEY" \
  --data-raw '{
    "user": {
      "first_name": "Sam",
      "last_name": "Park",
      "email": "sam@example.com"
    }
  }'
```

## JavaScript Helper

```js
async function registerUserType({ type, firstName, lastName = "", email, authKey }) {
  const safeType = String(type || "").trim();
  if (!safeType) throw new Error("type is required");

  const res = await fetch(`https://api.tradly.app/v1/users/register/${encodeURIComponent(safeType)}`, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      Authorization: `Bearer ${authKey}`,
    },
    body: JSON.stringify({
      user: {
        first_name: firstName,
        last_name: lastName,
        email,
      },
    }),
  });

  const payload = await res.json().catch(() => ({}));
  if (!res.ok) {
    throw new Error(payload?.error?.message || `Tradly register failed (${res.status})`);
  }
  return payload;
}
```

## Notes

- Keep lead collection calls server-side when possible (proxy via your API route).
- Normalize type names (lowercase + hyphen) and reuse exact strings across app/events.
- Handle duplicate emails gracefully; treat successful existing-user response as non-fatal for lead capture flows.

### 1.2 API Client Setup

Create a reusable API client module:

```javascript
// config/tradlyApi.js

class TradlyAPI {
  constructor() {
    this.baseURL = process.env.TRADLY_API_URL || 'https://api.tradly.app';
    this.tenantId = process.env.TRADLY_TENANT_ID;
    this.apiKey = process.env.TRADLY_API_KEY;
    
    if (!this.tenantId || !this.apiKey) {
      throw new Error('Missing Tradly API credentials');
    }
  }

  // Base request method with error handling
  async request(endpoint, options = {}) {
    const url = `${this.baseURL}${endpoint}`;
    
    const headers = {
      'Content-Type': 'application/json',
      'X-Tenant-Id': this.tenantId,
      'Authorization': `Bearer ${this.apiKey}`,
      ...options.headers,
    };

    try {
      const response = await fetch(url, {
        ...options,
        headers,
      });

      // Handle different response types
      const contentType = response.headers.get('content-type');
      const data = contentType?.includes('application/json')
        ? await response.json()
        : await response.text();

      if (!response.ok) {
        throw new Error(
          data?.message || 
          data?.error || 
          `API Error: ${response.status} ${response.statusText}`
        );
      }

      return data;
    } catch (error) {
      // Enhanced error handling
      if (error.name === 'TypeError' && error.message.includes('fetch')) {
        throw new Error('Network error: Unable to connect to Tradly API');
      }
      throw error;
    }
  }

  // Convenience methods
  get(endpoint, params = {}) {
    const queryString = new URLSearchParams(params).toString();
    const url = queryString ? `${endpoint}?${queryString}` : endpoint;
    return this.request(url, { method: 'GET' });
  }

  post(endpoint, body) {
    return this.request(endpoint, {
      method: 'POST',
      body: JSON.stringify(body),
    });
  }

  put(endpoint, body) {
    return this.request(endpoint, {
      method: 'PUT',
      body: JSON.stringify(body),
    });
  }

  delete(endpoint) {
    return this.request(endpoint, { method: 'DELETE' });
  }
}

// Export singleton instance
export const tradlyApi = new TradlyAPI();
```

### 1.3 Project Structure

**For New Marketplace Projects:**

```
marketplace-app/
├── src/
│   ├── api/
│   │   ├── tradlyApi.js         # API client
│   │   ├── auth.js              # Authentication methods
│   │   ├── products.js          # Product operations
│   │   ├── orders.js            # Order management
│   │   └── users.js             # User management
│   ├── components/
│   │   ├── ProductCard.js
│   │   ├── Cart.js
│   │   └── Checkout.js
│   ├── utils/
│   │   ├── errorHandler.js      # Centralized error handling
│   │   └── validators.js        # Input validation
│   └── App.js
├── .env.example                 # Template for env vars
├── .env                         # Actual credentials (gitignored)
└── .gitignore
```

**For Integrating into Existing Apps:**

```
existing-app/
├── src/
│   ├── integrations/
│   │   └── tradly/
│   │       ├── client.js        # Isolated API client
│   │       ├── services/
│   │       │   ├── products.js
│   │       │   └── orders.js
│   │       └── types.js         # Type definitions
│   └── ... (existing structure)
```

---

## Phase 2: Core Feature Implementation

### 2.1 Authentication & User Management

**User Registration:**

```javascript
// api/auth.js
import { tradlyApi } from './tradlyApi';

export const authService = {
  // Register new user
  async register(userData) {
    try {
      const response = await tradlyApi.post('/users/register', {
        email: userData.email,
        password: userData.password,
        first_name: userData.firstName,
        last_name: userData.lastName,
        type: userData.type || 'user', // 'user' or 'vendor'
      });
      
      // Store auth token
      if (response.token) {
        localStorage.setItem('tradly_auth_token', response.token);
        localStorage.setItem('tradly_user', JSON.stringify(response.user));
      }
      
      return response;
    } catch (error) {
      throw new Error(`Registration failed: ${error.message}`);
    }
  },

  // User login
  async login(email, password) {
    try {
      const response = await tradlyApi.post('/users/login', {
        email,
        password,
      });
      
      if (response.token) {
        localStorage.setItem('tradly_auth_token', response.token);
        localStorage.setItem('tradly_user', JSON.stringify(response.user));
      }
      
      return response;
    } catch (error) {
      throw new Error(`Login failed: ${error.message}`);
    }
  },

  // Get current user profile
  async getCurrentUser() {
    const token = localStorage.getItem('tradly_auth_token');
    if (!token) {
      throw new Error('No authentication token found');
    }

    return tradlyApi.get('/users/me', {
      headers: {
        'Authorization': `Bearer ${token}`,
      },
    });
  },

  // Logout
  logout() {
    localStorage.removeItem('tradly_auth_token');
    localStorage.removeItem('tradly_user');
  },

  // Check if user is authenticated
  isAuthenticated() {
    return !!localStorage.getItem('tradly_auth_token');
  },
};
```

**Protected Route Pattern:**

```javascript
// utils/authGuard.js
import { authService } from '../api/auth';

export const requireAuth = async () => {
  if (!authService.isAuthenticated()) {
    throw new Error('Authentication required');
  }

  try {
    const user = await authService.getCurrentUser();
    return user;
  } catch (error) {
    authService.logout();
    throw new Error('Session expired. Please login again.');
  }
};
```

### 2.2 Product Management

**Listing Products:**

```javascript
// api/products.js
import { tradlyApi } from './tradlyApi';

export const productService = {
  // Get all products with filters
  async getProducts(options = {}) {
    const {
      page = 1,
      perPage = 20,
      categoryId = null,
      search = '',
      sortBy = 'created_at',
      sortOrder = 'desc',
    } = options;

    const params = {
      page,
      per_page: perPage,
      sort_by: sortBy,
      sort_order: sortOrder,
    };

    if (categoryId) params.category_id = categoryId;
    if (search) params.search = search;

    try {
      const response = await tradlyApi.get('/listings', params);
      
      return {
        products: response.listings || [],
        pagination: {
          currentPage: response.page || 1,
          totalPages: response.total_pages || 1,
          totalItems: response.total_records || 0,
          perPage: response.per_page || 20,
        },
      };
    } catch (error) {
      throw new Error(`Failed to fetch products: ${error.message}`);
    }
  },

  // Get single product details
  async getProduct(productId) {
    if (!productId) {
      throw new Error('Product ID is required');
    }

    try {
      const response = await tradlyApi.get(`/listings/${productId}`);
      return response.listing || response;
    } catch (error) {
      throw new Error(`Failed to fetch product: ${error.message}`);
    }
  },

  // Create new product (vendor only)
  async createProduct(productData) {
    const token = localStorage.getItem('tradly_auth_token');
    if (!token) {
      throw new Error('Authentication required to create products');
    }

    const payload = {
      title: productData.title,
      description: productData.description,
      list_price: productData.price,
      sale_price: productData.salePrice || productData.price,
      category_id: productData.categoryId,
      stock: productData.stock || 1,
      images: productData.images || [],
      attributes: productData.attributes || {},
    };

    try {
      const response = await tradlyApi.post('/listings', payload, {
        headers: {
          'Authorization': `Bearer ${token}`,
        },
      });
      
      return response.listing || response;
    } catch (error) {
      throw new Error(`Failed to create product: ${error.message}`);
    }
  },

  // Update product
  async updateProduct(productId, updates) {
    const token = localStorage.getItem('tradly_auth_token');
    if (!token) {
      throw new Error('Authentication required');
    }

    try {
      const response = await tradlyApi.put(`/listings/${productId}`, updates, {
        headers: {
          'Authorization': `Bearer ${token}`,
        },
      });
      
      return response.listing || response;
    } catch (error) {
      throw new Error(`Failed to update product: ${error.message}`);
    }
  },

  // Delete product
  async deleteProduct(productId) {
    const token = localStorage.getItem('tradly_auth_token');
    if (!token) {
      throw new Error('Authentication required');
    }

    try {
      await tradlyApi.delete(`/listings/${productId}`, {
        headers: {
          'Authorization': `Bearer ${token}`,
        },
      });
      
      return { success: true };
    } catch (error) {
      throw new Error(`Failed to delete product: ${error.message}`);
    }
  },
};
```

### 2.3 Category Management

```javascript
// api/categories.js
import { tradlyApi } from './tradlyApi';

export const categoryService = {
  // Get all categories
  async getCategories(options = {}) {
    const {
      type = 'listings', // 'listings' or 'accounts'
      parentId = null,
    } = options;

    const params = { type };
    if (parentId) params.parent_id = parentId;

    try {
      const response = await tradlyApi.get('/categories', params);
      return response.categories || [];
    } catch (error) {
      throw new Error(`Failed to fetch categories: ${error.message}`);
    }
  },

  // Get category tree (hierarchical)
  async getCategoryTree() {
    try {
      const categories = await this.getCategories();
      
      // Build hierarchical structure
      const categoryMap = {};
      const rootCategories = [];

      categories.forEach(cat => {
        categoryMap[cat.id] = { ...cat, children: [] };
      });

      categories.forEach(cat => {
        if (cat.parent_id && categoryMap[cat.parent_id]) {
          categoryMap[cat.parent_id].children.push(categoryMap[cat.id]);
        } else {
          rootCategories.push(categoryMap[cat.id]);
        }
      });

      return rootCategories;
    } catch (error) {
      throw new Error(`Failed to build category tree: ${error.message}`);
    }
  },
};
```

### 2.4 Shopping Cart & Checkout

**Cart Management (Client-Side):**

```javascript
// api/cart.js

export const cartService = {
  // Get cart from localStorage
  getCart() {
    const cart = localStorage.getItem('tradly_cart');
    return cart ? JSON.parse(cart) : { items: [], total: 0 };
  },

  // Add item to cart
  addItem(product, quantity = 1, variants = {}) {
    const cart = this.getCart();
    
    const existingItem = cart.items.find(
      item => item.id === product.id && 
      JSON.stringify(item.variants) === JSON.stringify(variants)
    );

    if (existingItem) {
      existingItem.quantity += quantity;
    } else {
      cart.items.push({
        id: product.id,
        title: product.title,
        price: product.sale_price || product.list_price,
        image: product.images?.[0] || null,
        quantity,
        variants,
      });
    }

    this.saveCart(cart);
    return cart;
  },

  // Update item quantity
  updateQuantity(productId, quantity) {
    const cart = this.getCart();
    const item = cart.items.find(i => i.id === productId);
    
    if (item) {
      if (quantity <= 0) {
        this.removeItem(productId);
      } else {
        item.quantity = quantity;
        this.saveCart(cart);
      }
    }
    
    return cart;
  },

  // Remove item
  removeItem(productId) {
    const cart = this.getCart();
    cart.items = cart.items.filter(i => i.id !== productId);
    this.saveCart(cart);
    return cart;
  },

  // Clear cart
  clearCart() {
    localStorage.removeItem('tradly_cart');
    return { items: [], total: 0 };
  },

  // Calculate totals
  calculateTotals(cart = null) {
    const currentCart = cart || this.getCart();
    
    const subtotal = currentCart.items.reduce(
      (sum, item) => sum + (item.price * item.quantity),
      0
    );

    return {
      subtotal,
      tax: subtotal * 0.1, // Example: 10% tax
      shipping: subtotal > 50 ? 0 : 5.99, // Free shipping over $50
      total: subtotal + (subtotal * 0.1) + (subtotal > 50 ? 0 : 5.99),
    };
  },

  // Save cart to localStorage
  saveCart(cart) {
    localStorage.setItem('tradly_cart', JSON.stringify(cart));
    
    // Dispatch event for cart updates
    window.dispatchEvent(new CustomEvent('cartUpdated', { 
      detail: cart 
    }));
  },

  // Get item count
  getItemCount() {
    const cart = this.getCart();
    return cart.items.reduce((sum, item) => sum + item.quantity, 0);
  },
};
```

**Order Creation:**

```javascript
// api/orders.js
import { tradlyApi } from './tradlyApi';
import { cartService } from './cart';

export const orderService = {
  // Create order from cart
  async createOrder(shippingDetails, paymentMethod = 'stripe') {
    const token = localStorage.getItem('tradly_auth_token');
    if (!token) {
      throw new Error('Authentication required to place order');
    }

    const cart = cartService.getCart();
    if (!cart.items || cart.items.length === 0) {
      throw new Error('Cart is empty');
    }

    const totals = cartService.calculateTotals(cart);

    const orderData = {
      items: cart.items.map(item => ({
        listing_id: item.id,
        quantity: item.quantity,
        variant_id: item.variants?.id || null,
      })),
      shipping_address: {
        address_line1: shippingDetails.address,
        city: shippingDetails.city,
        state: shippingDetails.state,
        postal_code: shippingDetails.postalCode,
        country: shippingDetails.country,
      },
      payment_method: paymentMethod,
      total_amount: totals.total,
    };

    try {
      const response = await tradlyApi.post('/orders', orderData, {
        headers: {
          'Authorization': `Bearer ${token}`,
        },
      });

      // Clear cart after successful order
      if (response.order || response.id) {
        cartService.clearCart();
      }

      return response.order || response;
    } catch (error) {
      throw new Error(`Failed to create order: ${error.message}`);
    }
  },

  // Get user's orders
  async getOrders(options = {}) {
    const token = localStorage.getItem('tradly_auth_token');
    if (!token) {
      throw new Error('Authentication required');
    }

    const {
      page = 1,
      perPage = 10,
      status = null,
    } = options;

    const params = { page, per_page: perPage };
    if (status) params.status = status;

    try {
      const response = await tradlyApi.get('/orders', params, {
        headers: {
          'Authorization': `Bearer ${token}`,
        },
      });

      return {
        orders: response.orders || [],
        pagination: {
          currentPage: response.page || 1,
          totalPages: response.total_pages || 1,
          totalItems: response.total_records || 0,
        },
      };
    } catch (error) {
      throw new Error(`Failed to fetch orders: ${error.message}`);
    }
  },

  // Get single order details
  async getOrder(orderId) {
    const token = localStorage.getItem('tradly_auth_token');
    if (!token) {
      throw new Error('Authentication required');
    }

    try {
      const response = await tradlyApi.get(`/orders/${orderId}`, {
        headers: {
          'Authorization': `Bearer ${token}`,
        },
      });

      return response.order || response;
    } catch (error) {
      throw new Error(`Failed to fetch order: ${error.message}`);
    }
  },

  // Update order status (vendor only)
  async updateOrderStatus(orderId, status) {
    const token = localStorage.getItem('tradly_auth_token');
    if (!token) {
      throw new Error('Authentication required');
    }

    // Valid statuses: pending, confirmed, shipped, delivered, cancelled
    const validStatuses = ['pending', 'confirmed', 'shipped', 'delivered', 'cancelled'];
    if (!validStatuses.includes(status)) {
      throw new Error(`Invalid status. Must be one of: ${validStatuses.join(', ')}`);
    }

    try {
      const response = await tradlyApi.put(
        `/orders/${orderId}`,
        { status },
        {
          headers: {
            'Authorization': `Bearer ${token}`,
          },
        }
      );

      return response.order || response;
    } catch (error) {
      throw new Error(`Failed to update order status: ${error.message}`);
    }
  },
};
```

### 2.5 Search Implementation

```javascript
// api/search.js
import { tradlyApi } from './tradlyApi';

export const searchService = {
  // Search listings
  async searchProducts(query, options = {}) {
    const {
      page = 1,
      perPage = 20,
      categoryId = null,
      minPrice = null,
      maxPrice = null,
      sortBy = 'relevance',
    } = options;

    const params = {
      search: query,
      page,
      per_page: perPage,
      sort_by: sortBy,
    };

    if (categoryId) params.category_id = categoryId;
    if (minPrice !== null) params.min_price = minPrice;
    if (maxPrice !== null) params.max_price = maxPrice;

    try {
      const response = await tradlyApi.get('/listings/search', params);
      
      return {
        results: response.listings || [],
        pagination: {
          currentPage: response.page || 1,
          totalPages: response.total_pages || 1,
          totalItems: response.total_records || 0,
        },
        facets: response.facets || {}, // Category counts, price ranges, etc.
      };
    } catch (error) {
      throw new Error(`Search failed: ${error.message}`);
    }
  },

  // Auto-suggest / autocomplete
  async suggest(query, limit = 5) {
    if (!query || query.length < 2) {
      return [];
    }

    try {
      const response = await tradlyApi.get('/listings/suggest', {
        q: query,
        limit,
      });

      return response.suggestions || [];
    } catch (error) {
      console.error('Autocomplete failed:', error);
      return [];
    }
  },
};
```

### 2.6 Reviews & Ratings

```javascript
// api/reviews.js
import { tradlyApi } from './tradlyApi';

export const reviewService = {
  // Get reviews for a product
  async getReviews(listingId, options = {}) {
    const { page = 1, perPage = 10 } = options;

    try {
      const response = await tradlyApi.get(`/listings/${listingId}/reviews`, {
        page,
        per_page: perPage,
      });

      return {
        reviews: response.reviews || [],
        averageRating: response.average_rating || 0,
        totalReviews: response.total_reviews || 0,
        pagination: {
          currentPage: response.page || 1,
          totalPages: response.total_pages || 1,
        },
      };
    } catch (error) {
      throw new Error(`Failed to fetch reviews: ${error.message}`);
    }
  },

  // Submit a review
  async createReview(listingId, reviewData) {
    const token = localStorage.getItem('tradly_auth_token');
    if (!token) {
      throw new Error('Authentication required to submit reviews');
    }

    const payload = {
      rating: reviewData.rating, // 1-5
      title: reviewData.title,
      comment: reviewData.comment,
    };

    try {
      const response = await tradlyApi.post(
        `/listings/${listingId}/reviews`,
        payload,
        {
          headers: {
            'Authorization': `Bearer ${token}`,
          },
        }
      );

      return response.review || response;
    } catch (error) {
      throw new Error(`Failed to submit review: ${error.message}`);
    }
  },
};
```

---

## Phase 3: Advanced Features

### 3.1 Image Upload

```javascript
// api/upload.js
import { tradlyApi } from './tradlyApi';

export const uploadService = {
  // Upload image
  async uploadImage(file) {
    const token = localStorage.getItem('tradly_auth_token');
    if (!token) {
      throw new Error('Authentication required to upload images');
    }

    // Validate file
    const maxSize = 5 * 1024 * 1024; // 5MB
    const allowedTypes = ['image/jpeg', 'image/png', 'image/webp', 'image/gif'];

    if (file.size > maxSize) {
      throw new Error('File size must be less than 5MB');
    }

    if (!allowedTypes.includes(file.type)) {
      throw new Error('Only JPEG, PNG, WebP, and GIF images are allowed');
    }

    const formData = new FormData();
    formData.append('image', file);

    try {
      const response = await fetch(`${tradlyApi.baseURL}/uploads`, {
        method: 'POST',
        headers: {
          'Authorization': `Bearer ${token}`,
          'X-Tenant-Id': tradlyApi.tenantId,
        },
        body: formData,
      });

      if (!response.ok) {
        throw new Error(`Upload failed: ${response.statusText}`);
      }

      const data = await response.json();
      return data.url || data.image_url;
    } catch (error) {
      throw new Error(`Image upload failed: ${error.message}`);
    }
  },

  // Upload multiple images
  async uploadImages(files) {
    const uploadPromises = Array.from(files).map(file => 
      this.uploadImage(file)
    );

    try {
      const urls = await Promise.all(uploadPromises);
      return urls;
    } catch (error) {
      throw new Error(`Multiple upload failed: ${error.message}`);
    }
  },
};
```

### 3.2 Webhook Integration

```javascript
// api/webhooks.js (Server-side - Node.js/Express)
import crypto from 'crypto';

export const webhookService = {
  // Verify webhook signature
  verifySignature(payload, signature, secret) {
    const expectedSignature = crypto
      .createHmac('sha256', secret)
      .update(JSON.stringify(payload))
      .digest('hex');

    return signature === expectedSignature;
  },

  // Handle webhook events
  handleWebhook(req, res) {
    const signature = req.headers['x-tradly-signature'];
    const webhookSecret = process.env.TRADLY_WEBHOOK_SECRET;

    if (!this.verifySignature(req.body, signature, webhookSecret)) {
      return res.status(401).json({ error: 'Invalid signature' });
    }

    const { event, data } = req.body;

    // Handle different event types
    switch (event) {
      case 'order.created':
        // Handle new order
        console.log('New order:', data.order);
        break;
      
      case 'order.updated':
        // Handle order update
        console.log('Order updated:', data.order);
        break;
      
      case 'payment.completed':
        // Handle payment completion
        console.log('Payment completed:', data.payment);
        break;
      
      default:
        console.log('Unknown event:', event);
    }

    res.status(200).json({ received: true });
  },
};
```

### 3.3 Payment Integration

```javascript
// api/payments.js
import { tradlyApi } from './tradlyApi';

export const paymentService = {
  // Initialize payment (Stripe example)
  async initializePayment(orderId, amount) {
    const token = localStorage.getItem('tradly_auth_token');
    if (!token) {
      throw new Error('Authentication required');
    }

    try {
      const response = await tradlyApi.post(
        '/payments/initialize',
        {
          order_id: orderId,
          amount: amount,
          currency: 'USD',
          payment_method: 'stripe',
        },
        {
          headers: {
            'Authorization': `Bearer ${token}`,
          },
        }
      );

      return {
        clientSecret: response.client_secret,
        paymentIntentId: response.payment_intent_id,
      };
    } catch (error) {
      throw new Error(`Payment initialization failed: ${error.message}`);
    }
  },

  // Confirm payment
  async confirmPayment(paymentIntentId) {
    const token = localStorage.getItem('tradly_auth_token');
    if (!token) {
      throw new Error('Authentication required');
    }

    try {
      const response = await tradlyApi.post(
        '/payments/confirm',
        { payment_intent_id: paymentIntentId },
        {
          headers: {
            'Authorization': `Bearer ${token}`,
          },
        }
      );

      return response;
    } catch (error) {
      throw new Error(`Payment confirmation failed: ${error.message}`);
    }
  },
};
```

---

## Phase 4: Error Handling & Best Practices

### 4.1 Centralized Error Handling

```javascript
// utils/errorHandler.js

export class TradlyError extends Error {
  constructor(message, statusCode, originalError = null) {
    super(message);
    this.name = 'TradlyError';
    this.statusCode = statusCode;
    this.originalError = originalError;
  }
}

export const errorHandler = {
  // Parse API errors
  parseError(error) {
    if (error instanceof TradlyError) {
      return error;
    }

    // Network errors
    if (error.name === 'TypeError' && error.message.includes('fetch')) {
      return new TradlyError(
        'Unable to connect to Tradly API. Please check your internet connection.',
        0,
        error
      );
    }

    // API errors with status codes
    if (error.response) {
      const statusCode = error.response.status;
      const message = error.response.data?.message || error.message;

      return new TradlyError(message, statusCode, error);
    }

    // Generic errors
    return new TradlyError(
      error.message || 'An unexpected error occurred',
      500,
      error
    );
  },

  // User-friendly error messages
  getUserMessage(error) {
    const tradlyError = this.parseError(error);

    switch (tradlyError.statusCode) {
      case 400:
        return 'Invalid request. Please check your input.';
      case 401:
        return 'You need to log in to perform this action.';
      case 403:
        return 'You do not have permission to perform this action.';
      case 404:
        return 'The requested resource was not found.';
      case 409:
        return 'This resource already exists.';
      case 429:
        return 'Too many requests. Please try again later.';
      case 500:
      case 502:
      case 503:
        return 'Server error. Please try again later.';
      default:
        return tradlyError.message;
    }
  },

  // Log errors (in production, send to logging service)
  logError(error, context = {}) {
    const tradlyError = this.parseError(error);

    console.error('Tradly Error:', {
      message: tradlyError.message,
      statusCode: tradlyError.statusCode,
      context,
      timestamp: new Date().toISOString(),
      stack: tradlyError.originalError?.stack,
    });

    // In production, send to error tracking service
    // Example: Sentry.captureException(tradlyError);
  },
};
```

### 4.2 Input Validation

```javascript
// utils/validators.js

export const validators = {
  // Email validation
  isValidEmail(email) {
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    return emailRegex.test(email);
  },

  // Password strength
  isStrongPassword(password) {
    // At least 8 characters, 1 uppercase, 1 lowercase, 1 number
    const passwordRegex = /^(?=.*[a-z])(?=.*[A-Z])(?=.*\d).{8,}$/;
    return passwordRegex.test(password);
  },

  // Price validation
  isValidPrice(price) {
    const numPrice = parseFloat(price);
    return !isNaN(numPrice) && numPrice >= 0;
  },

  // Required fields
  validateRequired(fields, data) {
    const errors = {};

    fields.forEach(field => {
      if (!data[field] || data[field].toString().trim() === '') {
        errors[field] = `${field} is required`;
      }
    });

    return {
      isValid: Object.keys(errors).length === 0,
      errors,
    };
  },

  // Product validation
  validateProduct(product) {
    const errors = {};

    if (!product.title || product.title.trim().length < 3) {
      errors.title = 'Title must be at least 3 characters';
    }

    if (!product.description || product.description.trim().length < 10) {
      errors.description = 'Description must be at least 10 characters';
    }

    if (!this.isValidPrice(product.price)) {
      errors.price = 'Invalid price';
    }

    if (!product.categoryId) {
      errors.categoryId = 'Category is required';
    }

    return {
      isValid: Object.keys(errors).length === 0,
      errors,
    };
  },
};
```

### 4.3 Rate Limiting & Caching

```javascript
// utils/rateLimit.js

class RateLimiter {
  constructor(maxRequests = 100, windowMs = 60000) {
    this.maxRequests = maxRequests;
    this.windowMs = windowMs;
    this.requests = [];
  }

  canMakeRequest() {
    const now = Date.now();
    this.requests = this.requests.filter(time => now - time < this.windowMs);

    if (this.requests.length >= this.maxRequests) {
      return false;
    }

    this.requests.push(now);
    return true;
  }

  getTimeUntilReset() {
    if (this.requests.length === 0) return 0;
    
    const oldestRequest = Math.min(...this.requests);
    const resetTime = oldestRequest + this.windowMs;
    return Math.max(0, resetTime - Date.now());
  }
}

export const rateLimiter = new RateLimiter();

// Simple cache implementation
class SimpleCache {
  constructor(ttl = 300000) { // 5 minutes default
    this.cache = new Map();
    this.ttl = ttl;
  }

  set(key, value) {
    this.cache.set(key, {
      value,
      timestamp: Date.now(),
    });
  }

  get(key) {
    const item = this.cache.get(key);
    
    if (!item) return null;
    
    if (Date.now() - item.timestamp > this.ttl) {
      this.cache.delete(key);
      return null;
    }
    
    return item.value;
  }

  clear() {
    this.cache.clear();
  }
}

export const apiCache = new SimpleCache();
```

---

## Phase 5: Testing & Deployment

### 5.1 Testing Strategies

**Unit Testing Example (Jest):**

```javascript
// __tests__/productService.test.js
import { productService } from '../api/products';
import { tradlyApi } from '../api/tradlyApi';

// Mock the API client
jest.mock('../api/tradlyApi');

describe('Product Service', () => {
  beforeEach(() => {
    jest.clearAllMocks();
  });

  test('getProducts returns formatted data', async () => {
    const mockResponse = {
      listings: [{ id: 1, title: 'Test Product' }],
      page: 1,
      total_pages: 5,
      total_records: 100,
    };

    tradlyApi.get.mockResolvedValue(mockResponse);

    const result = await productService.getProducts();

    expect(result.products).toHaveLength(1);
    expect(result.pagination.totalPages).toBe(5);
    expect(tradlyApi.get).toHaveBeenCalledWith('/listings', {
      page: 1,
      per_page: 20,
      sort_by: 'created_at',
      sort_order: 'desc',
    });
  });

  test('getProducts handles errors', async () => {
    tradlyApi.get.mockRejectedValue(new Error('API Error'));

    await expect(productService.getProducts()).rejects.toThrow(
      'Failed to fetch products'
    );
  });
});
```

**Integration Testing:**

```javascript
// __tests__/integration/checkout.test.js
import { cartService } from '../api/cart';
import { orderService } from '../api/orders';

describe('Checkout Flow', () => {
  test('complete checkout process', async () => {
    // Add items to cart
    const product = {
      id: 1,
      title: 'Test Product',
      sale_price: 29.99,
    };

    cartService.addItem(product, 2);
    const cart = cartService.getCart();

    expect(cart.items).toHaveLength(1);
    expect(cart.items[0].quantity).toBe(2);

    // Create order
    const shippingDetails = {
      address: '123 Main St',
      city: 'New York',
      state: 'NY',
      postalCode: '10001',
      country: 'US',
    };

    // Mock authentication
    localStorage.setItem('tradly_auth_token', 'test-token');

    // This would require proper API mocking or test environment
    // const order = await orderService.createOrder(shippingDetails);
    // expect(order.id).toBeDefined();
  });
});
```

### 5.2 Environment Setup for Production

**Production Checklist:**

```javascript
// config/production.js

export const productionChecklist = {
  security: [
    '✓ API keys in environment variables (never in code)',
    '✓ HTTPS enabled for all API calls',
    '✓ Input validation on all user inputs',
    '✓ Rate limiting implemented',
    '✓ CORS properly configured',
    '✓ Authentication tokens stored securely',
  ],
  
  performance: [
    '✓ API response caching implemented',
    '✓ Image optimization (compression, lazy loading)',
    '✓ Pagination for large datasets',
    '✓ Code splitting and lazy loading',
    '✓ CDN for static assets',
  ],
  
  monitoring: [
    '✓ Error tracking (Sentry, etc.)',
    '✓ Analytics integration',
    '✓ API response time monitoring',
    '✓ User session tracking',
  ],
  
  testing: [
    '✓ Unit tests for critical functions',
    '✓ Integration tests for user flows',
    '✓ End-to-end tests for checkout',
    '✓ Cross-browser testing',
  ],
};
```

### 5.3 Deployment Configuration

**Next.js Deployment:**

```javascript
// next.config.js
module.exports = {
  env: {
    TRADLY_API_URL: process.env.TRADLY_API_URL,
    TRADLY_TENANT_ID: process.env.TRADLY_TENANT_ID,
  },
  images: {
    domains: ['api.tradly.app', 'cdn.tradly.app'],
  },
  async headers() {
    return [
      {
        source: '/api/:path*',
        headers: [
          { key: 'Access-Control-Allow-Origin', value: '*' },
          { key: 'Access-Control-Allow-Methods', value: 'GET,POST,PUT,DELETE' },
        ],
      },
    ];
  },
};
```

**Vercel Deployment:**

```bash
# Install Vercel CLI
npm i -g vercel

# Set environment variables
vercel env add TRADLY_API_URL
vercel env add TRADLY_TENANT_ID
vercel env add TRADLY_API_KEY

# Deploy
vercel --prod
```

---

## Common Patterns & Recipes

### Recipe 1: Complete Product Listing Page

```javascript
// pages/Products.js
import { useState, useEffect } from 'react';
import { productService } from '../api/products';
import { categoryService } from '../api/categories';
import { errorHandler } from '../utils/errorHandler';

export default function ProductsPage() {
  const [products, setProducts] = useState([]);
  const [categories, setCategories] = useState([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);
  const [filters, setFilters] = useState({
    categoryId: null,
    search: '',
    page: 1,
  });

  useEffect(() => {
    loadInitialData();
  }, []);

  useEffect(() => {
    loadProducts();
  }, [filters]);

  const loadInitialData = async () => {
    try {
      const cats = await categoryService.getCategories();
      setCategories(cats);
    } catch (err) {
      errorHandler.logError(err, { context: 'loadCategories' });
    }
  };

  const loadProducts = async () => {
    setLoading(true);
    setError(null);

    try {
      const result = await productService.getProducts(filters);
      setProducts(result.products);
    } catch (err) {
      setError(errorHandler.getUserMessage(err));
      errorHandler.logError(err, { context: 'loadProducts', filters });
    } finally {
      setLoading(false);
    }
  };

  const handleSearch = (searchTerm) => {
    setFilters({ ...filters, search: searchTerm, page: 1 });
  };

  const handleCategoryChange = (categoryId) => {
    setFilters({ ...filters, categoryId, page: 1 });
  };

  if (loading) return <div>Loading products...</div>;
  if (error) return <div>Error: {error}</div>;

  return (
    <div>
      <h1>Products</h1>
      
      {/* Search */}
      <input
        type="text"
        placeholder="Search products..."
        onChange={(e) => handleSearch(e.target.value)}
      />

      {/* Category Filter */}
      <select onChange={(e) => handleCategoryChange(e.target.value)}>
        <option value="">All Categories</option>
        {categories.map(cat => (
          <option key={cat.id} value={cat.id}>{cat.name}</option>
        ))}
      </select>

      {/* Product Grid */}
      <div className="product-grid">
        {products.map(product => (
          <ProductCard key={product.id} product={product} />
        ))}
      </div>
    </div>
  );
}
```

### Recipe 2: Shopping Cart Component

```javascript
// components/Cart.js
import { useState, useEffect } from 'react';
import { cartService } from '../api/cart';

export default function Cart() {
  const [cart, setCart] = useState(cartService.getCart());
  const [totals, setTotals] = useState(cartService.calculateTotals());

  useEffect(() => {
    // Listen for cart updates
    const handleCartUpdate = (event) => {
      setCart(event.detail);
      setTotals(cartService.calculateTotals(event.detail));
    };

    window.addEventListener('cartUpdated', handleCartUpdate);
    return () => window.removeEventListener('cartUpdated', handleCartUpdate);
  }, []);

  const updateQuantity = (productId, quantity) => {
    cartService.updateQuantity(productId, quantity);
  };

  const removeItem = (productId) => {
    cartService.removeItem(productId);
  };

  if (cart.items.length === 0) {
    return <div>Your cart is empty</div>;
  }

  return (
    <div className="cart">
      <h2>Shopping Cart</h2>
      
      {cart.items.map(item => (
        <div key={item.id} className="cart-item">
          <img src={item.image} alt={item.title} />
          <h3>{item.title}</h3>
          <p>${item.price}</p>
          
          <input
            type="number"
            min="1"
            value={item.quantity}
            onChange={(e) => updateQuantity(item.id, parseInt(e.target.value))}
          />
          
          <button onClick={() => removeItem(item.id)}>Remove</button>
        </div>
      ))}

      <div className="cart-totals">
        <p>Subtotal: ${totals.subtotal.toFixed(2)}</p>
        <p>Tax: ${totals.tax.toFixed(2)}</p>
        <p>Shipping: ${totals.shipping.toFixed(2)}</p>
        <h3>Total: ${totals.total.toFixed(2)}</h3>
      </div>

      <button onClick={() => window.location.href = '/checkout'}>
        Proceed to Checkout
      </button>
    </div>
  );
}
```

### Recipe 3: Vendor Dashboard

```javascript
// pages/VendorDashboard.js
import { useState, useEffect } from 'react';
import { productService } from '../api/products';
import { orderService } from '../api/orders';
import { authService } from '../api/auth';

export default function VendorDashboard() {
  const [myProducts, setMyProducts] = useState([]);
  const [myOrders, setMyOrders] = useState([]);
  const [stats, setStats] = useState({
    totalProducts: 0,
    totalOrders: 0,
    totalRevenue: 0,
  });

  useEffect(() => {
    loadDashboardData();
  }, []);

  const loadDashboardData = async () => {
    try {
      // Verify vendor authentication
      const user = await authService.getCurrentUser();
      if (user.type !== 'vendor') {
        throw new Error('Access denied. Vendor account required.');
      }

      // Load vendor's products
      const productsData = await productService.getProducts({
        vendorId: user.id,
      });
      setMyProducts(productsData.products);

      // Load vendor's orders
      const ordersData = await orderService.getOrders({
        vendorId: user.id,
      });
      setMyOrders(ordersData.orders);

      // Calculate stats
      const revenue = ordersData.orders.reduce(
        (sum, order) => sum + order.total_amount,
        0
      );

      setStats({
        totalProducts: productsData.products.length,
        totalOrders: ordersData.orders.length,
        totalRevenue: revenue,
      });
    } catch (error) {
      console.error('Dashboard load failed:', error);
    }
  };

  return (
    <div className="vendor-dashboard">
      <h1>Vendor Dashboard</h1>

      {/* Stats */}
      <div className="stats">
        <div className="stat-card">
          <h3>Total Products</h3>
          <p>{stats.totalProducts}</p>
        </div>
        <div className="stat-card">
          <h3>Total Orders</h3>
          <p>{stats.totalOrders}</p>
        </div>
        <div className="stat-card">
          <h3>Total Revenue</h3>
          <p>${stats.totalRevenue.toFixed(2)}</p>
        </div>
      </div>

      {/* Recent Orders */}
      <section>
        <h2>Recent Orders</h2>
        <table>
          <thead>
            <tr>
              <th>Order ID</th>
              <th>Customer</th>
              <th>Amount</th>
              <th>Status</th>
              <th>Actions</th>
            </tr>
          </thead>
          <tbody>
            {myOrders.slice(0, 10).map(order => (
              <tr key={order.id}>
                <td>{order.id}</td>
                <td>{order.customer_name}</td>
                <td>${order.total_amount}</td>
                <td>{order.status}</td>
                <td>
                  <button onClick={() => viewOrderDetails(order.id)}>
                    View
                  </button>
                </td>
              </tr>
            ))}
          </tbody>
        </table>
      </section>

      {/* Products Management */}
      <section>
        <h2>My Products</h2>
        <button onClick={() => window.location.href = '/vendor/products/new'}>
          Add New Product
        </button>
        {/* Product list... */}
      </section>
    </div>
  );
}
```

---

## Troubleshooting Guide

### Common Issues & Solutions

**1. Authentication Errors (401)**
```javascript
// Problem: Token expired or invalid
// Solution: Implement token refresh or re-login

const handleAuthError = () => {
  authService.logout();
  window.location.href = '/login?session_expired=true';
};
```

**2. CORS Errors**
```javascript
// Problem: Cross-origin requests blocked
// Solution: Ensure proper headers or use server-side proxy

// Server-side proxy example (Next.js API route)
// pages/api/tradly/[...path].js
export default async function handler(req, res) {
  const { path } = req.query;
  const apiPath = Array.isArray(path) ? path.join('/') : path;
  
  const response = await fetch(`https://api.tradly.app/${apiPath}`, {
    method: req.method,
    headers: {
      'Content-Type': 'application/json',
      'X-Tenant-Id': process.env.TRADLY_TENANT_ID,
      'Authorization': `Bearer ${process.env.TRADLY_API_KEY}`,
    },
    body: req.method !== 'GET' ? JSON.stringify(req.body) : undefined,
  });
  
  const data = await response.json();
  res.status(response.status).json(data);
}
```

**3. Rate Limiting (429)**
```javascript
// Problem: Too many requests
// Solution: Implement exponential backoff

async function retryWithBackoff(fn, maxRetries = 3) {
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await fn();
    } catch (error) {
      if (error.statusCode === 429 && i < maxRetries - 1) {
        const delay = Math.pow(2, i) * 1000; // Exponential backoff
        await new Promise(resolve => setTimeout(resolve, delay));
        continue;
      }
      throw error;
    }
  }
}
```

---

# Lynxo Integration Learnings (April 2026)

Use this section as a practical implementation reference for checkout/tracking/history flows with Tradly.

## 1. Production-Safe Checkout Rules

- Do not create local-only fallback orders when Tradly checkout fails.
- Keep the user on checkout and surface the real API error.
- Treat Tradly as source of truth for order creation and status.
- Bind checkout email to Tradly session identity. If persisted session email differs from checkout email, clear session and re-run login/register+verify for the new email before placing order.

## 2. Checkout API Sequence (Customer App)

1. `POST /products/v1/cart` for each cart line.
2. `POST /v1/addresses` with a valid `address` object.
3. `GET /v1/tenants/payment_methods` and choose active/default method.
4. `GET /v1/tenants/shipping_methods` and choose active/default method.
5. `POST /products/v1/cart/checkout` with:

```json
{
  "order": {
    "payment_method_id": 13,
    "shipping_method_id": 12,
    "shipping_address_id": 1681
  }
}
```

## 3. Address Payload Shape (Important)

For `POST /v1/addresses`, use address fields expected by docs:

```json
{
  "address": {
    "name": "John Doe",
    "phone_number": "0000000000",
    "address_line_1": "13, Building# 81",
    "address_line_2": "Guerrero St",
    "landmark": "St. James Church",
    "state": "California",
    "post_code": "94110",
    "country": "United States",
    "type": "shipping_address",
    "coordinates": { "latitude": 37.75269, "longitude": -122.42410 }
  }
}
```

## 4. Order Status Mapping (Official Legend)

Map Tradly order status codes exactly:

- `1` Incomplete
- `2` Confirmed
- `3` In progress
- `4` Shipped
- `5` Delivered
- `6` Canceled by customer
- `7` Canceled by admin
- `8` Completed

Terminal statuses: `5`, `6`, `7`, `8`.

## 5. Tracking Engine Guidance

- Poll `GET /products/v1/orders/{order_id}` every 10s for active orders.
- Remove demo auto-advance logic completely.
- Refresh action should fetch from API, not force status transitions.

## 6. Order History Screen Pattern

- List endpoint: `GET /products/v1/orders?page=1&per_page=20&type=orders`
- Optional filter: `order_status` query.
- Detail endpoint: `GET /products/v1/orders/{order_id}`
- Show both `order.id` and `order.reference` for dashboard reconciliation.

## 7. Category API Wiring Pattern

- Fetch categories once: `GET /v1/categories?page=1&per_page=50&parent=0`
- Build a local map from Tradly category names to app category keys.
- Maintain reverse map `localCategory -> tradlyCategoryIds[]`.
- Use `category_id` query in listings (`comma-separated IDs`) when category chip is selected.

**4. Image Upload Failures**
```javascript
// Problem: Large images failing to upload
// Solution: Compress images before upload

async function compressImage(file, maxSizeMB = 1) {
  return new Promise((resolve, reject) => {
    const reader = new FileReader();
    reader.readAsDataURL(file);
    reader.onload = (event) => {
      const img = new Image();
      img.src = event.target.result;
      img.onload = () => {
        const canvas = document.createElement('canvas');
        const ctx = canvas.getContext('2d');
        
        // Calculate new dimensions
        let width = img.width;
        let height = img.height;
        const maxDimension = 1920;
        
        if (width > maxDimension || height > maxDimension) {
          if (width > height) {
            height *= maxDimension / width;
            width = maxDimension;
          } else {
            width *= maxDimension / height;
            height = maxDimension;
          }
        }
        
        canvas.width = width;
        canvas.height = height;
        ctx.drawImage(img, 0, 0, width, height);
        
        canvas.toBlob(
          (blob) => resolve(new File([blob], file.name, { type: 'image/jpeg' })),
          'image/jpeg',
          0.85
        );
      };
    };
    reader.onerror = reject;
  });
}
```

---

## Performance Optimization

### Caching Strategy

```javascript
// utils/cache.js

class CacheManager {
  constructor() {
    this.cache = new Map();
    this.timestamps = new Map();
  }

  set(key, value, ttl = 300000) { // 5 minutes default
    this.cache.set(key, value);
    this.timestamps.set(key, Date.now() + ttl);
  }

  get(key) {
    if (!this.cache.has(key)) return null;
    
    const expiry = this.timestamps.get(key);
    if (Date.now() > expiry) {
      this.delete(key);
      return null;
    }
    
    return this.cache.get(key);
  }

  delete(key) {
    this.cache.delete(key);
    this.timestamps.delete(key);
  }

  clear() {
    this.cache.clear();
    this.timestamps.clear();
  }
}

export const cacheManager = new CacheManager();

// Use with API calls
export async function getCachedProducts(options) {
  const cacheKey = `products_${JSON.stringify(options)}`;
  const cached = cacheManager.get(cacheKey);
  
  if (cached) return cached;
  
  const data = await productService.getProducts(options);
  cacheManager.set(cacheKey, data);
  
  return data;
}
```

### Lazy Loading Images

```javascript
// components/LazyImage.js

export function LazyImage({ src, alt, placeholder }) {
  const [imageSrc, setImageSrc] = useState(placeholder);
  const [isLoaded, setIsLoaded] = useState(false);
  const imgRef = useRef();

  useEffect(() => {
    const observer = new IntersectionObserver(
      (entries) => {
        entries.forEach((entry) => {
          if (entry.isIntersecting) {
            setImageSrc(src);
            observer.disconnect();
          }
        });
      },
      { rootMargin: '50px' }
    );

    if (imgRef.current) {
      observer.observe(imgRef.current);
    }

    return () => observer.disconnect();
  }, [src]);

  return (
    <img
      ref={imgRef}
      src={imageSrc}
      alt={alt}
      className={isLoaded ? 'loaded' : 'loading'}
      onLoad={() => setIsLoaded(true)}
    />
  );
}
```

---

## Quick Reference

### Essential API Endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/users/register` | POST | Register new user |
| `/users/login` | POST | User login |
| `/users/me` | GET | Get current user |
| `/listings` | GET | Get products |
| `/listings/:id` | GET | Get single product |
| `/listings` | POST | Create product |
| `/listings/:id` | PUT | Update product |
| `/listings/:id` | DELETE | Delete product |
| `/categories` | GET | Get categories |
| `/orders` | GET | Get orders |
| `/orders` | POST | Create order |
| `/orders/:id` | GET | Get order details |
| `/orders/:id` | PUT | Update order |
| `/listings/:id/reviews` | GET | Get reviews |
| `/listings/:id/reviews` | POST | Create review |
| `/uploads` | POST | Upload image |

### Status Codes

- `200` - Success
- `201` - Created
- `400` - Bad Request (validation error)
- `401` - Unauthorized (login required)
- `403` - Forbidden (permission denied)
- `404` - Not Found
- `409` - Conflict (duplicate resource)
- `429` - Too Many Requests (rate limit)
- `500` - Server Error

### Best Practices Checklist

- [ ] Store API credentials in environment variables
- [ ] Implement proper error handling with user-friendly messages
- [ ] Add loading states for all async operations
- [ ] Validate user inputs before sending to API
- [ ] Cache frequently accessed data
- [ ] Implement pagination for large datasets
- [ ] Use HTTPS for all API calls
- [ ] Implement rate limiting on client side
- [ ] Add proper TypeScript types or JSDoc comments
- [ ] Write tests for critical flows
- [ ] Monitor API response times
- [ ] Implement analytics tracking
- [ ] Optimize images before upload
- [ ] Use lazy loading for images
- [ ] Implement proper authentication flow
- [ ] Handle token expiration gracefully
- [ ] Add comprehensive logging
- [ ] Implement retry logic for failed requests
- [ ] Use semantic HTML and accessibility features
- [ ] Test on multiple browsers and devices

---

## Production-Verified Patterns (ZS Commerce B2B Marketplace)

> These learnings come from building a live B2B F&B marketplace on Tradly. Several of these correct silent bugs that the generic patterns above will lead you into.

---

### P1. Authentication Headers — What Actually Works

The generic `X-Tenant-Id` header pattern shown above does **not** apply to all Tradly deployments. For standard `pk_live_*` key setups:

```javascript
// ✅ CORRECT — production-verified header pattern
const headers = {
  Accept: "application/json",
  "Content-Type": "application/json",
  "X-TRADLY-AGENT": "3",                   // required — identifies client type
  Authorization: `Bearer ${APP_KEY}`,       // pk_live_* key — always present
};

// When the user is logged in, add their session token:
headers["x-auth-key"] = userToken;         // user JWT — NOT a second Authorization header
```

**Key rules:**
- `X-TRADLY-AGENT: 3` must be present on every request
- App key (`pk_live_*`) goes as `Authorization: Bearer`
- User session token goes as `x-auth-key` (separate from the app key — both are sent simultaneously)
- Do **not** use `X-Tenant-Id` — it is not needed when using the Bearer app key

---

### P2. Search Parameter — Critical Bug

**`q` is silently ignored.** Using `?q=rice` returns all results (no error, no warning). The correct parameter is `search_key`.

```javascript
// ❌ WRONG — silently returns all records
params.set("q", searchTerm);

// ✅ CORRECT
params.set("search_key", searchTerm);
```

This applies to both `/products/v1/listings` and `/v1/accounts`.

---

### P3. Correct Endpoint Paths

The generic guide uses short paths like `/listings`. The real Tradly v1 paths are different:

| Resource | ❌ Generic guide | ✅ Actual path |
|----------|-----------------|---------------|
| Listings list | `/listings` | `/products/v1/listings` |
| Single listing | `/listings/:id` | `/products/v1/listings/:id` |
| Listing by slug | — | `/products/v1/listings/by_slug/:slug` |
| Accounts/Suppliers | — | `/v1/accounts?type=1` |
| Account by slug | — | `/v1/accounts/by_slug/:slug` |
| Categories | `/categories` | `/v1/categories?parent=0` |
| Collections | — | `/v1/collections_data?type=accounts` |
| Attributes schema | — | `/v1/attributes?type=accounts\|listings` |
| Register | `/users/register` | `/v1/users/register` |
| Login | `/users/login` | `/v1/users/login` |
| Verify OTP | — | `/v1/users/verify` |
| Resend OTP | — | `/v1/users/resend_otp` |
| Unique locations | — | `/v1/accounts/unique/locations?type=state` |

**Accounts `type` param:** `type=1` returns supplier/vendor accounts. Always pass it explicitly.

---

### P4. Response Shape — Always Nested Under `data`

All Tradly list responses are wrapped under `data`:

```javascript
// Listings
const listings = payload?.data?.listings ?? [];
const total    = payload?.data?.total_records ?? 0;

// Accounts
const accounts = payload?.data?.accounts ?? [];

// Categories
const categories = payload?.data?.categories ?? [];

// Collections
const collections = payload?.data?.collections ?? [];

// Single record
const listing = payload?.data?.listing ?? payload?.data ?? payload;
const account = payload?.data?.account ?? payload?.data ?? payload;
```

---

### P5. Authentication Flow — Production-Verified Payloads

Auth payloads are strict. For the tested tenant/key setup, login works when you send **email** (not username) with `type: "customer"`.

```javascript
// Step 1 — Register
// ⚠️ Wrap fields inside a `user` object.
const regPayload = await post("/v1/users/register", {
  user: {
    uuid: crypto.randomUUID(),
    type: 2,
    first_name: "Ahmad",
    last_name: "Razak",
    email: "ahmad@example.com".trim().toLowerCase(),
    password: "securepass",
  },
});
const verifyId = regPayload?.data?.verify_id;  // store this

// Step 2 — Verify OTP
// ⚠️ Do NOT wrap in `user: {}` — sent flat
const verifyPayload = await post("/v1/users/verify", {
  verify_id: verifyId,
  code: "123456",
});
const userToken = verifyPayload?.data?.token ?? verifyPayload?.token;

// Resend OTP (if user didn't receive it)
// ⚠️ Back to `user` wrapper for resend
await post("/v1/users/resend_otp", {
  user: { verify_id: verifyId },
});
```

**Login:**
```javascript
// ✅ Tested working payload shape
const loginPayload = await post("/v1/users/login", {
  user: {
    uuid: crypto.randomUUID(),
    type: "customer",
    email: "ahmad@example.com".trim().toLowerCase(),
    password: "securepass",
  },
});
// In this response, token is nested under user.key
const authKey = loginPayload?.data?.user?.key?.auth_key;
const refreshKey = loginPayload?.data?.user?.key?.refresh_key;
```

**Forgot password / reset password (tested flow):**
```javascript
// Step A: Request recovery OTP
const rec = await post("/v1/users/password/recovery", {
  user: { email: "ahmad@example.com".trim().toLowerCase() },
});
const verifyId = rec?.data?.verify_id;

// Step B: Set password with OTP
await post("/v1/users/password/set", {
  verify_id: verifyId,
  code: "123456",
  password: "newSecurePass123",
});
```

**Critical OTP note:** OTP must match the **latest** recovery request (`verify_id`). If you call recovery again, old OTPs become invalid for the new session.

---

### P6. Error Response Structure

Tradly errors come in a specific shape. Never surface the raw error to users.

```javascript
// Error response shape
{
  "timestamp": 1774859685,
  "error": {
    "code": 102,               // Tradly-specific error code
    "message": "User not registered",  // internal message (not for users)
    "type": "user"
  }
}

// ✅ Map error codes to user-friendly messages
const TRADLY_MESSAGES = {
  100: "Something went wrong. Please try again.",
  101: "Session expired. Please log in again.",
  102: "Email or password is incorrect.",
  103: "Account already exists with this email.",
  104: "Verification code is invalid or expired.",
  105: "Verification code is invalid.",
  110: "Your account has been suspended.",
  200: "Item not found.",
  300: "You don't have permission to do that.",
  400: "Invalid request. Please check your details.",
  500: "Something went wrong on our end. Please try again.",
};

function tradlyErrorMessage(code, fallback) {
  if (code && TRADLY_MESSAGES[code]) return TRADLY_MESSAGES[code];
  return fallback ?? "Something went wrong. Please try again.";
}

async function parseApiError(res) {
  try {
    const json = await res.json();
    const code = json?.error?.code;
    return tradlyErrorMessage(code, json?.error?.message ?? json?.message);
  } catch {}
  if (res.status === 401 || res.status === 403) return tradlyErrorMessage(300);
  if (res.status === 404) return tradlyErrorMessage(200);
  if (res.status === 422) return tradlyErrorMessage(400);
  if (res.status >= 500) return tradlyErrorMessage(500);
  return tradlyErrorMessage();
}
```

---

### P7. Location-Based Filtering

Use **backend query params** — never filter on the client side. Client-side filtering on `formatted_address` string is unreliable.

```javascript
// ✅ Backend filtering — accurate
GET /v1/accounts?type=1&state=Selangor&page=1&per_page=20
GET /v1/accounts?type=1&city=Kuala+Lumpur&page=1&per_page=20
GET /products/v1/listings?state=Johor&page=1&per_page=20

// Get list of unique location values for filter dropdowns
GET /v1/accounts/unique/locations?type=state    // all states with accounts
GET /v1/accounts/unique/locations?type=city     // all cities with accounts
GET /v1/accounts/unique/locations?type=country  // all countries

// ❌ Wrong — type=1, type=2 etc. not valid for unique/locations
// ✅ type= only accepts: "state", "city", "country"
```

**Note:** Accounts must have location data set in SuperAdmin for filtering to work. Accounts without location data will not appear in filtered results.

---

### P8. Image Field Normalization

Images can come back as strings, objects, or be missing. Always normalize:

```javascript
function extractImages(raw) {
  const imgs = [];
  if (Array.isArray(raw?.images)) {
    for (const img of raw.images) {
      const src = typeof img === "string" ? img : img?.image_path ?? img?.url ?? "";
      if (src) imgs.push(src);
    }
  }
  // fallback for single image field
  if (!imgs.length && raw?.image_path) imgs.push(raw.image_path);
  return imgs;
}
```

---

### P9. Collections Endpoint

Collections use `/v1/collections_data` (not `/v1/collections`), with a mandatory `type` param:

```javascript
// List all account collections
GET /v1/collections_data?type=accounts&page=1&per_page=20

// Single collection by ID
GET /v1/collections_data?type=accounts&collection_id=abc123

// Response shape
payload?.data?.collections   // array
payload?.data?.total_records // count
```

---

### P11. User Profile — `/v1/users/me` Does Not Work

`GET /v1/users/me` returns **403 "User not found or insufficient permission"** even with a valid token. Use the user's ID from the login response instead:

```javascript
// ❌ WRONG — returns 403
GET /v1/users/me

// ✅ CORRECT — use id from login response
const userId = loginPayload?.data?.user?.id;
GET /v1/users/:userId   // with x-auth-key header
```

**Login response shape** (userId is nested under `data.user`, not `data`):
```javascript
const token   = payload?.data?.user?.key?.auth_key;
const userId  = payload?.data?.user?.id;          // ✅ NOT payload?.data?.id
const acctId  = payload?.data?.user?.account_id;  // may be null
```

Store `userId` in your auth state so `fetchUserProfile` can use it without requiring a separate lookup.

---

### P12. Update User Profile — Use PATCH, Not PUT

`PUT /v1/users/:id` returns **404 "Cannot PUT /v1/users/:id"**. The correct method is `PATCH`:

```javascript
// ❌ WRONG — 404
PUT /v1/users/:id

// ✅ CORRECT
PATCH /v1/users/:id
body: { user: { first_name: "Ahmad", last_name: "Razak" } }

// Also works (no ID needed):
PATCH /v1/users/profile
```

Response shape: `payload?.data?.user` contains the updated user object.

---

### P13. Currency ID — Numeric, Not String

`currency_id` in listing create/update must be a **number**, not the ISO code string:

```javascript
// ❌ WRONG — 422 "Invalid value"
{ currency_id: "MYR" }

// ✅ CORRECT — use the numeric ID from /v1/currencies
{ currency_id: 3905 }   // 3905 = MYR (Malaysian Ringgit) for ZS tenant

// Fetch your tenant's currency list:
GET /v1/currencies   → payload?.data?.currencies → [{ id: 3905, code: "MYR", name: "Malaysian Ringgit" }]
```

Fetch currencies once at startup and cache the ID — it never changes for a given tenant.

---

### P14. Listings — `page` Param Is Required

`GET /products/v1/listings` without `page` returns **422 "Invalid value for param 'page'"**. Always send it:

```javascript
// ❌ WRONG — 422
GET /products/v1/listings?per_page=20

// ✅ CORRECT
GET /products/v1/listings?page=1&per_page=20
```

Also: this endpoint returns **gzip-encoded** responses. In browser `fetch` this is handled automatically. In `curl`, pass `--compressed`.

---

### P15. Account Images — Empty Array Rejected

`POST /v1/accounts` (and `PUT /v1/accounts/:id`) reject `images: []` with **422 "Invalid value"**. Either omit the field entirely when there are no images, or only send it when there is at least one URI:

```javascript
// ❌ WRONG — 422
{ account: { name: "Store", images: [] } }

// ✅ CORRECT — omit images when empty
const body = { name, description, category_id, type: "accounts" };
if (imageUris.length > 0) body.images = imageUris;
POST /v1/accounts  body: { account: body }
```

---

### P16. Update Listing — category_id and currency_id Always Required

`PUT /products/v1/listings/:id` rejects partial updates missing `category_id` or `currency_id`, even if you are only changing the title:

```javascript
// ❌ WRONG — 422 even for a title-only update
PUT /products/v1/listings/:id
{ listing: { title: "New Title", type: "listings" } }

// ✅ CORRECT — always include category_id and currency_id
PUT /products/v1/listings/:id
{ listing: { title: "New Title", category_id: [42423], currency_id: 3905, type: "listings" } }
```

---

### P17. Image Upload (S3 Signed URL)

The endpoint path is `/v1/utils/S3signedUploadURL`. Two-step flow:

```javascript
// Step 1: get signed URL(s)
POST /v1/utils/S3signedUploadURL
{ files: [{ name: "photo.jpg", type: "image/jpeg" }] }
→ [{ signedUrl: "https://s3.amazonaws.com/...", fileUri: "https://cdn.tradly.app/..." }]

// Step 2: PUT the raw file directly to S3 (NO app headers — plain S3 PUT)
await fetch(signedUrl, { method: "PUT", body: fileBlob });

// Step 3: store fileUri in your listing/account payload
{ images: [fileUri] }
```

**Note:** This endpoint has been observed returning **504 Gateway Timeout** intermittently. Implement retry logic or graceful fallback messaging when S3 upload fails.

---

### P18. Account and Listing Create — Response Only Returns ID

Create endpoints return a minimal response — only the new resource ID, not the full object:

```javascript
POST /v1/accounts → { status: true, data: { account: { id: 20957 } } }
POST /products/v1/listings → { status: true, data: { listing: { id: 726568 } } }
```

If you need the full object after creation, make a second fetch: `GET /v1/accounts/:id` or `GET /products/v1/listings/:id`.

---

### P10. Sitemap Generation (Cloudflare Workers)

Vite/React SPAs are static — the Worker intercepts sitemap routes **before** the ASSETS fallback and generates XML on-the-fly by calling the Tradly API server-side. This gives search engines and AI crawlers full URL coverage without any build-time step.

#### Worker structure

```typescript
// worker.ts
interface Env {
  ASSETS: Fetcher;
  TRADLY_API_BASE: string;   // https://api.tradly.app
  TRADLY_AUTH_KEY: string;   // pk_live_...
  SITE_URL: string;          // https://yourdomain.com
}

const PER_PAGE = 100;
const XML_HEADERS = {
  "Content-Type": "application/xml; charset=utf-8",
  "Cache-Control": "public, max-age=3600",
};

function xmlResp(body: string) {
  return new Response(`<?xml version="1.0" encoding="UTF-8"?>\n${body}`, { headers: XML_HEADERS });
}

// Always XML-escape dynamic values to prevent malformed output
function esc(s: string) {
  return s.replace(/&/g, "&amp;").replace(/</g, "&lt;").replace(/>/g, "&gt;").replace(/"/g, "&quot;");
}

async function tradlyGet(env: Env, path: string): Promise<any> {
  const res = await fetch(`${env.TRADLY_API_BASE}${path}`, {
    headers: { Accept: "application/json", Authorization: `Bearer ${env.TRADLY_AUTH_KEY}`, "X-TRADLY-AGENT": "3" },
  });
  if (!res.ok) return null;
  return res.json();
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);
    const path = url.pathname;

    if (request.method === "GET") {
      if (path === "/sitemap-index.xml")   return sitemapIndex(env);
      if (path === "/account-sitemap.xml") return accountSitemap(env, Number(url.searchParams.get("page") ?? 1));
      if (path === "/listing-sitemap.xml") return listingSitemap(env, Number(url.searchParams.get("page") ?? 1));
      if (path === "/category-sitemap.xml")   return categorySitemap(env);
      if (path === "/collection-sitemap.xml") return collectionSitemap(env);
      if (path === "/blog-sitemap.xml")       return blogSitemap(env);
    }

    // SPA fallback — serve index.html for HTML navigation (client-side routing)
    let response = await env.ASSETS.fetch(request);
    if (response.status === 404 && (request.headers.get("accept") || "").includes("text/html")) {
      const indexUrl = new URL("/index.html", request.url);
      response = await env.ASSETS.fetch(new Request(indexUrl.toString(), request));
    }
    return response;
  },
};
```

#### Sitemap index — paginated for large datasets

```typescript
async function sitemapIndex(env: Env): Promise<Response> {
  const siteUrl = env.SITE_URL;

  // Fetch totals in parallel to calculate how many pages each sitemap needs
  const [accData, listData] = await Promise.all([
    tradlyGet(env, `/v1/accounts?type=1&page=1&per_page=1`),
    tradlyGet(env, `/products/v1/listings?page=1&per_page=1`),
  ]);

  const accPages  = Math.max(1, Math.ceil((accData?.data?.total_records  ?? 0) / PER_PAGE));
  const listPages = Math.max(1, Math.ceil((listData?.data?.total_records ?? 0) / PER_PAGE));

  const sitemaps = [
    `  <sitemap><loc>${siteUrl}/sitemap.xml</loc></sitemap>`,          // static pages
    `  <sitemap><loc>${siteUrl}/blog-sitemap.xml</loc></sitemap>`,
    `  <sitemap><loc>${siteUrl}/category-sitemap.xml</loc></sitemap>`,
    `  <sitemap><loc>${siteUrl}/collection-sitemap.xml</loc></sitemap>`,
  ];

  for (let p = 1; p <= accPages; p++)
    sitemaps.push(`  <sitemap><loc>${siteUrl}/account-sitemap.xml?page=${p}</loc></sitemap>`);
  for (let p = 1; p <= listPages; p++)
    sitemaps.push(`  <sitemap><loc>${siteUrl}/listing-sitemap.xml?page=${p}</loc></sitemap>`);

  return xmlResp(`<sitemapindex xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">\n${sitemaps.join("\n")}\n</sitemapindex>`);
}
```

#### Individual sitemap generators

```typescript
async function accountSitemap(env: Env, page: number): Promise<Response> {
  const data = await tradlyGet(env, `/v1/accounts?type=1&page=${page}&per_page=${PER_PAGE}`);
  const accounts: any[] = data?.data?.accounts ?? [];
  const urls = accounts.map((a) => {
    const slug = a.slug || a.id;
    return `  <url><loc>${env.SITE_URL}/a/${esc(String(slug))}</loc><changefreq>weekly</changefreq><priority>0.7</priority></url>`;
  }).join("\n");
  return xmlResp(`<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">\n${urls}\n</urlset>`);
}

async function listingSitemap(env: Env, page: number): Promise<Response> {
  const data = await tradlyGet(env, `/products/v1/listings?page=${page}&per_page=${PER_PAGE}`);
  const listings: any[] = data?.data?.listings ?? [];
  const urls = listings.map((l) => {
    const slug = l.slug || l.id;
    return `  <url><loc>${env.SITE_URL}/l/${esc(String(slug))}</loc><changefreq>weekly</changefreq><priority>0.6</priority></url>`;
  }).join("\n");
  return xmlResp(`<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">\n${urls}\n</urlset>`);
}

async function categorySitemap(env: Env): Promise<Response> {
  const data = await tradlyGet(env, `/v1/categories?parent=0`);
  const cats: any[] = data?.data?.categories ?? [];
  const urls = cats.map((c) =>
    `  <url><loc>${env.SITE_URL}/categories/${esc(String(c.id))}</loc><changefreq>weekly</changefreq><priority>0.7</priority></url>`
  ).join("\n");
  return xmlResp(`<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">\n${urls}\n</urlset>`);
}

async function collectionSitemap(env: Env): Promise<Response> {
  const data = await tradlyGet(env, `/v1/collections_data?type=accounts&page=1&per_page=100`);
  const cols: any[] = data?.data?.collections ?? [];
  const urls = cols.map((c) =>
    `  <url><loc>${env.SITE_URL}/ac/${esc(String(c.id))}</loc><changefreq>weekly</changefreq><priority>0.6</priority></url>`
  ).join("\n");
  return xmlResp(`<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">\n${urls}\n</urlset>`);
}
```

#### Priority and changefreq guide

| Resource | `priority` | `changefreq` |
|----------|-----------|--------------|
| Home / key landing pages | 1.0 | daily |
| Categories | 0.7 | weekly |
| Supplier profiles | 0.7 | weekly |
| Listings | 0.6 | weekly |
| Collections | 0.6 | weekly |
| Blog posts | 0.6 | monthly |
| Static pages (about, contact) | 0.5 | monthly |

#### wrangler.jsonc

```json
{
  "vars": {
    "TRADLY_API_BASE": "https://api.tradly.app",
    "TRADLY_AUTH_KEY": "pk_live_...",
    "SITE_URL": "https://yourdomain.com"
  }
}
```

**Key rules:**
- Always escape `&`, `<`, `>`, `"` in dynamic values with `esc()` — malformed XML breaks Google indexing silently
- Use `slug || id` — some records have slugs, some don't; always fall back to ID
- Use `?page=N` query param on paginated sitemaps so the sitemap index can link to each page separately
- Cache at `max-age=3600` (1 hour) — sitemaps don't need to be real-time

---

### P19. JSON-LD Structured Data for SEO and AI Bots

JSON-LD (`<script type="application/ld+json">`) is the primary way to communicate structured data to Google, Bing, and AI crawlers (GPTBot, ClaudeBot, Perplexity). Add it via a `<Seo>` component using `react-helmet-async`.

#### Seo component pattern (React + react-helmet-async)

```tsx
// components/Seo.tsx
import { Helmet } from "react-helmet-async";

interface SeoProps {
  title?: string;
  description?: string;
  image?: string;
  url?: string;
  noindex?: boolean;
  jsonLd?: object | object[];   // pass one schema or array for multiple
}

export default function Seo({ title, description, image, url, noindex, jsonLd }: SeoProps) {
  const schemas = jsonLd ? (Array.isArray(jsonLd) ? jsonLd : [jsonLd]) : [];
  return (
    <Helmet>
      <title>{title}</title>
      <meta name="description" content={description} />
      <link rel="canonical" href={url} />
      {noindex && <meta name="robots" content="noindex,nofollow" />}
      <meta property="og:title" content={title} />
      <meta property="og:description" content={description} />
      <meta property="og:image" content={image} />
      <meta name="twitter:card" content="summary_large_image" />
      {schemas.map((schema, i) => (
        <script key={i} type="application/ld+json">{JSON.stringify(schema)}</script>
      ))}
    </Helmet>
  );
}
```

#### Schema types by page

**Home page — Organization + WebSite with SearchAction**
```tsx
const jsonLd = [
  {
    "@context": "https://schema.org",
    "@type": "Organization",
    name: "ZS Commerce",
    url: "https://yourdomain.com",
    logo: "https://yourdomain.com/logo.png",
    description: "Malaysia's B2B F&B marketplace",
    address: { "@type": "PostalAddress", addressCountry: "MY" },
  },
  {
    "@context": "https://schema.org",
    "@type": "WebSite",
    name: "ZS Commerce",
    url: "https://yourdomain.com",
    potentialAction: {
      "@type": "SearchAction",
      target: { "@type": "EntryPoint", urlTemplate: "https://yourdomain.com/search?q={search_term_string}" },
      "query-input": "required name=search_term_string",
    },
  },
];
// SearchAction tells Google (and AI) your site has a search box — enables sitelinks search
```

**Listing / Product detail — Product + Offer**
```tsx
const jsonLd = {
  "@context": "https://schema.org",
  "@type": "Product",
  name: listing.title,
  description: listing.description,
  image: listing.images[0],
  url: `https://yourdomain.com/l/${listing.slug || listing.id}`,
  offers: {
    "@type": "Offer",
    priceCurrency: listing.list_price?.currency ?? "MYR",
    price: listing.list_price?.amount,
    availability: "https://schema.org/InStock",
    seller: supplier ? { "@type": "Organization", name: supplier.name } : undefined,
  },
};
```

**Supplier / Account detail — LocalBusiness**
```tsx
const jsonLd = {
  "@context": "https://schema.org",
  "@type": "LocalBusiness",
  name: supplier.name,
  description: supplier.description,
  image: supplier.images?.[0],
  url: `https://yourdomain.com/a/${supplier.slug || supplier.id}`,
  address: {
    "@type": "PostalAddress",
    addressCountry: "MY",
    addressLocality: supplier.location?.city,
    addressRegion: supplier.location?.state,
  },
};
```

**Blog / Article — Article + BreadcrumbList**
```tsx
const jsonLd = {
  "@context": "https://schema.org",
  "@type": "Article",
  headline: post.title,
  description: post.description,
  image: post.featured_image,
  url: `https://yourdomain.com/blog/${post.slug}`,
  datePublished: post.created_at,
  dateModified: post.updated_at ?? post.created_at,
  author: { "@type": "Organization", name: "ZS Commerce" },
  publisher: { "@type": "Organization", name: "ZS Commerce", logo: { "@type": "ImageObject", url: "https://yourdomain.com/logo.png" } },
};
```

**Region / Place page**
```tsx
const jsonLd = {
  "@context": "https://schema.org",
  "@type": "Place",
  name: "Selangor, Malaysia",
  description: "F&B suppliers in Selangor",
  containedInPlace: { "@type": "Country", name: "Malaysia" },
  url: "https://yourdomain.com/malaysia/selangor",
};
```

#### Why JSON-LD matters for AI bots

AI crawlers (GPTBot, ClaudeBot, Perplexity, Google AI Overviews) parse structured data to understand:
- What type of entity a page represents (`Product`, `LocalBusiness`, `Article`)
- Price, availability, and seller for products
- Author, date, and publisher for content
- Geographic coverage for local pages

Without JSON-LD, AI bots have to infer structure from raw HTML — less reliable and less likely to surface your content in AI-generated answers.

**Key rules:**
- Always include `@context` and `@type` — they are not optional
- Use real data from the API response — never hardcode prices or availability
- Multiple schemas on one page: pass an array to `jsonLd` prop
- `noindex` pages should NOT get JSON-LD — there's nothing to index
- Test with Google Rich Results Test and Schema.org Validator before deploying

---

## Additional Resources

### Official Documentation
- **Tradly Developer Portal**: https://developer.tradly.app/
- **SuperAdmin Panel**: https://superadmin.tradly.app/
- **API Reference**: Check developer portal for latest endpoints

### Community & Support
- **Discord/Slack**: Check Tradly website for community links
- **GitHub**: Look for Tradly starter kits and examples
- **Support Email**: Contact via Tradly website

### Starter Kits (Reference)
- **Hummingbirds** - ReactJS web app
- **Butterflies** - NextJS + Tailwind
- **Wolverine** - React Native
- **Penguin** - React Native mobility
- **Kittiwake** - Flutter
- **Arcticfox2** - Kotlin Android

*Note: Starter kits provide pre-built UI components and patterns, but this skill focuses on direct API integration for maximum flexibility.*

---

## Final Notes

This guide provides a comprehensive foundation for building with Tradly API. Remember to:

1. **Start simple** - Build core features first (auth, products, cart)
2. **Test thoroughly** - Write tests as you build
3. **Optimize later** - Get it working, then make it fast
4. **Read the docs** - Check Tradly's official documentation for updates
5. **Ask for help** - Use community resources when stuck

The patterns and code examples in this guide are production-ready but should be adapted to your specific requirements and stack.

Happy building! 🚀
