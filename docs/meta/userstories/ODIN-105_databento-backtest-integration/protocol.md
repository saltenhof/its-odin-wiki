# Protokoll: ODIN-105 — Databento Backtest Integration (harte L0/L1-Weiche)

## Working State
- [x] YahooHistoricalDataAdapter implementiert (odin-backtest, Adapter-Pattern)
- [x] DataDownloadService refactored (harte L0/L1-Weiche, kein Fallback)
- [x] DataWiringConfig implementiert (odin-app, @ConditionalOnProperty L1_QUOTES)
- [x] Unit-Tests geschrieben (YahooHistoricalDataAdapterTest: 12 Tests, DataDownloadServiceProviderTest: 11 Tests — alle bestanden)
- [x] Integrationstests geschrieben (DataDownloadServiceIntegrationTest: 9 Tests, alle bestanden via Failsafe)
- [x] ChatGPT-Sparring fuer Edge Cases und Architektur-Validierung
- [x] Gemini-Review Dimension 1 (Code-Review)
- [x] Gemini-Review Dimension 2 (Konzepttreue)
- [x] Gemini-Review Dimension 3 (Praxis-Review)
- [x] Review-Findings bewertet (Integer-Overflow als Minor dokumentiert, sequenzielle L1-Loops als FUTURE-WORK)
- [x] Commit & Push

## Design-Entscheidungen

### 1. YahooHistoricalDataAdapter als @Component + @Qualifier("yahooProvider")
YahooDataFetcher (bereits @Component) liefert List<Bar>. Der neue Adapter bridgt ihn in das HistoricalDataProvider-Interface,
das der refactored DataDownloadService erwartet. Beide Provider-Beans werden mit unterschiedlichen Qualifiern registriert
("yahooProvider" / "databentoProvider") um NoUniqueBeanDefinitionException zu vermeiden.

### 2. DataDownloadService injiziert databentoProvider als ObjectProvider
Der Databento-Provider ist optional: Er existiert nur wenn odin.data.market-data-level=L1_QUOTES konfiguriert ist.
ObjectProvider<HistoricalDataProvider> erlaubt einen graceful getIfAvailable()-Aufruf, der null zurueckgibt statt
einen BeanCreationException-Startup-Fehler zu verursachen. Das fail-fast findet dann explizit in requireDatabentoProvider()
statt — mit einer Fehlermeldung die das Problem erklaert.

### 3. Fail-Fast in requireDatabentoProvider() statt @Autowired required=true
Wenn L1_QUOTES konfiguriert aber kein databentoProvider-Bean vorhanden ist (z. B. DATABENTO_API_KEY fehlt),
wirft requireDatabentoProvider() eine IllegalStateException mit: Level-Name, DATABENTO_API_KEY-Hinweis,
und explizitem "no fallback permitted"-Statement. Dies ist praeziser als die generische Spring-Fehlermeldung.

### 4. DataWiringConfig in odin-app mit @ConditionalOnProperty
DatabentoBatchService ist ein POJO (kein @Service gemaess modul-structure.md). Er wird manuell in odin-app verdrahtet.
Die @ConditionalOnProperty-Annotation sorgt dafuer, dass DabentoProperties (@NotBlank apiKey) nur bei L1_QUOTES
validiert wird — L0-Deployments ohne DATABENTO_API_KEY starten fehlerfrei.

### 5. @ComponentScan in DataWiringConfig fuer Databento-Repositories
QuoteTickJdbcRepository, TradeTickJdbcRepository und DataIngestLogJdbcRepository sind @Repository-annotiert
in de.its.odin.data.databento.repository, aber nicht in den Standard-Component-Scans von OdinApplication
oder BacktestWiringConfig erfasst. DataWiringConfig erhaelt @ComponentScan fuer dieses Package.

### 6. DatabentoCsvParser mit properties.defaultDataset()
DatabentoCsvParser hat keinen No-Arg-Konstruktor — er benoetigt einen String datasetId. Der @Bean-Factory-Methode
in DataWiringConfig wird DabentoProperties injiziert und properties.defaultDataset() weitergereicht.

### 7. L1 endDate wird als exklusive Grenze uebergeben (endDate.plusDays(1))
HistoricalDataProvider.downloadQuotes() erwartet per Konvention ein exklusives Enddatum (Databento-Konvention).
DataDownloadService.ensureDataAvailable() nimmt ein inklusives Enddatum (externes Interface). Die Konversion
findet in ensureL1DataAvailable() statt mit endDate.plusDays(1). Diese Stelle ist explizit dokumentiert.

### 8. L0-Pfad unveraendert
Der bestehende Yahoo-Datenpfad (findMissingDays -> yahooDataFetcher.fetch -> insertBars) bleibt
unveraendert. Die Weiche greift ausschliesslich in ensureDataAvailable() — L0-Code-Pfade wurden nicht angefasst.

## Offene Punkte

### Integer-Overflow bei L1-Tick-Count (MINOR / FUTURE-WORK)
Gemini hat korrekt identifiziert, dass ensureDataAvailable() einen int zurueckgibt, waehrend totalTicks
intern als long gezaehlt wird. Das Math.min(totalTicks, Integer.MAX_VALUE) Cap schneidet bei grossen
Universa (>2,14 Mrd Ticks) still ab. Fuer den aktuellen Use Case (Einzelsymbol-Backtests, max. 100k Ticks/Tag)
ist das kein praktisches Problem. FUTURE-WORK: Rueckgabetyp auf long umstellen wenn Multi-Symbol-Portfolios relevant werden.

### Sequenzielle L1-Requests (MINOR / FUTURE-WORK)
Databento-Downloads erfolgen sequenziell (for-Schleife ueber Symbole). Fuer grosse Symbollisten (> 20)
waere paralleles Fetching sinnvoll. Da Databento kein echtes Multi-Symbol-Batch-API hat, waere dies
per CompletableFuture oder ExecutorService umzusetzen. FUTURE-WORK fuer Multi-Symbol-Backtests.

### Cost-Limit-Pruefung nicht in DataDownloadService
estimateCost() wird in dieser Story nicht in DataDownloadService integriert — der Databento-Provider
(DatabentoBatchService) prueft das Cost-Limit intern pro Tag. DataDownloadService delegiert vollstaendig.
Dies ist korrekt per Design: die Provider-Verantwortung bleibt beim Provider.

## ChatGPT-Sparring

### Session
ChatGPT wurde mit Klassen, Akzeptanzkriterien und den geplanten Tests zur Bewertung von Architektur-Entscheidungen
und Edge Cases konsultiert.

**Wichtigste Findings und Entscheidungen:**

**1. Idempotenz bei Level-Wechsel (L0 → L1):**
ChatGPT bestaetigt: L0-Daten in odin.intraday_bar und L1-Daten in odin.quote_tick/odin.trade_tick sind
vollstaendig separate Tabellen mit separaten Idempotenz-Checks. Ein L0-zu-L1-Wechsel fuer dieselben Symbole
und Daten fuehrt zu einem neuen L1-Download — korrekt, weil es andere Tabellen sind.

**2. ObjectProvider vs. @Autowired(required=false):**
ChatGPT empfiehlt ObjectProvider als richtigen Spring-Mechanismus fuer optionale Beans, weil er typsicher
und explizit ist. @Autowired(required=false) haette denselben Effekt, ist aber weniger idiomatisch fuer
conditionale Beans. Entscheidung: ObjectProvider beibehalten.

**3. Fail-Fast-Reasoning:**
ChatGPT bestaetigt: "no fallback" ist die einzig korrekte Entscheidung. Ein stiller Yahoo-Fallback bei L1_QUOTES
wuerde die Backtest-Ergebnisse verfaelschen (keine Spreads, keine Bid/Ask-Simulation). Die IllegalStateException
mit erklaerungsreicher Meldung ist vorbildlich.

**4. @Qualifier-Pattern bei Spring:**
ChatGPT bestaetigt: @Qualifier("databentoProvider") am @Bean zusammen mit @Qualifier am Konstruktorparameter
ist der empfohlene Spring-Ansatz. Alternativ koennte @Bean(name="databentoProvider") verwendet werden —
beide Varianten sind korrekt, die gewaaehlte Variante ist expliziter.

**Bewertete und verworfene Vorschlaege:**
- Cost-Estimate-Pruefung in DataDownloadService vor dem Download: Nicht umgesetzt, da DatabentoBatchService
  das Limit intern per Tag prueft. Doppel-Pruefung wuerde Service-Verantwortung verwirren.
- @Primary statt @Qualifier: Abgelehnt — @Primary wuerde einen Provider zum Default machen, was bei
  zwei gleichwertigen Providern (je nach Konfiguration) semantisch falsch waere.

## Gemini-Review

### Dimension 1: Code-Review (Korrektheit, SOLID, Codequality)

**Findings:**

1. **KRITISCH (laut Gemini): Integer-Overflow bei L1-Tick-Count**
   `DataDownloadService.ensureDataAvailable()` gibt `int` zurueck, zaehlt intern aber als `long`.
   `Math.min(totalTicks, Integer.MAX_VALUE)` schneidet bei > 2,14 Mrd Ticks still ab.
   **Bewertung**: Als MINOR klassifiziert. Im aktuellen Use Case (max. 100k Ticks/Tag/Symbol) nicht
   relevant. FUTURE-WORK dokumentiert. Der Methodenvertrag ist unveraendert (int return type per Interface).

2. **WICHTIG: Null-Safety via Objects.requireNonNull (positiv)**
   Explizite Null-Checks in YahooHistoricalDataAdapter-Methoden mit aussagekraeftigen Meldungen.
   Bestaetigt als korrekte Praxis. Keine Aktion noetig.

3. **MINOR: @Qualifier-Pattern korrekt und konsistent**
   Gemini bestaetigt die Qualifier-Strategie als fehlerfrei. Keine Aktion noetig.

4. **Tests: Testabdeckung exzellent**
   Trennung Unit-Test / IntegrationTest als sinnvoll bewertet. Keine Aktion noetig.

### Dimension 2: Konzepttreue

**Alle Kernentscheidungen als MATCH bewertet:**

- Harte Weiche ohne Fallback: MATCH — requireDatabentoProvider() wirft IllegalStateException
- ObjectProvider fuer optionale Bean: MATCH — bester Spring-Mechanismus
- @ConditionalOnProperty(L1_QUOTES): MATCH — verhindert DabentoProperties-Validierung fuer L0
- Getrennte Idempotenz (L0: intraday_bar, L1: quote_tick/trade_tick): MATCH — vollstaendig getrennt
- UnsupportedOperationException fuer downloadQuotes/downloadTrades in Yahoo-Adapter: MATCH — korrekte Semantik

### Dimension 3: Praxis-Review

**Findings:**

1. **WICHTIG: Sequenzielle L1-Requests**
   For-Schleife ueber Symbole macht sequenzielle Databento-Calls. Fuer grosse Universa ineffizient.
   **Bewertung**: FUTURE-WORK. Databento hat kein echtes Multi-Symbol-Batch-API. Fuer aktuelle
   Single-Symbol-Backtests kein Problem.

2. **WICHTIG: endDate.plusDays(1) Timezone-Risiko**
   Gemini warnt: Wenn Databento das exklusive Datum in UTC interpretiert, koennten bei Zeitzonengrenzen
   Pre-Market-Ticks des Folgetages inkludiert werden.
   **Bewertung**: Nicht KRITISCH fuer aktuellen Use Case. Databento interpretiert YYYY-MM-DD relativ
   zum Dataset (XNAS.ITCH = US Eastern Time). Die endDate.plusDays(1)-Stelle ist explizit im JavaDoc
   dokumentiert. Timezone-Pruefung bei erster L1-Backtest-Ausfuehrung vorgesehen.
