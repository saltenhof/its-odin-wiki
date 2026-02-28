# ODIN-082c: Bandit-Persistierung und Reward-Anbindung

**Modul:** odin-brain, odin-persistence
**Phase:** 1
**Abhaengigkeiten:** ODIN-082b (Bandit-Arbiter-Integration — liefert `ThompsonSamplingBandit.getDistribution()`/`setDistribution()`), ODIN-071 (Post-Trade Counterfactual — liefert `CounterfactualResult` mit `quantOnlyProfitable` und `llmInfluenceProfitable`)
**Geschaetzter Umfang:** M

---

## Kontext

ODIN-082 (Adaptive Weighting via Bandit) wird in 3 Sub-Stories aufgeteilt. Diese dritte und letzte Sub-Story schliesst die Implementierung ab: Die Beta-Parameter des `ThompsonSamplingBandit` werden nach jedem Update in der Datenbank persistiert (ueberleben Neustarts) und beim Start wiederhergestellt. Ausserdem wird die Verbindung zum Post-Trade-Counterfactual (ODIN-071) hergestellt: Nach jedem Trade-Close wird `updateReward()` mit dem echten Outcome aufgerufen.

## Scope

**In Scope:**

- Flyway-Migration V030: Neue Tabelle `bandit_state`
- JPA-Entity `BanditStateEntity` in `de.its.odin.brain.persistence`
- JPA-Repository `BanditStateRepository` in `de.its.odin.brain.persistence`
- `BanditStateService` in `de.its.odin.brain.arbiter`: Laedt beim Startup alle `BanditStateEntity`-Eintraege und ruft `ThompsonSamplingBandit.setDistribution()` auf; persistiert nach `updateReward()` den neuen Zustand
- Integration in den Post-Trade-Feedback-Pfad: Nach Trade-Close wird `updateReward(regime, arm, profitable)` aufgerufen — basierend auf `CounterfactualResult`
- Startup-Reconciliation: Falls keine gespeicherten Parameter gefunden werden → Prior-Initialisierung (bereits im `ThompsonSamplingBandit`)
- DB-Tests: Embedded Postgres (Zonky), Flyway-Migration, Repository-Methoden

**Out of Scope:**

- Reset-API fuer Beta-Parameter (separate Story, manuell via Admin)
- Online-Visualisierung der Verteilungen im Frontend (separate Story)
- Discount-Faktor / Windowed Counts (Non-Stationaritaet)
- `ThompsonSamplingBandit` Kern (ODIN-082a) und Arbiter-Integration (ODIN-082b)

## Akzeptanzkriterien

### Flyway-Migration (V030)

- [ ] Migration `V030__create_bandit_state.sql` in `odin-persistence/src/main/resources/db/migration/`
- [ ] DDL:
  ```sql
  CREATE TABLE odin.bandit_state (
      id           BIGSERIAL PRIMARY KEY,
      regime       VARCHAR(30) NOT NULL,
      arm          VARCHAR(10) NOT NULL,
      alpha        DOUBLE PRECISION NOT NULL,
      beta         DOUBLE PRECISION NOT NULL,
      trade_count  INTEGER NOT NULL DEFAULT 0,
      last_updated TIMESTAMP WITH TIME ZONE NOT NULL,
      UNIQUE (regime, arm)
  );
  ```
- [ ] Schema-Prefix `odin.` konsistent mit allen anderen Tabellen (vgl. `odin.llm_call_record`)

### BanditStateEntity

- [ ] Neue Klasse `BanditStateEntity` in `de.its.odin.brain.persistence`
- [ ] Felder: `Long id` (BIGSERIAL PK), `String regime` (VARCHAR 30), `String arm` (VARCHAR 10), `double alpha`, `double beta`, `int tradeCount`, `Instant lastUpdated`
- [ ] `@Entity @Table(name = "bandit_state", schema = "odin")`
- [ ] Protected no-arg Konstruktor fuer JPA
- [ ] Public Konstruktor fuer alle Felder (ohne id)
- [ ] Getter fuer alle Felder (keine Setter — Update via `@Modifying`-Query)
- [ ] `equals()`/`hashCode()` auf Basis von `id` (analog zu `LlmCallRecordEntity`)
- [ ] JavaDoc vollstaendig

### BanditStateRepository

- [ ] Neues Interface `BanditStateRepository` in `de.its.odin.brain.persistence`
- [ ] Extends `JpaRepository<BanditStateEntity, Long>`
- [ ] Methode `Optional<BanditStateEntity> findByRegimeAndArm(String regime, String arm)`
- [ ] Methode `List<BanditStateEntity> findAll()` — zum Laden aller Eintraege beim Startup (geerbt von `JpaRepository`)
- [ ] `@Modifying @Query`-Methode `upsertState(String regime, String arm, double alpha, double beta, int tradeCount, Instant lastUpdated)` — nutzt PostgreSQL `INSERT ... ON CONFLICT (regime, arm) DO UPDATE SET alpha=..., beta=..., trade_count=..., last_updated=...`
- [ ] JavaDoc vollstaendig

### BanditStateService

- [ ] Neue Klasse `BanditStateService` in `de.its.odin.brain.arbiter`
- [ ] Konstruktor: `BanditStateService(BanditStateRepository repository, ThompsonSamplingBandit bandit, MarketClock clock)`
- [ ] Methode `void loadState()`: Laedt alle `BanditStateEntity`-Eintraege und ruft fuer jeden `bandit.setDistribution(Regime.valueOf(entity.getRegime()), BanditArm.valueOf(entity.getArm()), new BetaDistribution(entity.getAlpha(), entity.getBeta()))` auf. Logt Anzahl geladener Eintraege.
- [ ] Methode `void persistUpdate(Regime regime, BanditArm arm, BetaDistribution distribution, int tradeCount)`: Ruft `repository.upsertState(...)` mit aktuellen alpha/beta-Werten auf. Nutzt `clock.now()` fuer `lastUpdated`.
- [ ] `loadState()` wird beim Startup aufgerufen — Zeitpunkt: nach Startup-Reconciliation, vor erstem Decision-Zyklus. Integration via `PipelineFactory` oder `LifecycleManager` (Implementierer entscheidet basierend auf dem bestehenden Startup-Flow und dokumentiert in `protocol.md`)
- [ ] JavaDoc vollstaendig

### Reward-Anbindung an Post-Trade-Counterfactual (ODIN-071)

- [ ] Nach Trade-Close: `ThompsonSamplingBandit.updateReward(regime, BanditArm.QUANT, quantOnlyProfitable)` aufgerufen
- [ ] Nach Trade-Close: `ThompsonSamplingBandit.updateReward(regime, BanditArm.LLM, llmInfluenceProfitable)` aufgerufen
- [ ] `regime` kommt aus dem `TradeRecord` oder dem `CounterfactualResult` (Regime das beim Entry aktiv war)
- [ ] Nach jedem `updateReward()`-Aufruf: `banditStateService.persistUpdate(regime, arm, bandit.getDistribution(regime, arm), bandit.getTotalTradeCount())` aufgerufen
- [ ] Der Aufrufpunkt liegt nach Trade-Close-Verarbeitung — Implementierer prueft den bestehenden Counterfactual-Service (ODIN-071) und integriert dort
- [ ] **Fallback:** Wenn `CounterfactualResult` nicht verfuegbar (ODIN-071 nicht aktiv, Fehler), wird `updateReward()` NICHT aufgerufen — Bandit bleibt auf Prior. Kein Crash, kein silent ignore ohne Log.

### Startup-Wiederherstellung

- [ ] Beim Start: `loadState()` wird aufgerufen
- [ ] Wenn keine Eintraege in DB: `ThompsonSamplingBandit` startet mit Prior (bereits implizit durch Lazy-Init in `ThompsonSamplingBandit`)
- [ ] Wenn Eintraege vorhanden: Beta-Parameter werden korrekt wiederhergestellt
- [ ] Logeintrag bei Startup: Anzahl wiederhergestellter (Regime, Arm)-Paare

### DB-Tests

- [ ] `BanditStateRepositoryIntegrationTest`:
  - Speichert eine `BanditStateEntity` → laedt sie via `findByRegimeAndArm()` → Werte stimmen ueberein
  - `upsertState()` fuert bei Konflikt (gleiches regime/arm) ein UPDATE aus, kein Duplicate-Error
  - `findAll()` gibt alle gespeicherten Eintraege zurueck
  - Flyway-Migration V030 wird im Test ausgefuehrt (Zonky embedded Postgres)
- [ ] `BanditStateServiceIntegrationTest`:
  - `persistUpdate()` → `loadState()` auf neuer Instanz → `bandit.getDistribution()` hat die gespeicherten Werte
  - Round-Trip: alpha=7.0, beta=4.0 werden gespeichert und korrekt wiederhergestellt

## Technische Details

### Neue Klassen

| Klasse | Package | Beschreibung |
|--------|---------|-------------|
| `BanditStateEntity` | `de.its.odin.brain.persistence` | JPA Entity fuer Beta-Parameter-Persistierung |
| `BanditStateRepository` | `de.its.odin.brain.persistence` | JPA Repository mit Upsert-Methode |
| `BanditStateService` | `de.its.odin.brain.arbiter` | Vermittler zwischen `ThompsonSamplingBandit` und DB |

### Flyway-Migration

Datei: `V030__create_bandit_state.sql` in `odin-persistence/src/main/resources/db/migration/`

Naechste freie Versionsnummer ist V030 (V029 ist `allow_3m_bar_interval`).

### Upsert-Query (PostgreSQL-spezifisch)

```sql
INSERT INTO odin.bandit_state (regime, arm, alpha, beta, trade_count, last_updated)
VALUES (:regime, :arm, :alpha, :beta, :tradeCount, :lastUpdated)
ON CONFLICT (regime, arm)
DO UPDATE SET
    alpha        = EXCLUDED.alpha,
    beta         = EXCLUDED.beta,
    trade_count  = EXCLUDED.trade_count,
    last_updated = EXCLUDED.last_updated
```

Als `@Modifying @Query` in `BanditStateRepository` mit `nativeQuery = true`.

### Anbindung an ODIN-071 Counterfactual

Der Post-Trade-Counterfactual (ODIN-071) erzeugt ein `CounterfactualResult` nach Trade-Close. Dieses enthaelt:
- `quantOnlyProfitable: boolean` — War der Trade profitabel wenn nur Quant-Signale beachtet werden?
- `llmInfluenceProfitable: boolean` — Hat der LLM-Einfluss zum Profit beigetragen?
- `regime: Regime` — Regime das beim Entry aktiv war

Der Implementierer liest den bestehenden Counterfactual-Service (`odin-brain`, ODIN-071) und findet den Aufrufpunkt nach Trade-Close. Dort wird `updateReward()` + `persistUpdate()` eingehaengt.

### Transaktionsgrenzen

`BanditStateService.persistUpdate()` wird ausserhalb des normalen Decision-Cycle-Pfads aufgerufen (nach Trade-Close, asynchron zum Decision-Loop). `TransactionTemplate` verwenden statt `@Transactional` (analog zu anderen Pro-Pipeline-POJOs in odin-core/brain). Der `BanditStateService` ist kein Spring-Bean — er nutzt `TransactionTemplate` das ihm im Konstruktor uebergeben wird.

### MarketClock fuer lastUpdated

`BanditStateService` bekommt `MarketClock` als Konstruktorparameter. `lastUpdated` wird via `clock.now()` gesetzt. **Kein `Instant.now()` im Trading-Codepfad** (Coding-Regel aus CLAUDE.md).

## Konzept-Referenzen

- `theme-backlog.md` Thema 22: "Adaptive Gewichtung via Bandit-Algorithmen (Thompson Sampling)" — Persistierung, Reward-Signal-Quelle
- `theme-backlog.md` Thema 8: "Post-Trade Counterfactual" — `CounterfactualResult`-Struktur, Reward-Signal-Definition
- `docs/backend/architecture/08-data-model.md` — Flyway-Konventionen, Schema-Namensraum `odin`, Entity-Muster
- `docs/backend/architecture/05-llm-integration.md` — Zwei Schreibpfade, DDD-Persistenz in Domain-Modul

## Guardrail-Referenzen

- `docs/backend/guardrails/module-structure.md` — Entities in Domain-Modul (`odin-brain`), nicht in `odin-persistence`; DDD-Persistenz-Regel
- `docs/backend/guardrails/development-process.md` — Phasen, Checkpoints, Review-Pflicht
- `CLAUDE.md` — Coding-Regeln: `MarketClock` statt `Instant.now()`, `TransactionTemplate` statt `@Transactional` fuer Pro-Pipeline-POJOs, Flyway fuer alle Migrationen

## Definition of Done

### 2.1 Code-Qualitaet
- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-brain,odin-persistence`)
- [ ] Kein `var` — explizite Typen
- [ ] Keine Magic Numbers — `private static final` Konstanten
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen (kompakt aber vollstaendig)
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + JavaDoc). Kommentare: Englisch
- [ ] Namespace-Konvention: `odin.brain.*` fuer Konfiguration
- [ ] `MarketClock` statt `Instant.now()` — keine Violation im Trading-Codepfad
- [ ] `TransactionTemplate` statt `@Transactional` fuer `BanditStateService` (POJO, kein Spring-Bean)

### 2.2 Tests — Klassenebene (Unit-Tests)
- [ ] Unit-Tests fuer `BanditStateService`: `loadState()` ruft `setDistribution()` fuer jeden DB-Eintrag auf (gemocktes Repository)
- [ ] Unit-Tests fuer Reward-Anbindung: `updateReward()` + `persistUpdate()` werden nach Trade-Close korrekt aufgerufen
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)

### 2.3 Tests — Komponentenebene (Integrationstests)
- [ ] `BanditStateServiceIntegrationTest`: Round-Trip persist → load mit echten Klassen (kein Mocking ausser DB)
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)
- [ ] Mindestens 1 Integrationstest der den vollstaendigen Persistence-Round-Trip abdeckt

### 2.4 Tests — Datenbank
- [ ] Embedded-Postgres-Tests mit Zonky (`io.zonky.test:embedded-postgres`)
- [ ] Flyway-Migration V030 wird im Test ausgefuehrt
- [ ] `BanditStateRepository`-Methoden gegen echte DB getestet (findByRegimeAndArm, upsertState, findAll)
- [ ] Upsert-Verhalten verifiziert (kein Duplicate-Key-Error bei zweitem Aufruf mit gleichem regime/arm)
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)

### 2.5 Test-Sparring mit ChatGPT
- [ ] ChatGPT-Session gestartet mit: `BanditStateEntity.java`, `BanditStateRepository.java`, `BanditStateService.java`, Test-Klassen, Akzeptanzkriterien
- [ ] ChatGPT nach Grenzfaellen gefragt: Was passiert wenn `loadState()` DB-Fehler wirft? Was wenn `CounterfactualResult` null ist? Was wenn Reward-Update und Startup-Load gleichzeitig aufgerufen werden? Was wenn alpha/beta in DB korrumpiert sind (alpha < 0)?
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis im `protocol.md` dokumentiert

### 2.6 Review durch Gemini — Drei Dimensionen
- [ ] Dimension 1: Code-Review — Null-Safety in `loadState()`, Upsert-Query korrekt (PostgreSQL ON CONFLICT Syntax), `TransactionTemplate` korrekt verwendet, `clock.now()` statt `Instant.now()`?
- [ ] Dimension 2: Konzepttreue-Review — Entspricht die Reward-Anbindung dem `CounterfactualResult`-Kontrakt aus ODIN-071? Stimmt die Update-Regel (alpha+1 fuer profitable, beta+1 fuer unprofitable) mit dem Konzept ueberein?
- [ ] Dimension 3: Praxis-Review — Was passiert bei Neustart mitten in einem Trade? Reward-Update koennte verloren gehen — ist das akzeptabel? Wie robust ist der `loadState()`-Aufruf wenn die DB nicht erreichbar ist?
- [ ] Findings bewertet und berechtigte Findings behoben
- [ ] Ergebnis im `protocol.md` dokumentiert

### 2.7 Protokolldatei
- [ ] `protocol.md` im Story-Verzeichnis vorhanden
- [ ] Alle Pflichtabschnitte enthalten: Working State, Design-Entscheidungen, Offene Punkte, ChatGPT-Sparring, Gemini-Review
- [ ] Wird live waehrend der Arbeit aktualisiert

### 2.8 Abschluss
- [ ] Commit mit aussagekraeftiger Message
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

## Notizen fuer den Implementierer

### Aufrufpunkt fuer Startup-loadState()

Der Startup-Flow in ODIN laeuft ueber `LifecycleManager` (odin-core). Nach der Startup-Reconciliation mit dem Broker (GTC-Stop-Check) und vor dem ersten Decision-Zyklus. Pruefe `LifecycleManager` und `PipelineFactory` ob ein geeigneter Hook existiert (z.B. `afterPropertiesSet()` oder ein dedizierter `startup()`-Aufruf). Dokumentiere die Entscheidung in `protocol.md`.

### Robust gegen Fehler bei loadState()

Wenn `loadState()` einen DB-Fehler wirft, soll der System-Start NICHT abgebrochen werden. Stattdessen: Log-Warning, Bandit startet mit Prior. Die Entscheidung ist bewusst — ein fehlender Bandit-State ist kein kritischer Fehler (System faellt auf statische Gewichte zurueck wegen Fallback in ODIN-082b).

### Upsert vs. Delete+Insert

Upsert (ON CONFLICT DO UPDATE) ist die richtige Wahl hier: Nur 10 Eintraege maximal (5 Regimes x 2 Arme). Kein Risiko fuer Table-Bloat. Kein Loeschen von Daten beim Update.

### Reward-Verzoegerung (Reward Delay)

Das Reward-Signal kommt nach Trade-Close — das kann Stunden nach der Entry-Decision sein. Das ist konzeptionell korrekt: Der Bandit lernt aus echten Trade-Outcomes, nicht aus intraday-Schaetzungen. Der Update-Zyklus ist langsam (ein Update pro Trade), aber dafuer qualitativ hochwertig.

### Achtung: TradeCount in BanditStateEntity

`trade_count` in der DB ist die kumulierte Anzahl Updates fuer diesen (Regime, Arm)-Eintrag — nicht identisch mit `ThompsonSamplingBandit.getTotalTradeCount()` (der zaehlt alle Updates summaert). Der DB-Eintrag speichert den Paar-spezifischen Count. Das ist fuer zukuenftige Diagnose-Queries nuetzlich ("Wie viele TREND_UP-QUANT Trades haben wir gesehen?").
