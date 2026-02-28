# API-Client Design — Databento-Anbindung für ODIN

> Status: Konzept | Erstellt: 2026-02-28

## 1. Client-Architektur

### 1.1 Einordnung im Backend

Der Databento-Client wird ein neues **Maven-Modul `odin-data`** als Service implementiert,
da er die Datenbeschaffung (nicht Broker, nicht Brain, nicht Backtest-Logik) betrifft.
Alternativ kann er in `odin-backtest` angesiedelt werden, da der primäre Use Case
das Laden historischer Daten für Backtests ist.

**Empfehlung:** Neuer Service in `odin-data`, da die Datenbeschaffung auch für
Live-Systeme relevant werden könnte (Databento Live API als Datenquelle).

### 1.2 HTTP-Stack: Retrofit2 + OkHttp3

Standard-Stack im ODIN-Projekt. Vorteile gegenüber `java.net.http.HttpClient`:

- **Deklaratives API-Interface** — Endpoint-Definitionen als Java-Interface mit Annotationen
- **Interceptor-Chain** (OkHttp) — Auth, Logging, Retry als saubere Middleware
- **Streaming-Support** — `@Streaming` für große CSV-Downloads ohne Memory-Alloc
- **Retry/Timeout** zentral konfigurierbar auf OkHttpClient-Ebene
- **Bewährter Stack** — konsistent mit dem Rest des Projekts

### 1.3 Schichtenmodell

```
┌─────────────────────────────────────────────────┐
│              DatabentoBatchService               │
│  Orchestriert den Download-Workflow:             │
│  Kostenschätzung → Download → Parse → Persist    │
└────────────────────┬────────────────────────────┘
                     │ verwendet
        ┌────────────┼────────────┐
        ▼            ▼            ▼
┌──────────────┐ ┌──────────┐ ┌─────────────────┐
│ DabentoApi   │ │ Csv/Dbn  │ │TickJdbcPersister│
│ (Retrofit2   │ │ Parser   │ │ (quote_tick,     │
│  Interface)  │ │          │ │  trade_tick,     │
│ + OkHttp3    │ │          │ │  intraday_bar)   │
└──────────────┘ └──────────┘ └─────────────────┘
```

### 1.4 Module-Abhängigkeiten

```
odin-data (bestehend, wird erweitert)
  ├── DabentoApi                 → Retrofit2 Interface
  ├── DabentoApiFactory          → OkHttp3 + Retrofit2 Setup
  ├── DatabentoCsvParser         → Standard CSV
  ├── DatabentoBatchService      → Orchestrierung
  └── DabentoProperties          → @ConfigurationProperties

odin-backtest (Consumer)
  └── verwendet DatabentoBatchService zum Laden von Backtest-Daten

odin-app (REST API, optional)
  └── DatabentoController        → Manuelles Triggern von Downloads via API
```

### 1.5 Maven-Dependencies (in odin-data pom.xml)

```xml
<!-- Retrofit2 -->
<dependency>
    <groupId>com.squareup.retrofit2</groupId>
    <artifactId>retrofit</artifactId>
    <version>2.11.0</version>
</dependency>
<dependency>
    <groupId>com.squareup.retrofit2</groupId>
    <artifactId>converter-jackson</artifactId>
    <version>2.11.0</version>
</dependency>

<!-- OkHttp3 -->
<dependency>
    <groupId>com.squareup.okhttp3</groupId>
    <artifactId>okhttp</artifactId>
    <version>4.12.0</version>
</dependency>
<dependency>
    <groupId>com.squareup.okhttp3</groupId>
    <artifactId>logging-interceptor</artifactId>
    <version>4.12.0</version>
</dependency>
```

---

## 2. HTTP-Client Design

### 2.1 Retrofit2 Interface: `DabentoApi`

Deklaratives API-Interface — jeder Databento-Endpoint wird als Methode mit Annotationen definiert.

```java
/**
 * Databento Historical REST API.
 * Alle Endpunkte unter https://hist.databento.com/v0/.
 */
public interface DabentoApi {

    /**
     * Schätzt die Kosten einer Datenanfrage in USD.
     * Identische Parameter wie getRange, aber ohne Datendownload.
     */
    @GET("metadata.get_cost")
    Call<ResponseBody> getCost(
        @Query("dataset")   String dataset,
        @Query("symbols")   String symbols,
        @Query("schema")    String schema,
        @Query("start")     String start,
        @Query("end")       String end,
        @Query("stype_in")  String stypeIn,
        @Query("encoding")  String encoding
    );

    /**
     * Streaming-Download historischer Marktdaten.
     * @Streaming verhindert, dass OkHttp den gesamten Body in den RAM lädt.
     * Response wird zeilenweise als CSV geparst.
     */
    @Streaming
    @GET("timeseries.get_range")
    Call<ResponseBody> getRange(
        @Query("dataset")     String dataset,
        @Query("symbols")     String symbols,
        @Query("schema")      String schema,
        @Query("start")       String start,
        @Query("end")         String end,
        @Query("stype_in")    String stypeIn,
        @Query("encoding")    String encoding,
        @Query("compression") String compression
    );

    /** Batch-Job einreichen (für größere Anfragen). */
    @FormUrlEncoded
    @POST("batch.submit_job")
    Call<ResponseBody> submitBatchJob(
        @Field("dataset")   String dataset,
        @Field("symbols")   String symbols,
        @Field("schema")    String schema,
        @Field("start")     String start,
        @Field("end")       String end,
        @Field("stype_in")  String stypeIn,
        @Field("encoding")  String encoding
    );

    /** Status aller Batch-Jobs abrufen. */
    @GET("batch.list_jobs")
    Call<ResponseBody> listBatchJobs();

    /** Batch-Job-Dateien herunterladen. */
    @Streaming
    @GET("batch.download")
    Call<ResponseBody> downloadBatchJob(
        @Query("job_id") String jobId
    );
}
```

### 2.2 OkHttp3-Setup: `DabentoApiFactory`

Zentrale Factory für den konfigurierten Retrofit-Client.

```java
@Component
public class DabentoApiFactory {

    private final DabentoProperties properties;

    /**
     * Erstellt eine konfigurierte DabentoApi-Instanz.
     * OkHttp3 übernimmt Auth, Logging, Timeouts und Retry.
     */
    public DabentoApi create() {
        OkHttpClient httpClient = new OkHttpClient.Builder()
            // Basic Auth: API-Key als Username, leeres Passwort
            .addInterceptor(new DabentoAuthInterceptor(properties.apiKey()))
            // Request/Response Logging (nur Headers im Prod, Body im Debug)
            .addInterceptor(new HttpLoggingInterceptor()
                .setLevel(HttpLoggingInterceptor.Level.HEADERS))
            // Timeouts: großzügig für Streaming-Downloads
            .connectTimeout(Duration.ofSeconds(30))
            .readTimeout(Duration.ofSeconds(properties.timeoutSeconds()))
            .writeTimeout(Duration.ofSeconds(30))
            // Kein automatischer Retry auf OkHttp-Ebene
            // (wir steuern Retries selbst im DatabentoBatchService)
            .retryOnConnectionFailure(false)
            .build();

        Retrofit retrofit = new Retrofit.Builder()
            .baseUrl(properties.baseUrl())
            .client(httpClient)
            .addConverterFactory(JacksonConverterFactory.create())
            .build();

        return retrofit.create(DabentoApi.class);
    }
}
```

### 2.3 Auth-Interceptor

```java
/**
 * OkHttp Interceptor für Databento Basic Auth.
 * Fügt den Authorization-Header zu jedem Request hinzu.
 */
public class DabentoAuthInterceptor implements Interceptor {

    private final String credentials;

    public DabentoAuthInterceptor(String apiKey) {
        this.credentials = Credentials.basic(apiKey, "");
    }

    @Override
    public Response intercept(Chain chain) throws IOException {
        Request authenticatedRequest = chain.request().newBuilder()
            .header("Authorization", credentials)
            .build();
        return chain.proceed(authenticatedRequest);
    }
}
```

### 2.4 Streaming-Download-Pattern

Der zentrale Vorteil von `@Streaming` + OkHttp: CSV-Responses werden **zeilenweise**
verarbeitet, ohne den gesamten Body in den RAM zu laden. Wichtig bei MBP-1 Downloads
mit 100k+ Zeilen.

```java
/**
 * Beispiel: Streaming-Download mit zeilenweisem Parsing.
 */
public List<ParsedQuoteTick> streamQuotes(DabentoRequest request) {
    Call<ResponseBody> call = dabentoApi.getRange(
        request.dataset(), String.join(",", request.symbols()),
        request.schema(), request.startDate().toString(),
        request.endDate().toString(), request.stypeIn(),
        request.encoding(), "none"
    );

    try (Response<ResponseBody> response = call.execute()) {
        if (!response.isSuccessful()) {
            throw new DabentoApiException(response.code(), response.message());
        }

        try (BufferedSource source = response.body().source()) {
            // Header-Zeile lesen
            String header = source.readUtf8Line();

            List<ParsedQuoteTick> batch = new ArrayList<>(BATCH_SIZE);
            String line;
            while ((line = source.readUtf8Line()) != null) {
                batch.add(csvParser.parseQuoteLine(line));
                if (batch.size() >= BATCH_SIZE) {
                    persister.persistQuoteBatch(batch);
                    batch.clear();
                }
            }
            // Restliche Zeilen
            if (!batch.isEmpty()) {
                persister.persistQuoteBatch(batch);
            }
        }
    }
}
```

### 2.5 Request-Record

```java
public record DabentoRequest(
    String dataset,          // "XNAS.ITCH"
    List<String> symbols,    // ["IREN"]
    String schema,           // "mbp-1", "trades", "ohlcv-1m"
    LocalDate startDate,     // Inklusiv
    LocalDate endDate,       // Exklusiv
    String stypeIn,          // "raw_symbol" (Default)
    String encoding          // "csv" (Default)
) {
    // Factory-Methods für häufige Patterns:
    // DabentoRequest.quotes("IREN", LocalDate.of(2026, 2, 23))
    // DabentoRequest.trades("IREN", LocalDate.of(2026, 2, 23))
    // DabentoRequest.ohlcv1m("IREN", date, date.plusDays(5))
}
```

### 2.3 Port-Interface

```java
/**
 * Port-Interface für den Databento-Datenprovider.
 * Implementierung in odin-data, programmiert gegen Interface in odin-api.
 */
public interface HistoricalDataProvider {

    /** Schätzt die Kosten einer Anfrage in USD. */
    BigDecimal estimateCost(DabentoRequest request);

    /** Lädt Quote-Ticks (MBP-1) für einen Zeitraum herunter und persistiert sie. */
    DataIngestResult downloadQuotes(String symbol, LocalDate startDate, LocalDate endDate);

    /** Lädt Trade-Ticks für einen Zeitraum herunter und persistiert sie. */
    DataIngestResult downloadTrades(String symbol, LocalDate startDate, LocalDate endDate);

    /** Lädt OHLCV-Bars (1m) herunter und persistiert sie in intraday_bar. */
    DataIngestResult downloadBars(String symbol, LocalDate startDate, LocalDate endDate,
                                   int barIntervalSec);
}

public record DataIngestResult(
    String symbol,
    LocalDate startDate,
    LocalDate endDate,
    String schema,
    long rowCount,
    BigDecimal costUsd,
    Duration downloadDuration,
    Duration persistDuration
) {}
```

---

## 3. CSV-Parser Design

### 3.1 Warum CSV als primäres Format

- **Einfach in Java parsbar** — kein DBN-Binärparser nötig
- **Debuggbar** — menschenlesbar, per Texteditor inspizierbar
- **Ausreichend performant** für selektive Downloads (ein Instrument, ein Tag)
- DBN wäre 10-50x kompakter, aber für ODIN-Volumina nicht nötig

### 3.2 CSV-Felder (MBP-1)

Erwartete CSV-Spalten (Databento CSV-Encoding):

```csv
ts_event,ts_recv,publisher_id,instrument_id,action,side,depth,price,size,flags,
sequence,bid_px_00,ask_px_00,bid_sz_00,ask_sz_00,bid_ct_00,ask_ct_00,symbol
```

### 3.3 Parser-Records

```java
/** Geparster MBP-1 Record, bereit zur Persistierung. */
public record ParsedQuoteTick(
    long tsEventNs,          // Nanosekunden seit Epoch
    Instant tsEvent,         // Auf Mikrosekunden gerundet (für TIMESTAMPTZ)
    String symbol,
    String exchange,
    LocalDate tradingDay,
    char action,
    char side,
    short flags,
    long sequence,
    short publisherId,
    long price,              // Fixed-Point 1e-9
    int size,
    long bidPx,
    long askPx,
    int bidSz,
    int askSz,
    int bidCt,
    int askCt,
    Long tsRecvNs,
    Instant tsRecv
) {}

/** Geparster Trade Record. */
public record ParsedTradeTick(
    long tsEventNs,
    Instant tsEvent,
    String symbol,
    String exchange,
    LocalDate tradingDay,
    long price,
    int size,
    char side,
    short flags,
    long sequence,
    short publisherId,
    Long tsRecvNs,
    Instant tsRecv
) {}
```

### 3.4 Nanosekunden → Instant Konvertierung

```java
// Databento ns-Epoch → Java Instant (Mikrosekunden-Präzision für TIMESTAMPTZ)
Instant tsEvent = Instant.ofEpochSecond(
    tsEventNs / 1_000_000_000L,
    (tsEventNs % 1_000_000_000L) / 1000 * 1000  // Auf Mikrosekunden runden
);
```

---

## 4. Persistierung

### 4.1 JDBC Batch Insert (analog zu `IntradayBarJdbcRepository`)

```java
@Repository
public class QuoteTickJdbcRepository {

    private static final String INSERT_SQL = """
        INSERT INTO odin.quote_tick (
            ts_event, ts_event_ns, symbol, exchange, trading_day,
            action, side, flags, sequence, publisher_id,
            price, size,
            bid_px, ask_px, bid_sz, ask_sz, bid_ct, ask_ct,
            ts_recv, ts_recv_ns, source
        ) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, 'DATABENTO')
        ON CONFLICT (symbol, ts_event_ns, sequence, publisher_id) DO NOTHING
        """;

    // Batch-Size: 10.000 Rows pro executeBatch()
    // Gesamter Tag in einer Transaktion
}
```

### 4.2 COPY-Alternative (für maximale Performance)

Falls Insert-Performance zum Engpass wird:

```java
// PostgreSQL COPY-Protokoll über PgConnection
CopyManager cm = connection.unwrap(PgConnection.class).getCopyAPI();
cm.copyIn("COPY odin.quote_tick (...) FROM STDIN WITH (FORMAT csv)", inputStream);
```

**Für den initialen Client:** Batch-Insert reicht. COPY als Optimierung für später.

---

## 5. Download-Workflow

### 5.1 Standardablauf (einzelner Handelstag)

```
1. Prüfe: Existieren bereits Daten?
   → SELECT COUNT(*) FROM odin.data_ingest_log
     WHERE symbol='IREN' AND trading_day='2026-02-23' AND schema_name='mbp-1'
   → Falls ja: Skip (idempotent)

2. Kostenschätzung
   → GET /v0/metadata.get_cost?dataset=XNAS.ITCH&symbols=IREN
         &schema=mbp-1&start=2026-02-23&end=2026-02-24
   → Logge geschätzte Kosten

3. Download (Streaming)
   → GET /v0/timeseries.get_range (gleiche Parameter, encoding=csv)
   → Response streamen, zeilenweise parsen
   → In Batches à 10.000 Rows persistieren

4. Audit-Log schreiben
   → INSERT INTO odin.data_ingest_log (...)

5. Optional: Compression sofort triggern
   → SELECT compress_chunk(c) FROM show_chunks('odin.quote_tick') c
     WHERE range_end < now() - INTERVAL '1 day'
```

### 5.2 Mehrtägiger Download (z.B. eine Woche)

```
Für jedes Datum in [start_date, end_date):
    Schritt 1-5 wie oben (pro Tag)
```

Tageweise Granularität statt Wochen-Download, damit:
- Fortschritt trackbar (pro Tag im `data_ingest_log`)
- Fehler nur einen Tag betreffen, nicht den gesamten Range
- Idempotentes Re-Run für fehlgeschlagene Tage

### 5.3 Paralleler Download mehrerer Schemas

Für ein Instrument und einen Tag können MBP-1, Trades und OHLCV-1m
**parallel** heruntergeladen werden (drei HTTP-Requests).

---

## 6. Konfiguration (CSpec-konform)

```properties
# Namespace: odin.data.databento
odin.data.databento.api-key=${DATABENTO_API_KEY}
odin.data.databento.base-url=https://hist.databento.com/v0
odin.data.databento.default-dataset=XNAS.ITCH
odin.data.databento.default-encoding=csv
odin.data.databento.timeout-seconds=300
odin.data.databento.max-retries=3
odin.data.databento.batch-size=10000
odin.data.databento.cost-limit-usd=10.00
```

**`cost-limit-usd`:** Safety-Net — Download wird abgebrochen, wenn die geschätzte
Kosten diesen Wert überschreiten. Schützt vor versehentlich teuren Anfragen.

---

## 7. Fehlerbehandlung

### 7.1 OkHttp/Retrofit-Ebene

| Fehler | Quelle | Handling |
|--------|--------|----------|
| HTTP 401 Unauthorized | `response.code() == 401` | API-Key ungültig → `DabentoAuthException`, Log |
| HTTP 429 Rate Limit | `response.code() == 429` | Retry mit Exponential Backoff (OkHttp-Interceptor) |
| HTTP 400 Bad Request | `response.code() == 400` | Symbol/Dataset ungültig → `DabentoRequestException`, Log |
| HTTP 5xx Server Error | `response.code() >= 500` | Retry (max 3x), dann `DabentoServerException` |
| `IOException` | OkHttp Netzwerk | Timeout/Verbindungsfehler → Retry, Log |
| `SocketTimeoutException` | OkHttp Read-Timeout | Timeout bei großem Streaming-Download → readTimeout erhöhen |

### 7.2 Retry-Interceptor (OkHttp)

```java
/**
 * OkHttp Interceptor für Retry-Logik mit Exponential Backoff.
 * Nur für 429 (Rate Limit) und 5xx (Server Error).
 */
public class DabentoRetryInterceptor implements Interceptor {

    private static final int MAX_RETRIES = 3;
    private static final long INITIAL_BACKOFF_MS = 1000;

    @Override
    public Response intercept(Chain chain) throws IOException {
        Request request = chain.request();
        Response response = null;

        for (int attempt = 0; attempt <= MAX_RETRIES; attempt++) {
            if (response != null) {
                response.close();
            }
            response = chain.proceed(request);

            if (response.isSuccessful() || !isRetryable(response.code())) {
                return response;
            }

            long backoffMs = INITIAL_BACKOFF_MS * (1L << attempt);
            Thread.sleep(backoffMs);  // Simplified — use ScheduledExecutor in production
        }
        return response;
    }

    private boolean isRetryable(int code) {
        return code == 429 || code >= 500;
    }
}
```

### 7.3 Anwendungsebene

| Fehler | Handling |
|--------|----------|
| CSV-Parse-Fehler | Zeile loggen, überspringen, Warnung — kein Abbruch |
| DB-Constraint-Violation | ON CONFLICT DO NOTHING (Duplikate ignorieren) |
| Kosten über Limit | `DatabentoCostLimitException` vor dem Download |

---

## 8. Sicherheit

- API-Key **nur** über Environment-Variable `DATABENTO_API_KEY`
- Key **nie** in Properties-Dateien, Logs oder Fehlermeldungen
- Key **nie** an LLM-Pools (ChatGPT/Gemini) senden
- HTTPS für alle API-Calls (Databento erzwingt dies)
