# Sales Order Management — SAP BTP ABAP RAP Project

A full-featured Sales Order Management application built on **SAP BTP ABAP Environment** using the **ABAP RESTful Application Programming Model (RAP)**, exposed via **OData V4** and consumed through **SAP Fiori Elements**.

This project was built independently (self-taught, no internship) to demonstrate practical, end-to-end RAP development — from data modeling through business logic to a polished Fiori UI — including several advanced features (draft handling, side effects, cascading calculations) that go beyond typical CRUD tutorials.

---

## Live Feature Set

- **Composition-based data model** — Sales Order Header with a dependent Sales Order Item composition (deep create supported)
- **Master data integration** — Customer Master and Material Master as separate read-only RAP entities, linked via associations and value help
- **Draft handling** — full Edit / Activate / Discard / Resume / Prepare lifecycle, matching standard SAP Fiori UX patterns
- **Side effects** — live UI field refresh (e.g., selecting a Material instantly updates Unit, Unit Price, and Item Amount without requiring a save)
- **Status lifecycle** — Open → Released → Completed, or Open → Cancelled, driven by custom actions with dynamic button enablement (`get_instance_features`)
-- **Cascading business calculations**:
  - Material selection auto-fills Unit Price, Unit, and Currency from Material Master
  - Item Amount = Quantity × Unit Price (auto-calculated)
  - Header Total Amount = sum of all Item Amounts, net of an order-level Discount %
- **Multi-currency support**:
  - Header currency is fixed to INR (the company's reporting currency), enforced via a system-set determination and a readonly field
  - Materials can be priced in any supported currency (USD, EUR, GBP, JPY, AUD, CAD, AED, SGD); on item creation, price is automatically converted to INR using a dedicated Exchange Rate Master table
  - A secondary USD reference figure (`unit_price_usd`, `item_amount_usd`) is calculated per item for stakeholders who think in dollars, independent of the item's original sourcing currency
- **Validations**:
  - - Mandatory field checks (Sales Order ID, Customer ID, Order Date)
  - Order Date cannot be in the past
  - Referential integrity — Customer ID must exist in Customer Master
  - Uniqueness check — Sales Order ID cannot be duplicated (correctly distinguishes create vs. update)
  - Stock availability — ordered quantity cannot exceed Material Master stock
  - Item lock — items cannot be modified once an order leaves "Open" status
- **Value Help** on Customer ID, Material ID, and Currency (via standard `I_Currency`)
- **Custom numbering** — both Sales Order ID sequencing and Item Number sequencing (10, 20, 30…) implemented via early numbering / association-based numbering
- **Status shown as color-coded criticality badges** (Open / Released / Completed / Cancelled) with clean text-only display
- **Full audit trail** — Created By/On and Changed By/On, auto-populated via semantic annotations, displayed in an Administrative Data facet

---

## Architecture

```
Database Tables (zso_header_19, zso_item_19, zso_customer_19, zso_material_19, zso_exchange_rate_19)
        │
        ▼
Interface CDS Views (ZI_SO_HEADER_19, ZI_SO_ITEM_19, ZI_SO_CUSTOMER_19, ZI_SO_MATERIAL_19)
        │   — calculated fields (StatusText, StatusCriticality)
        │   — associations (_Items, _Customer, _Material, _Header)
        │   — value help, semantic annotations
        ▼
Behavior Definitions (Interface level)
        │   — draft, determinations, validations, actions, side effects
        │   — numbering (header + association-based item numbering)
        ▼
Behavior Implementation Class (ZBP_I_SO_HEADER_19 + item handler class)
        │   — all business logic: calculations, validations, feature control
        ▼
Projection CDS Views (ZC_SO_HEADER_19, ZC_SO_ITEM_19)
        │
        ▼
Projection Behavior Definitions
        │   — use draft, use side effects, use association, use action
        ▼
Metadata Extensions (ZSO_MDE_HEADER, ZSO_MDE_FOR_ITEM)
        │   — UI facets, field groups, line items, criticality, side effects targets
        ▼
Service Definition → Service Binding (OData V4, UI)
        │
        ▼
SAP Fiori Elements (List Report + Object Page)
```

---

## Entities

| Entity | Type | Purpose |
|---|---|---|
| `ZI_SO_HEADER_19` / `ZC_SO_HEADER_19` | Root, draft-enabled | Sales Order Header — status lifecycle, discount, total |
| `ZI_SO_ITEM_19` / `ZC_SO_ITEM_19` | Composition child, draft-enabled | Sales Order Items — quantity, pricing, amount |
| `ZI_SO_CUSTOMER_19` | Root, read-only | Customer Master (value help + referential validation source) |
|| `ZI_SO_MATERIAL_19` | Root, read-only | Material Master (value help + price/stock lookup source) |
| `ZI_SO_EXCHANGE_RATE_19` | Root, read-only | Exchange Rate Master — currency-to-INR conversion rates |
---

## Key Technical Challenges Solved

Building this wasn't a straight line — a few of the harder problems worked through during development:

- **Infinite recursion in determinations** — an `on modify` determination that updated the same field it was scoped to trigger on caused a "stack of on-modify determinations too deep" shortdump. Fixed by scoping the determination to specific trigger fields (`field quantity, unit_price`) instead of a broad `create; update;`.
- **Deep-create key violation** — early numbering for composition children (`SalesOrder\_Items`) initially wiped the inherited parent key by rebuilding rows with `APPEND VALUE #(...)` instead of modifying the framework-supplied `%target` rows in place — causing a "changed a key value already supplied by caller" shortdump.
- **Delete-time association failure** — recalculating the header total after an item delete via `READ ENTITIES ... BY \_Header` failed because the association couldn't resolve on an already-deleted child. Fixed by reading `sales_order_uuid` directly from the item's own key instead of navigating the association.
- **Draft vs. projection BDEF syntax** — the interface BDEF uses `with draft;` and `draft action ... optimized;`, while the projection BDEF requires the different `use draft;` / `use action` syntax, plus `with draft;` inside each association block. Easy to get wrong and got several rounds of syntax errors before landing on the correct combination.
- **Side effects scoping** — SAP RAP only allows a single trigger field to appear once per `side effects` block; multiple target fields for the same trigger must be comma-separated under one statement, not repeated as separate statements.
- **Silent mapping gap** — a field (`discount_percent`) was exposed in the CDS view and used correctly within a single request, but was never persisted because it was missing from the BDEF's `mapping for` block — a good reminder that CDS exposure and DB persistence mapping are two separate steps.
 - **Currency conversion consistency** — since the header currency is fixed to INR but materials can be priced in any currency, conversion logic had to run at the point of item creation (not deferred), and a separate reference-currency calculation had to be scoped carefully (`field unit_price, quantity`) to avoid re-triggering the same recursion issue seen earlier in the project.
---

## Tech Stack

- **SAP BTP ABAP Environment** (trial)
- **ABAP RESTful Application Programming Model (RAP)** — managed implementation
- **Core Data Services (CDS)** — interface & projection views, associations, compositions, value help, semantic annotations
- **OData V4 (UI)** via Service Binding
- **SAP Fiori Elements** — List Report + Object Page
- **Eclipse ABAP Development Tools (ADT)**

---

## Screenshots

![List Report](screenshots/list-report.png)
   ![Object Page](screenshots/object-page.png)
   ![Items with Material Auto-fill](screenshots/items-table.png)

---

## Author

Tanmay Patil — self-taught SAP ABAP Developer, actively seeking an SAP ABAP Developer role.
[LinkedIn](#) · [Resume](#)
