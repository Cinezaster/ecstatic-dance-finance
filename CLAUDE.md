# Ecstatic Dance · Financiële Simulator

## Project overview

A single-file HTML financial simulator for an ecstatic dance community event. No build step, no framework, no dependencies — everything runs in one `index.html` file that can be opened locally or hosted on GitHub Pages.

The tool has two purposes:
1. **Planning** — simulate costs, pricing and break-even before an event, project cash flow over a year.
2. **Recording** — log actual event results (revenue, costs, participants) to a shared Supabase database so the whole organising team can track finances over time.

Live URL: https://cinezaster.github.io/ecstatic-dance-finance/

---

## Architecture

| Layer | Choice | Why |
|---|---|---|
| UI | Vanilla HTML/CSS/JS | No build tooling, easy to edit and deploy |
| Charts | Custom SVG (no library) | Zero dependencies, works offline |
| Storage | Supabase (PostgreSQL + REST API) | Free tier, shared across team, no backend code needed |
| Hosting | GitHub Pages | Free, zero config, version controlled |

The entire app is one file: `index.html`. All JS is inline `<script>`, all CSS is inline `<style>`. There is no `package.json`, no bundler, no node_modules.

---

## Tabs

### 1. Per Event (`tab-event`)
Live break-even calculator. All inputs are sliders or number fields; results update on every change.

**Inputs:**
- Betalende deelnemers (paying participants) — range 10–120
- Vrijwilligers (volunteers, free entry) — range 0–12
- Standaard ticket price (€)
- Sociaal tarief (reduced/solidarity fee, €)
- % paying social fee
- Editable cost line items (naam + bedrag)
- Materiaalfonds % (equipment fund, % of gross revenue, max 30%)
- Vrijwilligerswaardering (volunteer appreciation, € per event)

**Outputs:**
- Bruto omzet (gross revenue)
- Totale kosten (total costs = fixed + fund + volunteer appreciation)
- Netto resultaat (net result)
- Break-even participant count
- Profit bar (visual fill showing % of break-even reached)
- Insight text (good / warn / bad)
- Horizontal bar chart (SVG) breaking down each cost category vs revenue

**Button:** "Event registreren" — opens a modal pre-filled with calculated values so the user can save this event to Supabase.

### 2. Projectie (`tab-projectie`)
12-month forward projection. Inherits all pricing/cost settings from the Per Event tab.

**Inputs:**
- Frequency: tweewekelijks (biweekly, 2/month) or wekelijks (weekly, 4–5/month)
- Startmaand (which calendar month to start from, dropdown)
- Gem. deelnemers per event (average participants)
- Groei per maand % (monthly participant growth rate, 0–10%)
- Eenmalige investering (one-time equipment investment, €)
- In welke maand (which month number, 1–12, the investment lands)
- Startkapitaal (starting cash buffer)

**Outputs:**
- 6 KPI boxes: total events, total revenue, total costs, year net, equipment fund accumulated, end cash balance
- Grouped bar chart (revenue bars + cost bars) with cumulative cash line overlay (SVG)
- Monthly breakdown table
- Equipment fund accumulation line chart with target line (SVG)

**Events per month logic:**
```js
// Biweekly: always 2
// Weekly: [4,4,4,5,4,4,5,4,4,4,5,4] — months with 5 Saturdays
```

### 3. Registraties (`tab-records`)
Historical record of actual events, stored in Supabase.

**Features:**
- Load all records on tab open (ordered by date desc)
- KPI summary row (total events, total revenue, total net, average participants)
- Bar chart of revenue per event (chronological order)
- Full table with: datum, naam, deelnemers, omzet, kosten, netto badge (green/red), notities, delete button
- "Nieuw event" button → manual entry modal
- "Vernieuwen" button → re-fetch from Supabase

**Empty / error states:** handled gracefully — shows a helpful message and link to Instellingen if Supabase is not configured.

### 4. Instellingen (`tab-instellingen`)
Step-by-step setup guide + Supabase credentials input.

**Credentials** are stored in `localStorage` (keys: `sb_url`, `sb_key`). Each team member enters their own credentials once in their browser. The anon public key is safe to share within the team.

**Connection test:** calls `GET /rest/v1/dans_events?limit=1&select=id` and shows a green/red dot.

**GitHub Pages deploy guide** is embedded as numbered steps A–D.

---

## Data model

### Supabase table: `dans_events`

```sql
create table dans_events (
  id             bigint primary key generated always as identity,
  datum          date not null,
  naam           text,
  deelnemers_betaald  integer,
  deelnemers_sociaal  integer,
  vrijwilligers  integer,
  omzet_bruto    numeric,
  kosten_vast    numeric,
  kosten_fonds   numeric,
  kosten_vrijw   numeric,
  netto          numeric,
  notities       text,
  created_at     timestamptz default now()
);
alter table dans_events enable row level security;
create policy "team_access" on dans_events
  for all using (true) with check (true);
```

**Column meanings:**
| Column | Meaning |
|---|---|
| datum | Event date |
| naam | Optional label (e.g. "Dance #42") |
| deelnemers_betaald | Total paying participants |
| deelnemers_sociaal | Of those, how many paid the social/reduced fee |
| vrijwilligers | Volunteers present (free entry) |
| omzet_bruto | Gross ticket revenue |
| kosten_vast | Fixed costs (DJ + venue + ceremony + technician + organiser) |
| kosten_fonds | Equipment fund amount set aside this event |
| kosten_vrijw | Volunteer appreciation paid out |
| netto | Net result (omzet_bruto − kosten_vast − kosten_fonds − kosten_vrijw) |
| notities | Free-text notes |

---

## Key formulas

```
gross_revenue   = (participants * (1 - social_pct) * std_price)
                + (participants * social_pct * social_price)

equipment_fund  = gross_revenue * equipment_pct

total_costs     = fixed_costs + equipment_fund + volunteer_appreciation

net_result      = gross_revenue - total_costs

break_even      = (fixed_costs + volunteer_appreciation)
                / (avg_ticket_price * (1 - equipment_pct))

avg_ticket      = std_price * (1 - social_pct) + social_price * social_pct
```

---

## Default values (as of current build)

| Parameter | Default |
|---|---|
| Standaard ticket | €20 |
| Sociaal tarief | €7 |
| % sociaal tarief | 15% |
| Deelnemers | 70 |
| Vrijwilligers | 6 |
| DJ | €250 |
| Zaalverhuur | €150 |
| Ceremonieleider | €60 |
| Technicus | €40 |
| Organisatiekosten | €500 |
| Materiaalfonds % | 5% (max 30%) |
| Vrijwilligerswaardering | €25/event |
| Groei per maand | 2% |
| Eenmalige investering | €4.000 |

---

## Supabase API integration

All Supabase calls go through a single helper:

```js
async function sbFetch(method, path, body=null) {
  const {url, key} = getSB(); // reads from localStorage
  const res = await fetch(url + '/rest/v1/' + path, {
    method,
    headers: {
      'apikey': key,
      'Authorization': 'Bearer ' + key,
      'Content-Type': 'application/json',
      'Prefer': 'return=representation'
    },
    body: body ? JSON.stringify(body) : null
  });
  if (!res.ok) throw new Error(await res.text());
  if (res.status === 204) return null;
  return res.json();
}
```

**Calls used:**
| Action | Call |
|---|---|
| Load records | `GET dans_events?order=datum.desc&select=*` |
| Save event | `POST dans_events` with JSON body |
| Delete event | `DELETE dans_events?id=eq.{id}` |
| Test connection | `GET dans_events?limit=1&select=id` |

---

## SVG chart system

All charts are drawn with vanilla SVG — no Chart.js or other library. The helper functions are:

| Function | Chart type | Used in |
|---|---|---|
| `tekenEventGrafiek(d)` | Horizontal bar breakdown | Per Event tab |
| `tekenProjGrafiek(rijen)` | Grouped bars + cumulative line | Projectie tab |
| `tekenMateriaalGrafiek(rijen, inv)` | Area line + target dashed line | Projectie tab |
| `tekenRecordGrafiek(data)` | Vertical bars (omzet + netto overlay) | Registraties tab |

All charts use a fixed `viewBox` (e.g. `0 0 700 300`) and `width:100%` on the SVG so they scale responsively. Colours come from CSS custom properties (`--accent`, `--accent2`, etc.).

---

## Styling

CSS custom properties (defined on `:root`):

```css
--bg:       #0f0e17   /* page background */
--surface:  #1a1828   /* card background */
--surface2: #221f35   /* input background, inner boxes */
--border:   #2e2a48   /* borders */
--accent:   #b48ef7   /* purple — primary highlight, revenue */
--accent2:  #7ecfb3   /* teal/green — positive results, cumulative line */
--accent3:  #f7c59f   /* orange — costs */
--red:      #f77c7c   /* negative results, loss */
--text:     #e8e3f5   /* body text */
--muted:    #8b87aa   /* labels, secondary text */
```

---

## Cost line items

Stored in a JS array `kostenRegels` (not in Supabase — these are the *planned* per-event defaults, not historical records):

```js
let kostenRegels = [
  {naam: 'DJ',                bedrag: 250},
  {naam: 'Zaalverhuur',       bedrag: 150},
  {naam: 'Ceremonieleider',   bedrag: 60},
  {naam: 'Technicus',         bedrag: 40},
  {naam: 'Organisatiekosten', bedrag: 500},
];
```

Users can add/remove lines. These settings are **not persisted** between page reloads — they reset to defaults. A future improvement would be to save them to `localStorage` or Supabase.

---

## Known limitations & future improvements

- **Cost line items not persisted** — the planned cost breakdown resets on reload. Could be saved to `localStorage`.
- **No authentication** — Supabase uses a permissive RLS policy. Anyone with the URL and anon key can read/write. For a small trusted team this is fine; for broader use, add Supabase Auth.
- **No edit on records** — records can only be deleted and re-created. An edit modal would improve UX.
- **No export** — adding a CSV export button for the Registraties tab would be useful for bookkeeping.
- **Projection vs actuals** — the Projectie tab uses the simulator's planned costs. A future "Vergelijking" tab could overlay projected vs actual data from Supabase.
- **Multi-currency / VAT** — currently € only, no VAT handling.
- **Mobile layout** — the SVG charts don't reflow for very narrow screens. Could add a simplified mobile chart or use CSS `transform: scale()`.

---

## File structure

```
ecstatic-dance-finance/
└── index.html        ← entire application (single file)
└── CLAUDE.md         ← this file
```

---

## Deployment

**GitHub Pages:**
1. Repo: `https://github.com/cinezaster/ecstatic-dance-finance`
2. File must be named `index.html` at repo root
3. Enable: Settings → Pages → Source: Deploy from branch → main → / (root)
4. Live at: `https://cinezaster.github.io/ecstatic-dance-finance/`

**Supabase project:** credentials stored per-user in `localStorage` under keys `sb_url` and `sb_key`.
