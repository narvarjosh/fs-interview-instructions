# Please do not pull this repo. It's for reading and interpreting back to your agent.

# Condition Editor

In the real application, this is part of a **condition builder** that determines which resolution options (Return By Mail, Exchange, Store Credit, etc.) are available to a customer based on their **return cart** — an order combined with the customer's in-progress return selections. You're building a slice of that system: a UI for creating conditions and a backend for evaluating them against return carts.

The exercise is split into two parts:

1. **Part 1 — Condition Builder.** This is the core of the exercise: a **Node/Express backend** that serves condition fields and evaluates conditions, and a **React frontend** where users create conditions by picking a field, an operator, and a value.
2. **Part 2 — Reason Conditions.** Once Part 1 works end to end, add a special "Reason" field backed by a nested checkbox selector.

Focus on Part 1 first. Only move on to Part 2 when you can build conditions in the UI and evaluate them against a cart.

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

---

## Guidelines

- **Time:** You have ~45-50 minutes. You're not expected to finish everything — we care about your process, not completion. Getting Part 1 working end to end matters more than reaching Part 2.
- **AI:** Use AI tools freely, but don't paste this entire README as a single prompt. We want to see how you break down problems and iterate with AI.
- **Evaluation:** We're watching how efficiently you use AI tooling, how you prompt, and how you debug when things go wrong.
- **Testing:** Vitest is pre-configured in both frontend and backend (`npm test`) if you want to write tests.
- **Screenshots:** The images in `screenshots/` come from a working reference implementation of this exercise, grouped to match the parts below (`part-1/`, `part-2/`, `stretch/`). They're there to pin down *behaviour* — which fields exist, what the reason selector does, how selections are summarised. Don't try to match the visual design; we don't grade styling.

---

## Part 1 — Condition Builder

Build the API endpoints and the editor UI described below. Route stubs already exist in `backend/src/routes/`. The orders and return-carts endpoints already work — you implement the rest.

### How order data and return cart data fit together

When a customer starts a return, there are two kinds of data:

- **Order data**: the original purchase (email, billing amount, shipments, items, etc.)
- **Return cart data**: what the customer is doing *right now* in the returns flow (which items they selected to return)

For this exercise, **conditions are evaluated against a return cart**. The return cart includes the order information plus return-specific information. Think of the request to `POST /api/conditions/evaluate` as a **snapshot in time** of the customer's return cart.

### Backend

#### `GET /api/condition-fields`

Return the list of fields that can be used in conditions. Each field has a type and a set of valid operators.

Each field describes something about the **return cart** (see `GET /api/return-carts` for the full structure). Where each one lives in the payload, and how you identify it in a condition, is up to you.

| Field | Type |
|-------|------|
| Customer Email | string |
| Order Total | number |
| Item Price | number |
| Carrier | string |
| Country | string |
| Is Final Sale | boolean |
| Is Gift | boolean |
| Item Color | string |

Operators by type:
- **string**: is, is not, contains, starts with, ends with
- **number**: is, is not, >, >=, <, <=
- **boolean**: is, is not

#### `GET /api/orders`

Already implemented. Returns the raw sample orders from `backend/src/data/orders.ts`. These show the underlying order structure for reference. There are 4 orders with different characteristics (high/low value, domestic/international, gifts, final sale items, different carriers, etc.).

#### `GET /api/return-carts`

Already implemented. Returns sample return carts from `backend/src/data/return-carts.ts`. Each return cart combines order data with `return_items` — the items the customer is returning. These carts are the **evaluation target** for conditions. There are 5 carts covering different scenarios (high/low value, final sale items, bulk returns, partial returns). You'll use the `reason_id` on each return line in Part 2.

#### `POST /api/conditions/evaluate`

Accept a set of conditions and a return cart. Return whether the cart matches.

Conditions can be nested using AND/OR groups. Here is an example request:

```json
{
  "conditions": {
    "logic": "AND",
    "conditions": [
      { "field": "customer_email", "operator": "CONTAINS", "value": "@example.com" },
      {
        "logic": "OR",
        "conditions": [
          { "field": "order_total", "operator": "GT", "value": 500 },
          { "field": "item_price", "operator": "GT", "value": 100 }
        ]
      }
    ]
  },
  "cart": { ... }
}
```

Response: `{ "match": true }`

The `field` identifiers above are illustrative. How you identify a field on the wire is your call, as long as the frontend and backend agree.

Some fields don't have a single value per cart. Item Price is the obvious one: an order has many items, each with its own price. You need to decide what `Item Price > 100` means — **"any item over $100"**, **"all items over $100"**, or something else — and which items it applies to. The choice is yours — be prepared to explain your reasoning.

**Example rules your endpoint should handle:**
- Customer Email **contains** `@example.com`
- Order Total **>** `500`
- Item Price **>** `100` (your semantics)
- Carrier **is** `fedex`
- Is Final Sale **is** `true` (your semantics)
- Country **is not** `US`

### Frontend

Build a condition editor UI. See `screenshots/part-1/condition-builder.png` for the overall shape: a case containing condition rows, and a way to evaluate the result against a sample cart.

#### Condition Rows

Each condition row has three inputs:
1. **Field dropdown** — shows the fields from `GET /api/condition-fields`
2. **Operator dropdown** — shows only the operators valid for the selected field's type
3. **Value input** — text input for strings, number input for numbers, checkbox/dropdown for booleans

#### Adding Conditions

Users can add more condition rows. Each row is joined to the others by AND or OR.

#### Evaluate Button

Add an "Evaluate" button. When clicked, it sends the current conditions to `POST /api/conditions/evaluate` along with one of the sample return carts from `GET /api/return-carts`. Display whether the cart matched or not. You can let the user pick which cart to test, or use a hardcoded one.

See `screenshots/part-1/evaluate-result.png` for the reference implementation's version, including the request body it sends.

---

## Part 2 — Reason Conditions

Only start this part once Part 1 works end to end.

When a customer returns an item, they pick a **reason** from a nested tree (e.g., Fit → Too large → Collar). In the return cart, every `return_item` has a single `reason_id` field (not a list) — the same line never has multiple reasons attached at once.

### Backend

#### `GET /api/reasons`

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

#### Add "Reason" to `GET /api/condition-fields`

| Field | Type | Notes |
|-------|------|-------|
| Reason | special | One reason per return line only; values come from `/api/reasons` |

#### Extend `POST /api/conditions/evaluate`

The **Reason** field evaluates against `return_items[].reason_id` in the cart. Each returned line has only that one reason ID (no list of reasons per line). The reason IDs come from the reason tree (e.g., `"too-large-collar"`, `"ripped"`). A design decision: does matching check for an exact leaf reason, or should a parent ID like `"fit"` match any reason under it (prefix matching)? Think about what makes sense for the condition builder UX.

**Example rule your endpoint should handle:**
- `return_items.reason_id EQ too-large-collar` — return reason matches (your semantics)

### Frontend — Nested Reason Selector

When the user picks **"Reason"** as the field, the value input changes to a checkbox dropdown. This dropdown shows the reason tree from `GET /api/reasons`. See `screenshots/part-2/nested-reasons-all.png` and `screenshots/part-2/nested-reasons-some.png` for reference.

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

---

## Stretch Goals

If you have extra time, consider these extensions:

### Nested Condition Groups

Within a single case, support sub-groups that have their own AND/OR logic. For example: `(email contains "@vip.com" OR total > 1000) AND country is "US"`. See `screenshots/stretch/nested-group.png` and `screenshots/stretch/condition-joining.png` for reference.

### Multiple Cases

Support a Case 1 / Case 2 / Else structure where each case has its own set of conditions and maps to a different resolution option. This mirrors the real application: Case 1 conditions → Return By Mail, Case 2 → Exchange, Else → Store Credit. See `screenshots/stretch/secondary-case.png` and `screenshots/part-1/condition-builder.png` for reference.
