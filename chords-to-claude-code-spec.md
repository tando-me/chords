# chords.to — Full Project Specification for Development

> **STATUS: PARKED (2026-07-17, Jason)** — no active development planned; the phase plan below is preserved for whenever this revives. See `_Ecosystem/architecture/plans-registry.md`.

## Overview

chords.to is a platform for Bitcoin circular economies to visualize, diagnose, and fix supply chain leakage. Each local economy maps its vendor relationships as an interactive chord diagram, revealing where merchants must exit to fiat because a BTC-accepting supplier doesn't exist yet.

**Domain:** chords.to

A working HTML prototype already exists (`chords-to-prototype.html`) with the complete frontend visualization using mock data. The development task is to build the backend, database, and wire the prototype to a live API.

---

## Core Concepts

| Term | Definition |
|------|-----------|
| **Community** | A local Bitcoin circular economy (e.g., "Bitcoin Ekasi", "Nairobi BTC Circle") |
| **Vendor Type** | A category of business (e.g., Butcher, Knife Supplier, Packaging) |
| **Business** | An actual named merchant within a vendor type |
| **Dependency** | A vendor type that a business needs to purchase from to operate |
| **Chord** | A visual link between two vendor types showing a spending relationship |
| **Leakage** | A dependency where no BTC-accepting supplier exists in the community |
| **Coverage** | A dependency where ≥1 BTC-accepting supplier exists |

---

## Architecture

### Stack

| Layer | Technology |
|-------|-----------|
| **Frontend** | Static HTML/CSS/JS served by Express, D3.js v7 for chord diagram |
| **Fonts** | DM Sans (body/headings), Space Mono (monospace accents, stats) |
| **Theme** | Dark background (#0a0a0f), BTC orange (#f7931a), fiat green (#4ade80), warning red (#ef4444), gray (#5a5666) |
| **Backend** | Node.js / Express on EC2 (Ubuntu) |
| **Database** | PostgreSQL |
| **Process Manager** | PM2 |
| **Future** | BTC Map API (v3) integration for merchant verification |

### Multi-Tenant URL Structure

```
chords.to/                          → Landing page / community directory
chords.to/:slug                     → Community page (chord diagram + leakage panel, public)
chords.to/:slug/contribute          → Merchant self-reporting form (public)
chords.to/:slug/admin               → Organizer dashboard (token-protected)
```

### Project Structure

```
chords-to/
├── server/
│   ├── index.js              # Express app entry point
│   ├── routes/
│   │   ├── communities.js    # Community CRUD
│   │   ├── vendorTypes.js    # Vendor type CRUD
│   │   ├── businesses.js     # Business CRUD
│   │   ├── dependencies.js   # Dependency CRUD
│   │   └── chordData.js      # Aggregated data for visualization
│   ├── middleware/
│   │   └── auth.js           # Admin token verification
│   └── db.js                 # PostgreSQL connection pool
├── public/
│   ├── index.html            # Landing page
│   ├── community.html        # Chord diagram page (based on prototype)
│   ├── contribute.html       # Merchant self-reporting form
│   ├── admin.html            # Organizer dashboard
│   ├── js/
│   │   ├── chord.js          # D3 chord diagram (extract from prototype)
│   │   └── app.js            # Client-side routing/fetch logic
│   └── css/
│       └── style.css         # Extracted from prototype
├── sql/
│   └── schema.sql            # Database schema
├── .env                      # DB credentials, PORT, etc.
├── package.json
└── ecosystem.config.js       # PM2 config
```

---

## Database Schema

### Tables

```sql
-- Communities (multi-tenant root)
CREATE TABLE communities (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    slug            VARCHAR(80) UNIQUE NOT NULL,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    organizer_email VARCHAR(255) NOT NULL,
    organizer_token VARCHAR(255) NOT NULL,      -- simple bearer token for admin
    location        JSONB,                       -- { lat, lng, city, country }
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW()
);

-- Vendor Types (organizer-approved categories)
CREATE TABLE vendor_types (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    community_id    UUID REFERENCES communities(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    icon            VARCHAR(50),                 -- emoji icon key
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(community_id, name)
);

-- Businesses (actual merchants, self-reported)
CREATE TABLE businesses (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    community_id    UUID REFERENCES communities(id) ON DELETE CASCADE,
    vendor_type_id  UUID REFERENCES vendor_types(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    accepts_btc     BOOLEAN DEFAULT TRUE,
    btcmap_id       VARCHAR(255),                -- optional link to BTC Map element
    lightning_address VARCHAR(255),              -- optional, for future verification
    contact_info    JSONB,                       -- { phone, email, website }
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- Dependencies (what each business needs to buy)
CREATE TABLE dependencies (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    business_id     UUID REFERENCES businesses(id) ON DELETE CASCADE,
    vendor_type_id  UUID REFERENCES vendor_types(id) ON DELETE CASCADE,
    priority        INT DEFAULT 1,               -- 1 = most critical
    notes           TEXT,
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(business_id, vendor_type_id)
);
```

### Derived View for Visualization

```sql
CREATE VIEW chord_matrix AS
SELECT
    b.community_id,
    bt.id AS source_type_id,
    bt.name AS source_type,
    bt.icon AS source_icon,
    d.vendor_type_id AS target_type_id,
    vt.name AS target_type,
    vt.icon AS target_icon,
    COUNT(DISTINCT b.id) AS weight,
    BOOL_OR(
        EXISTS(
            SELECT 1 FROM businesses b2
            WHERE b2.vendor_type_id = d.vendor_type_id
            AND b2.community_id = b.community_id
            AND b2.accepts_btc = TRUE
        )
    ) AS has_btc_supplier
FROM dependencies d
JOIN businesses b ON d.business_id = b.id
JOIN vendor_types bt ON b.vendor_type_id = bt.id
JOIN vendor_types vt ON d.vendor_type_id = vt.id
GROUP BY b.community_id, bt.id, bt.name, bt.icon, d.vendor_type_id, vt.name, vt.icon;
```

---

## Auth Model

| Role | Authentication | Permissions |
|------|---------------|-------------|
| **Viewer** | None needed | View chord diagram, leakage report, vendor list |
| **Merchant** | Community link + simple form | Add their business, declare dependencies |
| **Organizer** | Bearer token (emailed at community creation) | Approve/edit vendor types, manage businesses, moderate data |

MVP uses no user accounts. Organizer gets a token link (e.g., `chords.to/nairobi-btc/admin?token=abc123`). Merchants submit freely via the public contribute page. Organizer can moderate/remove entries.

---

## API Endpoints

### Communities
```
POST   /api/communities                     → Create community (returns admin token)
GET    /api/communities                     → List all communities (for landing page)
GET    /api/communities/:slug               → Public community info + stats
PUT    /api/communities/:slug               → Update community (admin token required)
```

### Vendor Types
```
GET    /api/communities/:slug/vendor-types  → List all vendor types with business counts
POST   /api/communities/:slug/vendor-types  → Create vendor type (admin)
PUT    /api/vendor-types/:id                → Edit vendor type (admin)
DELETE /api/vendor-types/:id                → Remove vendor type (admin)
```

### Businesses
```
GET    /api/communities/:slug/businesses                  → All businesses
GET    /api/communities/:slug/businesses?vendor_type=:id  → Filter by type
POST   /api/communities/:slug/businesses                  → Merchant self-report (public)
PUT    /api/businesses/:id                                → Edit (admin)
DELETE /api/businesses/:id                                → Remove (admin)
```

### Dependencies
```
GET    /api/businesses/:id/dependencies     → List dependencies for a business
POST   /api/businesses/:id/dependencies     → Merchant declares dependency (public)
DELETE /api/dependencies/:id                → Remove (admin)
```

### Chord Data (aggregated for visualization)
```
GET    /api/communities/:slug/chord-data    → Returns:
```

The `/chord-data` endpoint must return everything the frontend needs in a single call:

```json
{
  "community": {
    "name": "Nairobi BTC Circle",
    "slug": "nairobi-btc",
    "location": { "city": "Nairobi", "country": "Kenya" }
  },
  "vendorTypes": [
    {
      "id": "uuid",
      "name": "Butcher",
      "icon": "🥩",
      "businesses": [
        { "id": "uuid", "name": "Mama Nyama Meats", "accepts_btc": true },
        { "id": "uuid", "name": "Prime Cuts Kenya", "accepts_btc": false }
      ]
    }
  ],
  "dependencies": [
    {
      "source_type_id": "uuid",
      "target_type_id": "uuid",
      "weight": 3,
      "has_btc_supplier": true
    }
  ],
  "stats": {
    "total_vendor_types": 12,
    "total_businesses": 34,
    "btc_links": 18,
    "fiat_links": 13,
    "health_pct": 58
  }
}
```

### Leakage
```
GET    /api/communities/:slug/leakage       → Ranked leakage report:
```

```json
{
  "leakage": [
    {
      "vendor_type_id": "uuid",
      "vendor_type_name": "Landlord / Rent",
      "icon": "🏠",
      "businesses_affected": 15,
      "has_any_businesses": true,
      "btc_accepting_count": 0,
      "total_business_count": 2,
      "sources": ["Café / Restaurant", "Grocer", "Butcher", "Transport"]
    }
  ]
}
```

---

## Frontend Design Spec (Finalized in Prototype)

### Page Layout

```
┌─────────────────────────────────────────────────┐
│  h1: Visualizing Bitcoin Circular Economy Health │
│  h2: Stop Leaking Value to Fiat!                │
│  h3: See the gaps. Close the loop.              │
├─────────────────────────────────────────────────┤
│  Community Banner (full width)                   │
│  ┌──────────────────┬──────────────────────────┐│
│  │ h4: ₿ Community  │  [18 BTC links] [⚠ 13]  ││
│  │ Location · stats  │                          ││
│  ├──────────────────┴──────────────────────────┤│
│  │ 58%  of supply chains fulfilled in BTC      ││
│  │ ◄──────── gradient bar with arrow ────────► ││
│  │ 0% BTC                           100% BTC   ││
│  └─────────────────────────────────────────────┘│
├─────────────────────────────────────────────────┤
│  [All Connections] [Leakage Only] [BTC Only]     │
├─────────────────────────────────────────────────┤
│           ┌─────────────────────┐                │
│           │                     │                │
│           │   Chord Diagram     │                │
│           │   (760px max,       │                │
│           │    centered)        │                │
│           │                     │                │
│           └─────────────────────┘                │
│    ── BTC covered  - - Fiat leak  █ Arc health   │
├────────────────────┬────────────────────────────┤
│  Top Leakage       │  Vendor Types              │
│  Points            │  (clickable, with          │
│  (with CTA desc)   │   BTC health dot)          │
└────────────────────┴────────────────────────────┘
```

### Heading Hierarchy

| Level | Copy |
|-------|------|
| **h1** | Visualizing Bitcoin Circular Economy Health |
| **h2** | Stop Leaking Value to Fiat! |
| **h3** | See the gaps. **Close the loop.** (bold orange) |
| **h4** | Community name (e.g., "₿ Nairobi BTC Circle") |

### Color Language

**Chords (connections between vendor types):**
- Orange solid chord (#f7931a, 0.45 opacity) = BTC-covered supply chain
- Green dashed chord (#4ade80, 0.5 opacity, stroke-dasharray: 4,3) = Fiat leakage
- Chord thickness = number of businesses reporting that dependency
- Goal: turn all dashed green lines into solid orange

**Arcs (vendor type segments around the ring):**
- Green-to-orange gradient based on BTC acceptance ratio (btc_businesses / total_businesses)
- 100% BTC = full orange (#f7931a)
- 0% BTC = full green (#4ade80)
- Intermediate values interpolate RGB between the two
- Gray (#5a5666) = vendor type has zero businesses at all
- Pure-sink vendor types (no outbound dependencies, only inbound) still get arcs — use diagonal padding in the matrix to ensure D3 allocates them space

**Health gradient bar (in community banner):**
- Full-width bar: green (#4ade80) on left → orange (#f7931a) on right
- White arrow indicator positioned at community's BTC health percentage
- Labels: "0% BTC" left, "100% BTC" right
- Large white percentage number + "of supply chains fulfilled in BTC"

**Stat chips:**
- BTC links: subtle orange background/border, orange text
- Fiat leaks: red background (#ef4444), pulsing red border, clickable → scrolls to leakage panel with flash highlight animation

**Leakage badges:**
- ⚠ FIAT (red) = vendor type has businesses but none accept BTC
- ✕ GAP (red, brighter) = vendor type has zero businesses at all

### Interactions

- **Hover arc:** Highlight all connected chords, dim others to 8% opacity. Tooltip shows: business count, BTC acceptance ratio, inbound/outbound dependency counts.
- **Click arc:** Detail overlay slides up inside diagram showing list of actual businesses with ₿ BTC / FIAT badges per business.
- **Hover chord:** Highlight chord, show animated flow dots traveling from source to target vendor type (orange dots for BTC, green dots for fiat). Tooltip shows dependency direction, weight, and coverage status.
- **Filter toggles:** "All Connections" / "Leakage Only" / "BTC Only" — filters the chord matrix and re-renders. Diagonal entries (sink padding) are preserved during filtering so sink arcs remain visible.
- **Fiat leaks chip → leakage panel:** Smooth scroll with flash highlight animation on the panel card.
- **Vendor list sidebar:** Click any vendor type to open the detail overlay in the diagram.

### Legend (HTML footer below diagram)

Horizontal row, centered: solid orange line (BTC covered) | dashed green line (Fiat leak) | green→orange gradient swatch (Arc BTC health) | gray dot (No vendors)

### Leakage Panel

Title: "Top Leakage Points"

Description text: "These types of vendors are leaking value back into the fiat economy. Consider approaching them and bringing them into the Bitcoin circular economy to close the loop. Keep the value of bitcoin in your community!"

Items ranked by businesses_affected (descending). Each item shows: icon + name, "X businesses affected · Fiat only / No vendor exists", badge (⚠ FIAT or ✕ GAP).

### Key Technical Notes from Prototype

- D3.js v7 chord layout with `padAngle(0.04)`
- Label margin: `max(90, size * 0.13)` to prevent clipping of long vendor type names
- Font size scales: `max(10, min(13, size / 55))`
- SVG `overflow: visible` on the SVG element
- `mix-blend-mode: screen` on ribbon paths for visual depth on dark background
- CSS variables (var()) do NOT work in SVG attributes — always use hex values in D3 code
- Arc thickness: `outerRadius - innerRadius = max(16, outerRadius * 0.07)`
- Pure-sink vendor types need diagonal matrix padding: `matrix[i][i] = totalInboundWeight`
- Self-chord ribbons (diagonal) must be filtered out: `.filter(d => d.source.index !== d.target.index)`
- Flow dot animation: emit every 300ms, 800ms travel duration, `d3.easeCubicInOut`, clean up via `clearInterval` on mouseout and re-render
- Arc color interpolation formula (JS):
  ```js
  const btcRatio = btcBusinesses / totalBusinesses;
  const r = Math.round(74 + (247 - 74) * btcRatio);
  const g = Math.round(222 + (147 - 222) * btcRatio);
  const b = Math.round(128 + (26 - 128) * btcRatio);
  // rgb(74,222,128) = #4ade80 at ratio 0, rgb(247,147,26) = #f7931a at ratio 1
  ```

---

## Build Phases

### Phase 1: Backend Foundation (NEXT)

1. Initialize Node.js project with Express, pg, dotenv, cors, helmet
2. Create PostgreSQL database and run schema.sql
3. Implement `/api/communities` CRUD endpoints
4. Implement `/api/communities/:slug/vendor-types` CRUD
5. Implement `/api/communities/:slug/businesses` CRUD
6. Implement `/api/businesses/:id/dependencies` CRUD
7. Implement `/api/communities/:slug/chord-data` aggregation endpoint
8. Implement `/api/communities/:slug/leakage` report endpoint
9. Implement admin auth middleware (bearer token check)
10. Seed database with the mock data from the prototype for testing
11. PM2 ecosystem config

### Phase 2: Wire Frontend to API (NEXT)

1. Extract CSS and JS from prototype into separate files
2. Replace hardcoded mock data with `fetch('/api/communities/:slug/chord-data')`
3. Make community page dynamic — read slug from URL path
4. Update stats, leakage panel, vendor list from API response
5. Test all interactions work with live data

### Phase 3: Community Creation & Landing Page

1. Landing page at `/` listing existing communities
2. "Create a Circular Economy" form → POST /api/communities
3. Organizer receives token link
4. Community page accessible at `/:slug`

### Phase 4: Merchant Self-Onboarding Form

1. Public `/contribute` page per community
2. Form: "I am a [vendor type] and I need to buy from: 1.___ 2.___ 3.___"
3. Autocomplete against existing vendor types
4. If vendor type doesn't exist → flag for organizer approval
5. Confirmation with link back to updated chord diagram

### Phase 5: Organizer Admin Dashboard

1. Token-protected admin page at `/:slug/admin?token=xxx`
2. Manage vendor types (add/edit/remove)
3. View/moderate business submissions
4. View/remove dependencies

### Future (Post-MVP)

- BTC Map API integration (v3) for merchant cross-referencing
- Cross-community vendor suggestions
- Exportable leakage reports (CSV)
- Embeddable chord diagram widget
- Time-series view (watch economy grow over time)
- Gamification (community health score milestones)
- Lightning-based verification of BTC acceptance

---

## Reference Files

- `chords-to-prototype.html` — Complete working frontend prototype with mock data. This is the source of truth for all visual design, interactions, and D3 code. Extract and adapt for production.

---

## Environment Setup

```bash
# Database
createdb chords_to
psql chords_to < sql/schema.sql

# Project
npm init -y
npm install express pg dotenv cors helmet crypto

# Dev
npm install --save-dev nodemon

# PM2
pm2 start ecosystem.config.js
```

### .env

```
PORT=3000
DATABASE_URL=postgresql://user:pass@localhost:5432/chords_to
NODE_ENV=development
```

### ecosystem.config.js

```js
module.exports = {
  apps: [{
    name: 'chords-to',
    script: 'server/index.js',
    env: {
      NODE_ENV: 'production',
      PORT: 3000
    }
  }]
};
```
