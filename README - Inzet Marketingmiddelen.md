# Leveranciers Marketing Planner

**Technische Unie** — RFC 909 · Fase 1

Interne webapplicatie voor het beheren van marketingmiddelen, leveranciersplanners en capaciteitsplanning. Gebouwd met React en Supabase (PostgreSQL).

![Status](https://img.shields.io/badge/status-prototype-orange) ![Fase](https://img.shields.io/badge/fase-1-green)

---

## Wat doet deze applicatie?

De Marketing Planner vervangt het huidige Excel-gedreven proces voor het beheren van marketingmiddelen bij Technische Unie. Het systeem ondersteunt twee rollen:

| Rol | Rechten |
|-----|---------|
| **Inkoopmedewerker** | Planners aanmaken, marketingmiddelen koppelen, planning en voortgang bijhouden |
| **Marketingmedewerker** | Marketingmiddelen en capaciteit beheren, alle planners inzien (read-only), exporteren |

### Functionaliteit (4 Epics)

**Epic 1 — Marketingmiddelen beheer**
- Aanmaken, bewerken en verwijderen van marketingmiddelen
- Capaciteitsbeheer per periode met automatische reserveringslogica
- Overzicht met filters op rubriek, segment, campagnetype en beschikbaarheid

**Epic 2 — Leveranciersgegevens**
- Read-only overzicht van leveranciers (SAP-synchronisatie)
- Zoeken op bedrijfsnaam of leveranciernummer

**Epic 3 — Leveranciers Marketing Planner**
- Planners aanmaken per leverancier met statusworkflow (Concept → Actief → Afgerond/Geannuleerd)
- Marketingmiddelen selecteren met capaciteitsindicatie en waarschuwing bij vol
- Planning, voortgang en status per gekoppeld middel
- Cascade-annulering met automatisch vrijgeven van reserveringen

**Epic 4 — Overzicht en rapportage**
- Overzicht van alle planners met filters
- CSV-export van plannergegevens

---

## Snel starten

### Vereisten

- Een [Supabase](https://supabase.com) project (gratis tier is voldoende)
- Een webbrowser

### 1. Database opzetten

Open je Supabase project → **SQL Editor** → voer de scripts uit in deze volgorde:

```
1. supabase_schema.sql          → Tabellen, constraints, triggers, RLS
2. seed_marketingmiddelen.sql   → 56 marketingmiddelen + 148 capaciteitsperiodes
```

> **Let op:** het seed script past ook de UNIQUE constraint aan van `(pakketnaam, rubriek)` naar `(pakketnaam, rubriek, formaat)` omdat meerdere formaten per pakket bestaan.

### 2. Supabase credentials instellen

Open `index.html` en pas de volgende regels aan (regel ~6-7 in het `<script>` blok):

```javascript
const SB_URL = "https://jouw-project.supabase.co";
const SB_KEY = "jouw-anon-key-hier";
```

Je vindt deze in Supabase → **Settings** → **API**.

### 3. Deployen

**Optie A — GitHub Pages (aanbevolen voor demo)**
1. Maak een repo aan en push `index.html`
2. Ga naar Settings → Pages → Source: `main` branch, `/ (root)`
3. Je app is live op `https://gebruiker.github.io/repo-naam/`

**Optie B — Lokaal openen**
Open `index.html` direct in een browser. Werkt out of the box.

**Optie C — Elke statische hosting**
Upload `index.html` naar Vercel, Netlify, Azure Static Web Apps, of een interne webserver.

---

## Bestandsoverzicht

```
├── index.html                      → Volledige applicatie (single-file, ~45KB)
├── supabase_schema.sql             → Database schema + triggers + RLS policies
├── seed_marketingmiddelen.sql      → Marketingmiddelen en capaciteit uit brondata
└── README.md
```

## Database schema

5 tabellen in PostgreSQL (Supabase/Dataverse-equivalent):

```
leverancier                          ← SAP sync, read-only
  └── leveranciers_marketing_planner ← 1 planner per leverancier per periode
        └── planner_marketingmiddel  ← N:M koppeltabel met planning/voortgang

marketingmiddel                      ← Catalogus beheerd door marketing
  ├── marketingmiddel_capaciteit     ← Beschikbare plekken per periode
  └── planner_marketingmiddel        ← (ook FK hierheen)
```

### Relaties

| Van | Type | Naar |
|-----|------|------|
| `leverancier` | 1:N | `leveranciers_marketing_planner` |
| `marketingmiddel` | 1:N | `marketingmiddel_capaciteit` |
| `leveranciers_marketing_planner` | 1:N | `planner_marketingmiddel` |
| `marketingmiddel` | 1:N | `planner_marketingmiddel` |

---

## Data-bronnen

De seed data is afgeleid uit twee interne Excel-bestanden:

| Bron | Tabel | Records | Mapping |
|------|-------|---------|---------|
| `Tarieven_tab.xlsx` | `marketingmiddel` | 56 | Kolom A→rubriek, B→pakketnaam, C→formaat, E→tarief, F→type_campagne, G→segment |
| `Timing_Inzet_Marketingmiddelen.xlsx` | `marketingmiddel_capaciteit` | 148 | Kolom A→middel (match op pakketnaam), maand→periode, publicatiedatum→start |

**Niet opgenomen:**
- spaarTUmee E/W-installateur (tarief: "Nog te bevestigen")
- Werkbus (ontbreekt in Tarieven)

---

## Technische details

| Onderdeel | Technologie |
|-----------|-------------|
| Frontend | React 18 (CDN) + Babel standalone |
| Styling | Inline styles + CSS custom properties |
| Typografie | DM Sans + JetBrains Mono (Google Fonts) |
| Backend | Supabase (PostgreSQL + REST API) |
| Auth | Niet geïmplementeerd (rol-toggle in UI) |
| Hosting | Statisch (enkele HTML file) |

### Beperkingen huidige versie

- **Geen authenticatie** — rolwisseling is een UI toggle, niet afgedwongen
- **Babel-in-browser** — JSX wordt client-side getranspiled (~200ms overhead)
- **Supabase credentials in code** — voor productie verplaatsen naar environment variables
- **Geen offline support** — vereist actieve Supabase connectie
- **Capaciteitslogica client-side** — reserveringen worden via de UI bijgewerkt, niet via database triggers

---

## Doorontwikkeling

### Aanbevolen migratiestappen richting productie

1. **Vite + React project** — migreer van single HTML naar project-structuur:
   ```
   src/
     lib/supabase.js
     components/ui/        → Badge, Btn, Card, Modal, etc.
     pages/
       MiddelenPage.jsx    → Epic 1
       PlannersPage.jsx    → Epic 3+4
       LeveranciersPage.jsx → Epic 2
     App.jsx
   ```

2. **Supabase Auth** — implementeer login met gescheiden rechten per rol via Supabase Row Level Security

3. **Server-side capaciteitslogica** — verplaats reserveringsupdates naar PostgreSQL triggers of Supabase Edge Functions

4. **SAP integratie** — koppel leverancierstabel aan SAP via Power Automate of een ETL-pipeline

5. **Dynamics 365 integratie** — afstemming met RFC 1026 (Customer Insights) zodra het fundament staat

---

## Context

Dit project valt onder **RFC 909 — Vervanging Inzet Marketingmiddelen**, een strategisch initiatief van Technische Unie dat zich richt op het digitaliseren van marketingmiddelenprocessen. Het is inhoudelijk geaccordeerd door het IMO (begin 2025) en wordt gepositioneerd als logisch vervolg op de Dynamics 365 Customer Insights implementatie (RFC 1026).

Zie het [Functioneel Ontwerp](./docs/) voor de volledige specificatie inclusief user stories en acceptatiecriteria.

---

## Licentie

Intern gebruik — Technische Unie.
