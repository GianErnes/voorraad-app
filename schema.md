# Voorraad-app — Database schema

Supabase project: **schilder-voorraad** (`rcwlbcfuvfprnnkypbba.supabase.co`)
Alle tabellen hebben Row Level Security ingeschakeld.

> Dit schema is afgeleid uit de app-code. Verifieer kolomtypes via 
> Supabase Table Editor als je twijfelt; types onderaan staan met "?" 
> waar onzeker.

---

## `products`

Hoofdtabel met alle artikelen in de voorraad.

| Kolom | Type | Default | Notitie |
|---|---|---|---|
| `id` | serial | auto | Primary key |
| `name` | text | — | Productnaam (intern, leesbaar voor team) |
| `cat` | text | '' | Categorie (vrij tekstveld) |
| `unit` | text | 'stuk' | Eenheid: stuk, liter, rol, blik, kg, set, ml |
| `qty` | numeric | 0 | Huidige voorraad (3 decimalen na step-0,125 update) |
| `min` | numeric | 0 | Minimum voorraad — onder dit getal komt 'm op de bestellijst |
| `target` | numeric | NULL | Streefvoorraad — bestelling wordt hierheen aangevuld |
| `pack` | numeric | 1 | Verpakkingseenheid (rond besteladvies hierop af) |
| `price` | numeric | 0 | Inkoopprijs per `unit` |
| `supplier_id` | int | NULL | FK → `suppliers.id` |
| `barcode` | text | NULL | Streepjescode (optioneel) |
| `factuur_naam` | text | NULL | Naam zoals leverancier 't op factuur noemt — gebruikt in bestellijsten |
| `last_ordered_at` | timestamptz | NULL | Datum laatste bestelling. NULL = niet besteld of al ontvangen. Failsafe na 10 werkdagen. |
| `ordered_qty` | numeric | 0 | v2.10.0 — openstaande **losse projectbestelling**. >0 = op bestellijst ongeacht `min`. Wordt 0 bij ontvangst. Iedereen mag losse bestellingen ontvangen (v2.10.1). |
| `ordered_project_id` | bigint | NULL | v2.10.0 — FK → `projects.id`. Project waarvoor de losse bestelling is; bij ontvangst kiest men per regel: direct op project (history `order` + `out`, voorraad netto 0) of magazijn (→ `staged_items`-regel). |

**Bestelladvies-formule**: `Math.ceil((target − qty) / pack) * pack` (alleen als `qty < min`) **+ `ordered_qty`**

**Op bestellijst** (`needsOrder`): `qty < min` OF `ordered_qty > 0`.

**Verouderde kolom (mag drop)**: `aliases JSONB DEFAULT '{}'` — uit een eerdere iteratie, vervangen door `factuur_naam`.

---

## `suppliers`

Leveranciers van de producten.

| Kolom | Type | Notitie |
|---|---|---|
| `id` | serial | Primary key |
| `name` | text | Leverancier naam |
| `contact` | text | Contactpersoon |
| `email` | text | Bestel-mailadres |
| `phone` | text | Telefoonnr (gebruikt voor `tel:` en WhatsApp) |

**Bekende leveranciers**: Felix Verfgroep (debiteurnr. 3004260, grootste), 
Sikkens, Storch/VVBHussan, Bock, Caparol, Repair Care, Trend 2000, ProGold.

**Standaardkortingen** (uit factuur-analyse 2026):
- Sigma 25–35%, Sikkens 20%, Repair Care 15%, ProGold 25%

---

## `categories`

| Kolom | Type | Notitie |
|---|---|---|
| `id` | serial | Primary key |
| `name` | text | Categorie naam |

(Tabel bestaat maar `cat` op products is een vrije tekstveld, dus deze tabel 
wordt mogelijk niet strikt referentieel gebruikt.)

---

## `projects`

Schilderprojecten waaraan materiaal wordt toegerekend.

| Kolom | Type | Notitie |
|---|---|---|
| `id` | serial | Primary key |
| `name` | text | Projectnaam |
| `client` | text | Klant |
| `date` | date | Startdatum |
| `status` | text | Statuslabel (lopend, klaar, etc.) |
| `active` | bool? | Of project nog actief is |

---

## `history`

Logboek van alle voorraadmutaties.

| Kolom | Type | Notitie |
|---|---|---|
| `id` | serial | Primary key |
| `type` | text | `'out'` (uitgegeven), `'in'` (retour), `'order'` (ontvangen) |
| `product_id` | int | FK → products.id |
| `project_id` | int? | FK → projects.id (NULL bij `order`) |
| `qty` | numeric | Hoeveelheid (bij `in` is dit retour-qty) |
| `note` | text | Vrije notitie ("Bestelling ontvangen", "Verbruikt op project X") |
| `date` | date | Datum van mutatie |
| `created_at` | timestamptz | Auto-timestamp van DB-insert |

**Belangrijke regels:**
- Pagineren via `dbGetAll('history', ...)` (chunks van 1000) — bij 500+ 
  records gaan oudste anders verloren.
- Project-export gebruikt 3-secties logica: hoofdlijst (netto verbruik > 0), 
  "volledig retour ontvangen" (netto = 0), en "meer retour dan uitgegeven" 
  (netto < 0, boekfout).
- Bij ontvangen wordt `last_ordered_at` op products gecleard.

---

## `staged_items` (v2.10.0 — klaarzetlijst)

Materiaal dat door een eigenaar is klaargezet om te laden voor een project. Voorraad
verandert pas bij het afvinken ("Geladen"); dan volgt `applyStockDelta` + een `history`-regel
van type `out` op het project.

| Kolom | Type | Default | Notitie |
|---|---|---|---|
| `id` | bigserial | auto | Primary key |
| `product_id` | bigint | — | FK → products.id (ON DELETE CASCADE) |
| `project_id` | bigint | — | FK → projects.id (ON DELETE CASCADE) |
| `qty` | numeric | 1 | Klaargezet aantal (bij laden aanpasbaar) |
| `location_id` | bigint | NULL | FK → buses.id; NULL = magazijn. Hieruit wordt geladen. |
| `color_code` | text | NULL | Kleur zoals gekozen bij klaarzetten |
| `note` | text | NULL | Vrije notitie ("Uit losse bestelling") |
| `status` | text | 'open' | `'open'` (op de lijst) of `'loaded'` (afgevinkt, geboekt) |
| `created_by` | text | NULL | Medewerker die klaarzette |
| `created_at` | timestamptz | now() | |
| `loaded_qty` | numeric | NULL | Werkelijk geladen aantal |
| `loaded_by` | text | NULL | Wie afvinkte |
| `loaded_at` | timestamptz | NULL | |

**App leest alleen `status=eq.open`.** Poll vergelijkt een handtekening (id:qty) van de open
regels zodat een nieuwe klaarzetregel ook zonder history-wijziging binnen 5 s op andere
apparaten verschijnt. Ontbreekt de tabel (SQL niet gedraaid), dan laadt de app door en toont
het dashboard een waarschuwing voor eigenaren.

---

## App-conventies

**ID-systeem**: alle tabellen serial integer PK's, geen UUID.

**Sync**: app polled iedere 10 sec via REST/PostgREST. `dbPost`, `dbPatch`, 
`dbDelete`, `dbGet`, `dbGetAll` zijn de helpers in de JS.

**Cache-bust**: GitHub Pages cached aggressief. Na deploy URL aanvullen 
met `?v=N` (incrementen) om iPhone te dwingen vers te laden.

**Pinkode**: 6422 (sessionStorage, opnieuw vragen na sluiten).

**Visuele stijl**: wit met accent `#e63946`, fonts Inter (UI) + 
JetBrains Mono (cijfers/codes), border-radius 10–14px.
