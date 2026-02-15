# ODIN — Guardrail: Frontend (React/TypeScript)

Version: 1.0
Stand: 2026-02-15

---

## 1. Zweck & Geltungsbereich

Dieser Guardrail definiert die verbindlichen Bauvorschriften fuer `odin-frontend` — das React/TypeScript-Dashboard von ODIN.

**Geltungsbereich:** Ausschliesslich `odin-frontend` (React 18+, TypeScript, Vite).

**Abgrenzung:**
- Kapitel 9 (`09-frontend.md`) definiert die **Architektur** (WAS gebaut wird): Komponenten, Endpoints, Kommunikationsprotokolle, Sim-Semantik
- Dieser Guardrail definiert die **Bauvorschriften** (WIE gebaut wird): Projektstruktur, Naming, TypeScript-Regeln, State Management, Test-Strategie
- Java-Backend-Module → `guardrail-module-structure.md` + `guardrail-development-process.md`
- Redundanzen mit Backend-Guardrails sind bewusst akzeptiert (Frontend-Entwickler sollen nicht Backend-Guardrails lesen muessen)

**Referenzen:**
- Kapitel 9: React-Frontend (Architektur, Endpoints, Contracts)
- Kapitel 0: Systemuebersicht (MarketClock, Ports, Kill-Switch)
- Kapitel 6: Rules Engine (Pipeline-States, FSM)

---

## 2. Projekt-Struktur (DDD / Feature-basiert)

Feature-basierte Ordnerstruktur — **kein technisches Top-Level-Layering** (nicht `components/`, `hooks/`, `utils/` auf oberster Ebene):

```
odin-frontend/
├── public/
├── src/
│   ├── features/
│   │   ├── pipeline/          # Pipeline-Monitoring pro Instrument
│   │   │   ├── components/    # PipelineStatusPanel, PipelineStateIndicator
│   │   │   ├── hooks/         # usePipelineStream, usePipelineState
│   │   │   ├── types/         # PipelineState, PipelineSnapshot
│   │   │   └── index.ts       # Public API des Features
│   │   ├── chart/             # Candlestick, Indikatoren, Entry/Exit-Marker
│   │   ├── dashboard/         # Global Dashboard, P&L, Exposure, Risk
│   │   ├── controls/          # Kill-Switch, Pause/Resume
│   │   ├── alerts/            # Alert-Banner, Sound, Acknowledge
│   │   └── history/           # Run-Auswahl, Replay, DecisionLog, LLM-Historie
│   ├── shared/
│   │   ├── api/               # SSE-Client, REST-Client, Reconnect-Logik
│   │   ├── components/        # Shared UI (Button, Badge, ConfirmDialog)
│   │   ├── hooks/             # useAuth, useEventSource
│   │   ├── types/             # Globale Types (InstrumentId, RunId, Enums)
│   │   └── utils/             # Formatter, Constants
│   ├── app/                   # App-Shell, Routing, Layout, Providers
│   └── main.tsx
├── package.json
├── tsconfig.json
├── vite.config.ts
└── vitest.config.ts
```

### Abhaengigkeitsregeln (strikt)

| Richtung | Erlaubt? |
|----------|----------|
| Feature → `shared/` | **Erlaubt** |
| Feature → `app/` | **Erlaubt** (nur fuer Routing-Constants) |
| Feature → Feature | **Verboten** (analog: Fachmodul→Fachmodul im Backend, Kap 1) |
| `shared/` → `features/` | **Verboten** |
| `app/` → `features/` | **Erlaubt** (App-Shell importiert Features fuer Routing) |
| `app/` → `shared/` | **Erlaubt** |

### Barrel-Exports

- Jeder Feature-Ordner hat ein `index.ts` als Public API (Barrel Export)
- Barrel-Exports **nur auf Feature-Ebene**, nicht in Sub-Ordnern (`components/`, `hooks/` etc.)
- Imports aus anderen Features (wenn erlaubt, z.B. `app/` → Feature) gehen immer ueber `index.ts`

---

## 3. Naming-Konventionen

| Kategorie | Pattern | Beispiel |
|-----------|---------|----------|
| Komponenten | PascalCase, `.tsx` | `PipelineStatusPanel.tsx` |
| Hooks | camelCase mit `use`-Prefix, `.ts` | `usePipelineStream.ts` |
| Types/Interfaces | PascalCase, eigene Datei, `.ts` | `PipelineSnapshot.ts` |
| Utilities | camelCase, `.ts` | `formatCurrency.ts` |
| Constants | UPPER_SNAKE_CASE im Code, camelCase-Dateiname | `pipelineStates.ts` |
| CSS Modules | `ComponentName.module.css` | `PipelineStatusPanel.module.css` |
| Test-Dateien | neben Source, `.test.tsx` / `.test.ts` | `PipelineStatusPanel.test.tsx` |

---

## 4. TypeScript-Regeln

- **`strict: true`** — kein `any` ohne Begruendung im Kommentar (`// any: <reason>`)
- **Backend-DTOs** als TypeScript-Interfaces (1:1-Abbildung des Backend-Contracts aus Kap 9 §3)
- **Union-Types** fuer endliche Mengen — kein TypeScript `enum`-Keyword (Tree-Shaking-Probleme, nicht const-enum-sicher)
  ```typescript
  // Richtig:
  type PipelineState = 'STARTUP' | 'WARMUP' | 'OBSERVING' | 'POSITIONED' | 'PENDING_FILL' | 'HALTED' | 'DAY_STOPPED';

  // Falsch:
  enum PipelineState { STARTUP, WARMUP, ... }
  ```
- **Discriminated Unions** fuer SSE-Event-Typen (basierend auf Kap 9 §3 Event-Typen)
  ```typescript
  type SseEvent =
    | { type: 'snapshot'; payload: MarketSnapshot }
    | { type: 'state-change'; payload: StateChangePayload }
    | { type: 'trade-update'; payload: TradeUpdatePayload }
    // ...
  ```
- Kein `as`-Type-Assertion ohne Runtime-Guard
- **Explizite Return-Types** fuer alle exportierten Funktionen
- **`readonly`-Modifier** fuer Props und State-Objekte
- Kein `!` (Non-Null-Assertion) ohne Begruendung im Kommentar

---

## 5. State Management

| Prinzip | Regel |
|---------|-------|
| SSE-Streams | Primaere Datenquelle → Custom Hook pro Stream (`usePipelineStream`, `useGlobalStream`) |
| React Context | Fuer globale Singletons: Auth-State, SSE-Connection-Status, Kill-Switch-State |
| Kein Redux / Zustand | In v1 nicht noetig — SSE-Hooks + Context ausreichend |
| Derived State | `useMemo` / berechnete Werte — kein redundanter State |
| Reconnect-Logik | In `shared/api/` gekapselt, nicht in Feature-Komponenten |
| Kein lokaler Cache | SSE-Daten nicht cachen — der Stream IST der State |

**Anti-Patterns:**
- `useState` fuer Daten die aus SSE kommen (→ stattdessen SSE-Hook verwenden)
- `useEffect` mit manuellem SSE-Subscription-Management in Komponenten (→ stattdessen `shared/api/` nutzen)
- Prop Drilling ueber mehr als 2 Ebenen (→ stattdessen Context oder Composition)

---

## 6. Kommunikation mit Backend

Normative Quelle fuer Endpoints und Contracts: **Kap 9 §3**.

### SSE-Client

- Zentral in `shared/api/sseClient.ts`
- Reconnect mit Exponential Backoff (1s/2s/4s/8s, max 30s — Kap 9 §3)
- `Last-Event-ID` bei Reconnect mitsenden
- Event-Parsing ueber Discriminated Unions (siehe §4)

### REST-Client

- Typisierte Wrapper pro Endpoint in `shared/api/restClient.ts`
- **Kein rohes `fetch()`** in Feature-Code
- CSRF-Token automatisch aus Cookie in REST-Mutations-Requests (POST `/api/controls/*`)

### Error-Handling

- Zentrale Error-Boundary auf App-Ebene
- Feature-lokale Fallbacks fuer nicht-kritische Fehler (z.B. Chart-Ladefehler zeigt Placeholder)
- SSE-Verbindungsabbruch → Connection-Status-Banner (nicht Error-Boundary)

### Event-Type-Katalog

Basierend auf Kap 9 §3 — alle SSE-Event-Typen als Discriminated Union in `shared/types/`:
`snapshot`, `state-change`, `trade-update`, `order-update`, `llm-update`, `alert`, `pnl`, `risk-update`

---

## 7. Komponenten-Regeln

| Regel | Details |
|-------|---------|
| Funktionale Komponenten | Keine Klassen-Komponenten |
| Props-Interface | Benanntes Interface (nicht inline), exportiert aus der Komponenten-Datei |
| Keine Business-Logik | Keine Transformations-/Berechnungslogik in Komponenten — in Hooks oder Utils |
| `React.memo` | Nur bei nachgewiesenem Re-Render-Problem (analog R11.2: erst messen, dann optimieren) |
| Key-Prop | Stabile Domain-IDs (`instrumentId`, `runId`) — keine Array-Indices |
| Event-Handler | `handle`-Prefix: `handleKillSwitch`, `handlePauseClick` |
| Eine Komponente pro Datei | Keine Multi-Component-Dateien |

---

## 8. Styling

| Regel | Details |
|-------|---------|
| CSS Modules | `.module.css` — kein globales CSS ausser Reset und CSS-Variablen |
| Design-Tokens | CSS Custom Properties (`--color-state-positioned`, `--spacing-md`) |
| State-Farbcodierung | Aus Kap 9 §4.2 als Token-Set (verbindlich): STARTUP=grau, WARMUP=gelb, OBSERVING=blau, POSITIONED=gruen, PENDING_FILL=orange, HALTED=rot, DAY_STOPPED=grau dunkel |
| Dark Theme | Einziges Theme (Trading-UI-Konvention) |
| Kein CSS-in-JS | Keine styled-components, emotion oder aehnliche Libraries |
| Desktop-only | Kein Mobile-Support in v1 |

---

## 9. Build & Tooling

| Aspekt | Regel |
|--------|-------|
| Build-Tool | **Vite** mit React SWC-Plugin |
| Maven-Integration | **`frontend-maven-plugin`** — `mvn package` baut Frontend mit (Kap 9 §6) |
| Linting | **ESLint** + **Prettier** (Pflicht, gemeinsame Config) |
| Import-Aliase | `@features/`, `@shared/`, `@app/` — in `tsconfig.json` + `vite.config.ts` |
| `console.log` | Verboten in Produktion — ESLint-Rule `no-console: error` |
| Import-Sortierung | ESLint-Plugin fuer sortierte Imports |

---

## 10. Test-Struktur

| Aspekt | Regel |
|--------|-------|
| Framework | **Vitest** + **React Testing Library** |
| Hook-Tests | `renderHook` fuer Custom Hooks (insb. SSE-Integration) |
| Kein E2E in v1 | Manuelles Testing gegen laufendes Backend |
| Test-Dateien | Neben der Source: `Component.test.tsx`, `hookName.test.ts` |
| Mocking | SSE-Streams und REST-Calls mocken — kein Backend erforderlich |
| Coverage-Fokus | Hooks, Utilities, Reconnect-Logik — nicht jede presentational Component |

---

## 11. Checkliste (neues Feature)

Bei jedem neuen Feature in `src/features/` pruefen:

- [ ] Feature-Ordner unter `src/features/` angelegt?
- [ ] `index.ts` mit Public API (Barrel Export)?
- [ ] TypeScript-Types mit Backend-Contract abgeglichen (Kap 9 §3)?
- [ ] SSE-/REST-Hook in `shared/api/` oder Feature-lokal?
- [ ] Keine Cross-Feature-Imports?
- [ ] Props-Interfaces exportiert?
- [ ] CSS Module verwendet, keine Inline-Styles?
- [ ] State-Farbcodierung aus Token-Set (§8)?
- [ ] Error-Handling (Error-Boundary oder Fallback)?
- [ ] Tests fuer Hooks und Utilities geschrieben?
- [ ] ESLint + Prettier fehlerfrei?
- [ ] `npm run build` erfolgreich?
