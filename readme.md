# Condition Editor

In the real application, this is part of a **condition builder** that determines which resolution options (Return By Mail, Exchange, Store Credit, etc.) are available to a customer based on their **return cart** — an order combined with the customer's in-progress return selections (which items they're returning and why). You're building a slice of that system: a UI for creating conditions and a backend for evaluating them against return carts.

Build a condition editor with two parts:

1. A **Node/Express backend** that serves data and evaluates conditions
2. A **React frontend** where users create conditions by picking a field, an operator, and a value

The editor also includes a nested reason selector with checkboxes.

Use TypeScript for all code.

---

## Getting Started

```bash
# Backend
cd backend
npm install
npm run dev    # Runs on http://localhost:3001

# Frontend (in a separate terminal)
cd frontend
npm install
npm run dev    # Runs on http://localhost:3000, proxies /api to backend
```

Types are defined in `frontend/src/types.ts` and `backend/src/types.ts`.

---

## Guidelines

- **Time:** You have ~45-50 minutes. You're not expected to finish everything — we care about your process, not completion.
- **AI:** Use AI tools freely, but don't paste this entire README as a single prompt. We want to see how you break down problems and iterate with AI.
- **Evaluation:** We're watching how efficiently you use AI tooling, how you prompt, and how you debug when things go wrong.
- **Testing:** Vitest is pre-configured in both frontend and backend (`npm test`) if you want to write tests.

---

## Part 1 — Backend

Build the API endpoints listed below. Route stubs already exist in `backend/src/routes/`. The orders endpoint already works — you need to implement the rest.

### How order data and return cart data fit together

When a customer starts a return, there are two kinds of data:

- **Order data**: the original purchase (email, billing amount, shipments, items, etc.)
- **Return cart data**: what the customer is doing *right now* in the returns flow (which items they selected to return, and the reason chosen for each item)

Each line being returned has **only one reason**: every `return_item` has a single `reason_id` field (not a list). The same line never has multiple reasons attached at once.

For this exercise, **rules are evaluated against a return cart**. The return cart includes order information plus return-specific information (reasons). Think of the request to `POST /api/conditions/evaluate` as a **snapshot in time** of the customer’s return cart.

### `GET /api/reasons`

Return a reason tree. The tree must be at least 3 levels deep with multiple branches. Use this structure:

```
Fit
  Too large
    Collar, Sleeves, Waist
  Too small
    Collar, Sleeves, Waist
  Too short
    Collar, Sleeves, Waist
Item damaged
  Wrinkled
    Severely, Easily noticeable, Somewhat noticeable
  Ripped
    Severely, Easily noticeable, Somewhat noticeable
  Thread lose
    Severely, Easily noticeable, Somewhat noticeable
```

Response shape:

```json
[
  {
    "id": "fit",
    "label": "Fit",
    "children": [
      {
        "id": "too-large",
        "label": "Too large",
        "children": [
          { "id": "too-large-collar", "label": "Collar" }
        ]
      }
    ]
  }
]
```

### `GET /api/condition-fields`

Return the list of fields that can be used in conditions. Each field has a type and a set of valid operators.

These fields map to properties inside the **return cart** payload (see `GET /api/return-carts` for the full structure). Most fields come from the cart’s embedded order info, but **Reason** comes from the cart’s `return_items`.

| Field | Type | Path in Order | Notes |
|-------|------|---------------|-------|
| Customer Email | string | `customer.email` | |
| Order Total | number | `billing.amount` | |
| Item Price | number | `order_items.unit_price` | Array field |
| Carrier | string | `shipments.carrier` | Array field |
| Country | string | `customer.address.country` | |
| Is Final Sale | boolean | `order_items.is_final_sale` | Array field |
| Is Gift | boolean | `order_items.is_gift` | Array field |
| Item Color | string | `order_items.color` | Array field |
| Reason | special | `return_items[].reason_id` | One reason per return line only; values come from `/api/reasons` |

Operators by type:
- **string**: is, is not, contains, starts with, ends with
- **number**: is, is not, >, >=, <, <=
- **boolean**: is, is not

### `GET /api/orders`

Already implemented. Returns the raw sample orders from `backend/src/data/orders.ts`. These show the underlying order structure for reference. There are 4 orders with different characteristics (high/low value, domestic/international, gifts, final sale items, different carriers, etc.).

### `GET /api/return-carts`

Already implemented. Returns sample return carts from `backend/src/data/return-carts.ts`. Each return cart combines order data with `return_items` — the items the customer is returning. Each return line has **exactly one** `reason_id` from the reason tree (never more than one reason per line). These carts are the **evaluation target** for conditions. There are 4 carts covering different scenarios (fit reasons, damage reasons, final sale items, bulk returns).

### `POST /api/conditions/evaluate`

Accept a set of conditions and a return cart. Return whether the cart matches.

Conditions can be nested using AND/OR groups. Here is an example request:

```json
{
  "conditions": {
    "logic": "AND",
    "conditions": [
      { "field": "customer.email", "operator": "CONTAINS", "value": "@example.com" },
      {
        "logic": "OR",
        "conditions": [
          { "field": "billing.amount", "operator": "GT", "value": 500 },
          { "field": "order_items.unit_price", "operator": "GT", "value": 100 }
        ]
      }
    ]
  },
  "cart": { ... }
}
```

Response: `{ "match": true }`

Some fields (like `order_items.unit_price`) point to arrays in the cart's order info. For example, an order can have many items, each with its own price. You need to decide: does `order_items.unit_price GT 100` mean **"any item over $100"** or **"all items over $100"**? The choice is yours — be prepared to explain your reasoning.

The **Reason** field evaluates against `return_items[].reason_id` in the cart. Each returned line has only that one reason ID (no list of reasons per line). The reason IDs come from the reason tree (e.g., `"too-large-collar"`, `"ripped"`). A design decision: does matching check for an exact leaf reason, or should a parent ID like `"fit"` match any reason under it (prefix matching)? Think about what makes sense for the condition builder UX.

**Example rules your endpoint should handle:**
- `customer.email CONTAINS @example.com` — email contains "@example.com"
- `billing.amount GT 500` — order total is greater than 500
- `order_items.unit_price GT 100` — item price over 100 (your semantics)
- `shipments.carrier EQ fedex` — carrier is "fedex"
- `order_items.is_final_sale EQ true` — final sale check (your semantics)
- `customer.address.country NOT_EQ US` — country is not "US"
- `return_items.reason_id EQ too-large-collar` — return reason matches (your semantics)

---

## Part 2 — Frontend

Build a condition editor UI.

### Condition Rows

Each condition row has three inputs:
1. **Field dropdown** — shows the fields from `GET /api/condition-fields`
2. **Operator dropdown** — shows only the operators valid for the selected field's type
3. **Value input** — text input for strings, number input for numbers, checkbox/dropdown for booleans

### Adding Conditions

Users can add more condition rows. Each row is joined to the others by AND or OR.

### Nested Reason Selector

When the user picks **"Reason"** as the field, the value input changes to a checkbox dropdown. This dropdown shows the reason tree from `GET /api/reasons`.

**How it should work:**

1. Show the reason tree as nested checkboxes inside a dropdown
2. **Checking a box** checks that item and all items below it. Items at the same level and above are not affected.
   - Example: Everything is unchecked. You check "Fit". Now Fit, Too large, Too small, Too short, and all their children (Collar, Sleeves, Waist) are checked. "Item damaged" is still unchecked.
3. **Unchecking a box** unchecks that item and all items below it. Items at the same level and the parent stay as they are.
   - Example: Everything is checked. You uncheck "Collar" (under Too large). Only Collar becomes unchecked. "Sleeves" and "Waist" stay checked. "Too large" shows a dash (indeterminate) because not all of its children are checked.
4. **Indeterminate state (dash):** When some children are checked and some are not, the parent shows a dash instead of a checkmark. This applies at every level — if "Too large" shows a dash, then "Fit" also shows a dash.
5. **Summary display:** Show the selected values as chips/tags:
   - If all children of a parent are selected: `"Fit > All"`
   - If some children are selected: `"Fit > Too large > Collar, Sleeves"`
   - A mix of both is possible — show multiple chips
6. Individual leaf items (items with no children) can always be checked or unchecked on their own

### Evaluate Button

Add an "Evaluate" button. When clicked, it sends the current conditions to `POST /api/conditions/evaluate` along with one of the sample return carts from `GET /api/return-carts`. Display whether the cart matched or not. You can let the user pick which cart to test, or use a hardcoded one.

---

## Stretch Goals

If you have extra time, consider these extensions:

### Nested Condition Groups

Within a single case, support sub-groups that have their own AND/OR logic. For example: `(email contains "@vip.com" OR total > 1000) AND country is "US"`. See `screenshots/nested-group.png` and `screenshots/condition-joining.png` for reference.

### Multiple Cases

Support a Case 1 / Case 2 / Else structure where each case has its own set of conditions and maps to a different resolution option. This mirrors the real application: Case 1 conditions → Return By Mail, Case 2 → Exchange, Else → Store Credit. See `screenshots/secondary-case.png` and `screenshots/condition-builder.png` for reference.
