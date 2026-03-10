# chords.to

**Visualize, diagnose, and fix supply chain leakage in Bitcoin circular economies.**

chords.to maps the vendor relationships within a local Bitcoin economy as an interactive chord diagram — revealing exactly where merchants must exit to fiat because a BTC-accepting supplier doesn't yet exist.

---

## The Problem

Bitcoin circular economies leak value. A butcher accepts BTC but pays rent in fiat. A café accepts BTC but buys packaging from a supplier that doesn't. Every fiat exit is a leak — value leaving the loop.

chords.to makes these leaks visible so communities can close them.

## How It Works

- **Merchants self-report** their business type and what they need to buy (their dependencies)
- **The chord diagram** draws connections between vendor types — orange for BTC-covered supply chains, dashed green for fiat leaks
- **The leakage panel** ranks the highest-impact gaps so the community knows where to recruit next
- **Organizers** moderate submissions and manage their community's data via a token-protected dashboard

---

## Demo

Open `chords-to-prototype.html` in any browser to see the full interactive visualization with mock data — no server required.

---

## Core Concepts

| Term | Definition |
|------|-----------|
| **Community** | A local Bitcoin circular economy (e.g. "Bitcoin Ekasi", "Nairobi BTC Circle") |
| **Vendor Type** | A category of business (e.g. Butcher, Knife Supplier, Packaging) |
| **Dependency** | A vendor type that a business needs to purchase from to operate |
| **Chord** | A visual link between two vendor types showing a spending relationship |
| **Leakage** | A dependency where no BTC-accepting supplier exists in the community |
| **Coverage** | A dependency where ≥1 BTC-accepting supplier exists |

---

## Stack

| Layer | Technology |
|-------|-----------|
| Frontend | Static HTML/CSS/JS, D3.js v7 |
| Backend | Node.js / Express |
| Database | PostgreSQL |
| Process Manager | PM2 |
| Deployment | EC2 (Ubuntu) |

---

## URL Structure

```
chords.to/                     → Landing page / community directory
chords.to/:slug                → Community chord diagram (public)
chords.to/:slug/contribute     → Merchant self-reporting form (public)
chords.to/:slug/admin          → Organizer dashboard (token-protected)
```

---

## Project Structure

```
chords-to/
├── server/
│   ├── index.js              # Express entry point
│   ├── routes/
│   │   ├── communities.js
│   │   ├── vendorTypes.js
│   │   ├── businesses.js
│   │   ├── dependencies.js
│   │   └── chordData.js      # Aggregated data for visualization
│   ├── middleware/
│   │   └── auth.js           # Admin token verification
│   └── db.js                 # PostgreSQL connection pool
├── public/
│   ├── index.html            # Landing page
│   ├── community.html        # Chord diagram page
│   ├── contribute.html       # Merchant self-reporting form
│   ├── admin.html            # Organizer dashboard
│   ├── js/
│   │   ├── chord.js          # D3 chord diagram
│   │   └── app.js            # Client-side routing/fetch
│   └── css/
│       └── style.css
├── sql/
│   └── schema.sql
├── chords-to-prototype.html  # Self-contained prototype (mock data)
├── .env
├── package.json
└── ecosystem.config.js
```

---

## Getting Started

### Prerequisites

- Node.js 18+
- PostgreSQL

### Setup

```bash
# Clone the repo
git clone https://github.com/tando-me/chords.git
cd chords

# Install dependencies
npm install

# Create the database
createdb chords_to
psql chords_to < sql/schema.sql

# Configure environment
cp .env.example .env
# Edit .env with your database credentials

# Start dev server
npm run dev
```

### Environment Variables

```
PORT=3000
DATABASE_URL=postgresql://user:pass@localhost:5432/chords_to
NODE_ENV=development
```

### Production (PM2)

```bash
pm2 start ecosystem.config.js
```

---

## API Reference

### Communities
```
POST   /api/communities                        Create community (returns admin token)
GET    /api/communities                        List all communities
GET    /api/communities/:slug                  Community info + stats
PUT    /api/communities/:slug                  Update community (admin)
```

### Vendor Types
```
GET    /api/communities/:slug/vendor-types     List vendor types
POST   /api/communities/:slug/vendor-types     Create vendor type (admin)
PUT    /api/vendor-types/:id                   Edit (admin)
DELETE /api/vendor-types/:id                   Remove (admin)
```

### Businesses
```
GET    /api/communities/:slug/businesses       All businesses
POST   /api/communities/:slug/businesses       Merchant self-report (public)
PUT    /api/businesses/:id                     Edit (admin)
DELETE /api/businesses/:id                     Remove (admin)
```

### Dependencies
```
GET    /api/businesses/:id/dependencies        List dependencies
POST   /api/businesses/:id/dependencies        Declare dependency (public)
DELETE /api/dependencies/:id                   Remove (admin)
```

### Visualization
```
GET    /api/communities/:slug/chord-data       Full data payload for chord diagram
GET    /api/communities/:slug/leakage          Ranked leakage report
```

---

## Roadmap

- [x] Frontend prototype with interactive chord diagram
- [ ] Phase 1 — Backend: Express + PostgreSQL + all API endpoints
- [ ] Phase 2 — Wire frontend to live API
- [ ] Phase 3 — Community creation & landing page
- [ ] Phase 4 — Merchant self-onboarding form
- [ ] Phase 5 — Organizer admin dashboard
- [ ] BTC Map API integration for merchant verification
- [ ] Embeddable chord diagram widget
- [ ] Cross-community vendor suggestions
- [ ] Time-series view (watch economy grow)

---

## Auth Model

| Role | Auth | Permissions |
|------|------|-------------|
| Viewer | None | View diagram, leakage report, vendor list |
| Merchant | Public form | Add business, declare dependencies |
| Organizer | Bearer token (emailed at creation) | Manage vendor types, moderate submissions |

MVP uses no user accounts. Organizers receive a token link at community creation (e.g. `chords.to/nairobi-btc/admin?token=abc123`).

---

## Contributing

This is an early-stage open project. If you run or participate in a Bitcoin circular economy and want to map it, open an issue or reach out.

---

## License

MIT
