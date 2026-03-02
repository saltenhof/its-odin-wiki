# ODIN-093: Tick JDBC Repositories — Implementation Protocol

## Scope

JDBC Batch Repositories for `odin.quote_tick`, `odin.trade_tick`, and `odin.data_ingest_log`
in the `odin-data` module. Batch-Insert with ON CONFLICT DO NOTHING for tick tables,
insert/update/exists for ingest audit log. Configurable batch size (default 10,000).

## Deliverables

### Production Code (4 files)

| File | Description |
|------|-------------|
| `QuoteTickJdbcRepository.java` | @Repository, batch insert for 21 columns, ON CONFLICT (symbol, ts_event, ts_event_ns, sequence, publisher_id) DO NOTHING |
| `TradeTickJdbcRepository.java` | @Repository, batch insert for 14 columns, same ON CONFLICT pattern |
| `DataIngestLogJdbcRepository.java` | @Repository, logIngest (KeyHolder), existsIngest (COALESCE for NULL dataset), markCompleted, markFailed |
| `DataIngestLogEntry.java` | Record with 12 fields mapping to odin.data_ingest_log |

All in: `odin-data/src/main/java/de/its/odin/data/databento/repository/`

### Test Code (6 files)

| File | Type | Tests |
|------|------|-------|
| `QuoteTickJdbcRepositoryTest.java` | Unit | 7 tests: emptyList, singleChunk, multipleChunks, withConflicts, defaultConstructor, batchSizeZero, batchSizeNegative |
| `TradeTickJdbcRepositoryTest.java` | Unit | 7 tests: emptyList, singleChunk, multipleChunks, withConflicts, exactBatchSize, batchSizeZero, batchSizeNegative |
| `DataIngestLogJdbcRepositoryTest.java` | Unit | 7 tests: existsIngest true/false/null, markCompleted, markFailed, recordFields, nullableFields |
| `QuoteTickJdbcRepositoryIntegrationTest.java` | Integration | 9 tests: singleTick, multipleTicks, duplicateTicks, nullableBboFields, charFields, bigintPriceFields, emptyList, sourceColumn, duplicatesWithinSameCall |
| `TradeTickJdbcRepositoryIntegrationTest.java` | Integration | 8 tests: singleTick, multipleTicks, duplicateTicks, sideField, nullReceiveTimestamps, priceField, emptyList, sourceColumn |
| `DataIngestLogJdbcRepositoryIntegrationTest.java` | Integration | 12 tests: logIngest, storesAllFields, existsIngest false/true/nullDataset/differentDatasets, markCompleted, markFailed, markCompleted_nonExistentId, fullLifecycle, markFailed_nonExistentId, logIngest_duplicateKey |

All in: `odin-data/src/test/java/de/its/odin/data/databento/repository/`

### Test Infrastructure (2 files)

| File | Description |
|------|-------------|
| `V000__stub_timescaledb.sql` | PL/pgSQL no-ops for create_hypertable and add_compression_policy |
| `application.properties` | Flyway config: schemas=odin, default-schema=odin, create-schemas=true |

In: `odin-data/src/test/resources/`

### Build Configuration

- `odin-data/pom.xml`: Added odin-persistence dependency, spring-boot-starter-jdbc, Zonky test dependencies

## Test Results

- Unit Tests: **21 pass** (7 + 7 + 7)
- Integration Tests: **29 pass** (9 + 8 + 12)
- Total new tests: **50**
- Full odin-data unit test suite: **396 tests pass** (no regressions)

## Technical Decisions

### ON CONFLICT Dedup Index
`(symbol, ts_event, ts_event_ns, sequence, publisher_id)` — ts_event is included because
TimescaleDB mandates the partitioning column in all unique indexes on hypertables.

### Nullable Field Binding
Nullable BIGINT/INTEGER fields use `ps.setObject(idx, value, Types.BIGINT)` instead of
`ps.setLong()` to avoid NullPointerException from auto-unboxing.

### CHAR(1) Binding
Action and side fields bound with `ps.setObject(idx, String.valueOf(char), Types.CHAR)`.

### DataIngestLog: No ON CONFLICT
Callers must use `existsIngest()` before `logIngest()`. TOCTOU race condition is documented
and acceptable for the current use case (single-threaded tageweise downloads).

### COALESCE for NULL Dataset
`COALESCE(dataset, '')` in both the EXISTS query and the unique index ensures correct
NULL-safe equality for providers without a dataset concept (e.g., Yahoo).

### Batch Splitting
Manual chunk-based iteration over `List.subList()` with configurable batchSize, despite Spring's
built-in batch support, for explicit control over memory and execution boundaries.

### countInserted
Handles `Statement.SUCCESS_NO_INFO` as 1 inserted row. Acceptable for telemetry purposes;
not used for transactional correctness.

## LLM Sparring Summary

### ChatGPT Review (DoD 5.5) — Owner: ODIN-093-worker

**Findings adopted:**
- batchSize <= 0 caused infinite loop in persistBatch -> Added IllegalArgumentException validation in both tick repository constructors
- Missing test: markFailed_nonExistentId -> Added integration test
- Missing test: logIngest_duplicateKey -> Added integration test (verifies DataIntegrityViolationException)
- Missing test: duplicatesWithinSameCall -> Added integration test for QuoteTickJdbcRepository

**Findings noted (no action needed):**
- SUCCESS_NO_INFO counting may overcount -> Acceptable for telemetry
- existsIngest + logIngest TOCTOU race -> Documented, acceptable for single-threaded use
- COUNT(*) vs EXISTS performance -> Acceptable for current volume (<1 query/day/instrument)

### Gemini Review 1: Code-Bugs (DoD 5.6) — Owner: ODIN-093-worker

**Findings:**
- Redundant batch-splitting logic (Spring already handles chunking) -> Kept for explicit control
- NullPointerException risk on tsEvent/startedAt -> NOT NULL in DB schema, intentionally not guarded
- SUCCESS_NO_INFO counting -> Same as ChatGPT finding
- TOCTOU race condition -> Same as ChatGPT finding

**Result:** No code changes required. All findings were already known or intentional design decisions.

### Gemini Review 2: Konzepttreue (DoD 5.6) — Owner: ODIN-093-worker

**Result:** Full concept compliance confirmed. All 5 checkpoints passed:
1. ON CONFLICT matches dedup index exactly
2. All DDL columns covered (ingested_at correctly omitted — uses DEFAULT)
3. DataIngestLogEntry matches table structure (ingest_id correctly omitted — generated)
4. Batch size configurable via constructor
5. Source set to 'DATABENTO'

### Gemini Review 3: Praxis-Gaps (DoD 5.6) — Owner: ODIN-093-worker

**Findings:**
1. Half trading days (Early Close) -> No impact at repository level; downstream concern for backtest logic
2. Network interrupt mid-batch -> Partial commits safe due to ON CONFLICT DO NOTHING; zombie STARTED entries possible if caller doesn't call markFailed -> Caller's responsibility
3. High volume (1M+ ticks) -> Full List in RAM is caller's concern; streaming is a future optimization option
4. Timezone traps -> `Timestamp.from(Instant)` may shift based on JVM timezone; mitigated by storing both TIMESTAMP and nanosecond BIGINT columns, and Instant being UTC-based
5. Disk full -> External monitoring needed; if DB is 100% full, markFailed will also fail
6. Thread safety -> Confirmed fully thread-safe (final fields, stack-local variables only)

**Result:** No code changes required. All findings are operational/architectural concerns outside the repository layer's scope.
