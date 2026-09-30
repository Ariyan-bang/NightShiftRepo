# Meridian Platform — Subsystem Architecture & Interaction Specification

> **Architectural Provenance**: This document was reverse-engineered directly from the Meridian source codebase (C, Python, FORTRAN, Java, JavaScript, TypeScript, and SQL). It details how the **seven subsystems** interact, how they pass control and data, and what programming languages and standards each subsystem is written in.

---

## 1. Executive Architectural Overview

Meridian is an agri-environmental telemetry ingestion, numerical crop simulation, REST API, web console, and operational reporting platform. The system is partitioned into **seven distinct subsystems**, spanning multiple eras of software development from 1994 to 2023:

```
  [Binary Telemetry]
          │
          ▼
   1. ingest (C)
          │  popen("sqlite3 ...")
          ▼
   [SQLite DB: reading]
          │
          ▼
   2. analytics (Python) ──writes deck──► [MERIDIAN.DAT]
          │                                      │
     subprocess.run()                            ▼
          │                              3. model (FORTRAN)
          │                                      │
          │◄─────reads output───── [MERIDIAN.OUT]
          │
          ▼
   [SQLite DB: forecast & forecast_daily]
       ▲                 ▲
       │                 │
  sqlite3 -json      read-only URI
       │                 │
 4. services (Java)  7. reporting (Python)
    :8081 REST API       Operational Reports
       ▲       ▲
  $.ajax()   fetch()
       │       │
 5. dashboard  6. ui-next
 (JS/jQuery)   (TS/React)
```

---

## 2. Seven Subsystems Matrix

| # | Subsystem | Directory | Language & Standard | Primary Role | Control Ingress / Trigger | Control Egress / Handoff | Data Ingress Format | Data Egress Format |
|---|---|---|---|---|---|---|---|---|
| **1** | **`ingest`** | `ingest/` | **C (C99 / POSIX)** | Telemetry Ingestion & QC Gate | CLI execution / cron (`mrd_ingest`) | Spawns `sqlite3` CLI via `popen()` | 44-byte binary big-endian frames from spool | SQL `INSERT OR REPLACE` into `reading` table |
| **2** | **`analytics`** | `analytics/` | **Python (3.8+)** | Batch Orchestration & Deck Builder | CLI execution / cron (`run_forecast.py`) | Spawns `cropmod` via `subprocess.run()` (blocks) | Queries `reading` table from SQLite DB | Writes `MERIDIAN.DAT`, persists to `forecast` & `forecast_daily` |
| **3** | **`model`** | `model/` | **FORTRAN (F77 / F90)** | Numerical Crop Growth & Water Balance | Invoked as OS child process by `analytics` | Terminates via `STOP` (exit code 0) back to parent | Column-exact ASCII `MERIDIAN.DAT` | Column-exact ASCII `MERIDIAN.OUT` + stdout log |
| **4** | **`services`** | `services/` | **Java (JDK 8+)** | HTTP REST API Server | Daemon process on port 8081 (`HttpServer`) | Spawns `sqlite3 -json` via `ProcessBuilder` | HTTP GET requests from browser clients | JSON payloads with CORS headers (`*`) |
| **5** | **`dashboard`** | `dashboard/` | **JavaScript (ES5 / jQuery) + Node.js** | Legacy Web Operator Console | Browser DOM event loop + URL hashchange | Dispatches async AJAX `$.ajax()` calls | JSON from Java API on `:8081` | Dynamic HTML DOM updates & `<div>` bar charts |
| **6** | **`ui-next`** | `ui-next/` | **TypeScript / React 18 / Vite** | Modern Web Operator Console | Browser DOM event loop + React Router 6 | Dispatches async `fetch()` HTTP requests | Typed JSON from Java API on `:8081` | Virtual DOM rendering & multi-channel SVG charts |
| **7** | **`reporting`** | `reporting/` | **Python (3.8+ / CLI)** | Fixed-Format Operational Reports | CLI invocation (`python -m reporting`) | Context-managed execution; writes to file/stdout | Read-only SQL (`mode=ro`) from SQLite DB | Column-exact 80/132-col text (greenbar) or CSV |

---

## 3. UML Component Diagram

The UML Component Diagram ([docs/uml/subsystems_component.puml](file:///c:/Users/archo/CIS3250/meridian-archaeology/docs/uml/subsystems_component.puml)) models the seven subsystems, their implementation languages, one-line responsibilities, exposed interfaces, and control vs data flows:

```mermaid
graph TB
    subgraph T1["1. Ingestion Tier"]
        Ingest["<b>1. ingest</b><br><i>C (C99 / POSIX)</i><br><b>Responsibility:</b> Decodes/QC-validates binary frames and appends readings to SQLite.<br><b>Interface:</b> CLI: mrd_ingest &lt;spool&gt; &lt;db&gt;"]
    end

    subgraph T2["2. Persistence & Scratch Store"]
        DB[("<b>meridian.db</b><br><i>SQLite 3 Store</i><br>reading | forecast | forecast_daily")]
        FS[("<b>Scratch Workspace</b><br><i>Fixed-Width Files</i><br>MERIDIAN.DAT | MERIDIAN.OUT")]
    end

    subgraph T3["3. Batch Simulation Tier"]
        Analytics["<b>2. analytics</b><br><i>Python (3.8+)</i><br><b>Responsibility:</b> Queries weather, builds decks, executes cropmod, and stores forecasts.<br><b>Interface:</b> CLI: run_forecast.py --db &lt;path&gt;"]
        Model["<b>3. model</b><br><i>FORTRAN (F77 / F90)</i><br><b>Responsibility:</b> Simulates daily soil water balance and crop biomass growth until harvest.<br><b>Interface:</b> Executable & I/O: cropmod (MERIDIAN.DAT)"]
    end

    subgraph T4["4. API Services Tier"]
        Services["<b>4. services</b><br><i>Java (JDK 8+)</i><br><b>Responsibility:</b> Serves telemetry and forecast records as JSON over HTTP with CORS.<br><b>Interface:</b> REST API: HTTP :8081 /api/{stations, forecast, series, summary, health}"]
    end

    subgraph T5["5. Presentation & Operational Reporting Tier"]
        Dashboard["<b>5. dashboard</b><br><i>JavaScript (ES5/jQuery)</i><br><b>Responsibility:</b> Renders legacy browser console with hash routing and jQuery AJAX bar charts.<br><b>Interface:</b> Web UI: HTTP :8080 index.html"]
        UiNext["<b>6. ui-next</b><br><i>TypeScript (React 18)</i><br><b>Responsibility:</b> Renders modern typed SPA with client routing and SVG time-series charts.<br><b>Interface:</b> Web SPA: HTTP :5174 /"]
        Reporting["<b>7. reporting</b><br><i>Python (3.8+)</i><br><b>Responsibility:</b> Generates 80/132-col greenbar printouts and CSV exports via read-only SQLite.<br><b>Interface:</b> CLI: python -m reporting &lt;type&gt; --db &lt;path&gt;"]
    end

    %% Control Flows (solid blue)
    Cron["Cron / CLI Trigger"] ==>|<<control>> cron 15m| Ingest
    Ingest ==>|<<control>> popen('sqlite3')| DB
    Cron ==>|<<control>> cron nightly| Analytics
    Analytics ==>|<<control>> subprocess.run| Model
    Cron ==>|<<control>> start daemon| Services
    Services ==>|<<control>> ProcessBuilder sqlite3 -json| DB
    WebUser["Web Browser"] ==>|<<control>> navigates :8080| Dashboard
    Dashboard ==>|<<control>> $.ajax()| Services
    WebUser ==>|<<control>> navigates :5174| UiNext
    UiNext ==>|<<control>> fetch()| Services
    Cron ==>|<<control>> run CLI| Reporting
    Reporting ==>|<<control>> connect mode=ro| DB

    %% Data Flows (dashed green)
    Spool["Spool: spool_*.bin<br>(44-byte frames)"] -.->|<<data>> binary frames| Ingest
    Ingest -.->|<<data>> INSERT INTO reading| DB
    DB -.->|<<data>> SELECT weather readings| Analytics
    Analytics -.->|<<data>> writes MERIDIAN.DAT| FS
    FS -.->|<<data>> reads MERIDIAN.DAT| Model
    Model -.->|<<data>> writes MERIDIAN.OUT| FS
    FS -.->|<<data>> parses MERIDIAN.OUT| Analytics
    Analytics -.->|<<data>> INSERT INTO forecast tables| DB
    DB -.->|<<data>> JSON stream| Services
    Services -.->|<<data>> HTTP 200 JSON| Dashboard
    Services -.->|<<data>> HTTP 200 JSON| UiNext
    DB -.->|<<data>> SELECT read-only| Reporting
    Reporting -.->|<<data>> ASCII greenbar / CSV| OutputReports["Official Printed Reports / CSV"]

    classDef c_lang fill:#e8f4f8,stroke:#0277bd,stroke-width:1.5px;
    classDef py_lang fill:#fff3cd,stroke:#ff8f00,stroke-width:1.5px;
    classDef fortran_lang fill:#f8d7da,stroke:#c2185b,stroke-width:1.5px;
    classDef java_lang fill:#e2d9f3,stroke:#6a1b9a,stroke-width:1.5px;
    classDef js_lang fill:#d2f4ea,stroke:#00796b,stroke-width:1.5px;
    classDef ts_lang fill:#cfe2ff,stroke:#1565c0,stroke-width:1.5px;
    classDef db_node fill:#f5f5f5,stroke:#424242,stroke-width:1.5px;

    class Ingest c_lang;
    class Analytics,Reporting py_lang;
    class Model fortran_lang;
    class Services java_lang;
    class Dashboard js_lang;
    class UiNext ts_lang;
    class DB,FS db_node;
```

---

## 4. UML Sequence & Interaction Diagram

The sequence diagram details the end-to-end execution pipeline, highlighting the transfer of control, process blocking, asynchronous events, and data payloads.

```mermaid
sequenceDiagram
    autonumber
    actor Cron as Cron / Operator
    participant Ingest as 1. ingest (C)
    participant DB as SQLite (meridian.db)
    participant Analytics as 2. analytics (Python)
    participant Model as 3. model (FORTRAN)
    participant Services as 4. services (Java :8081)
    participant Dashboard as 5. dashboard (JS)
    participant UiNext as 6. ui-next (TS/React)
    participant Reporting as 7. reporting (Python)

    %% Phase 1
    rect rgb(232, 244, 248)
    Note over Cron, Ingest: Phase 1: Telemetry Ingestion (C)
    Cron->>Ingest: Exec: mrd_ingest spool_2025.bin meridian.db
    activate Ingest
    Ingest->>Ingest: Parse 44-byte binary packets & verify CRC16-CCITT
    Ingest->>Ingest: Scale values (x100) & run QC range checks
    Ingest->>DB: popen("sqlite3 meridian.db") -> BEGIN; INSERT INTO reading; COMMIT;
    Ingest-->>Cron: Log stats to stderr & exit(0)
    deactivate Ingest
    end

    %% Phase 2
    rect rgb(255, 243, 205)
    Note over Cron, Model: Phase 2: Numerical Forecast Batch (Python & FORTRAN)
    Cron->>Analytics: Exec: python3 analytics/run_forecast.py --db meridian.db
    activate Analytics
    Analytics->>DB: SELECT readings for station/year
    DB-->>Analytics: Return weather observations (tmax, tmin, rain, srad)
    Analytics->>Analytics: mkdtemp() & write column-exact MERIDIAN.DAT
    Analytics->>Model: subprocess.run(['cropmod'], cwd=scratch) [BLOCKING]
    activate Model
    Model->>Model: READ MERIDIAN.DAT -> WATBAL -> GROWTH
    Model->>Model: WRITE MERIDIAN.OUT -> STOP (exit 0)
    Model-->>Analytics: Process exit code 0
    deactivate Model
    Analytics->>Analytics: parse.py: Fixed-column slice of MERIDIAN.OUT (MRD-143)
    Analytics->>DB: INSERT INTO forecast & forecast_daily
    Analytics-->>Cron: Print summary table & exit(0)
    deactivate Analytics
    end

    %% Phase 3
    rect rgb(226, 217, 243)
    Note over Cron, UiNext: Phase 3: REST API & Web Consoles (Java, JS, TS)
    Cron->>Services: Launch Java ApiServer on :8081
    activate Services

    Dashboard->>Services: HTTP GET /api/stations ($.ajax)
    Services->>DB: ProcessBuilder: sqlite3 -json "SELECT ... FROM reading ..."
    DB-->>Services: JSON Array
    Services-->>Dashboard: HTTP 200 JSON with CORS
    Dashboard->>Dashboard: Render DOM table & <div> bar charts

    UiNext->>Services: HTTP GET /api/series?station=GUELPH&year=2021 (fetch)
    Services->>DB: ProcessBuilder: sqlite3 -json "SELECT ... FROM forecast_daily ..."
    DB-->>Services: JSON Array
    Services-->>UiNext: HTTP 200 JSON
    UiNext->>UiNext: SeriesChart.tsx renders interactive SVG time series
    end

    %% Phase 4
    rect rgb(232, 248, 245)
    Note over Cron, Reporting: Phase 4: Operational Reporting (Python)
    Cron->>Reporting: Exec: python -m reporting season --year 2021
    activate Reporting
    Reporting->>DB: sqlite3.connect("file:meridian.db?mode=ro", uri=True)
    DB-->>Reporting: Forecast and daily records
    Reporting->>Reporting: Rank yields, format 80-col greenbar with \f pagination
    Reporting-->>Cron: Emit fixed-format report to stdout/file & exit(0)
    deactivate Reporting
    end
```

### 4.1 Whole-System End-to-End Trace: Guelph Field Reading to Console

The dedicated UML sequence diagram ([docs/uml/end_to_end_trace.puml](file:///c:/Users/archo/CIS3250/meridian-archaeology/docs/uml/end_to_end_trace.puml), [PNG](file:///c:/Users/archo/CIS3250/meridian-archaeology/docs/uml/end_to_end_trace.png), [SVG](file:///c:/Users/archo/CIS3250/meridian-archaeology/docs/uml/end_to_end_trace.svg)) models the lifecycle of a single weather reading from field station **GUELPH** through the exact architectural hand-off path:
`frame -> ingest -> store -> orchestrator -> model -> output -> parser -> store -> API -> console`

#### Data Format Handoff Table

| Step | From $\to$ To | Data Format / Protocol | Concrete Payload / Schema |
|---|---|---|---|
| **1. frame** | Field Station $\to$ Ingest | 44-byte Binary Big-Endian Frame (`packet.h`) | Magic `0x4D524431` ("MRD1"), Station `'GUELPH  '`, Year `2025`, DOY `120`, Channels $\times 100$: TMAX=1850 ($18.50^\circ\text{C}$), TMIN=620 ($6.20^\circ\text{C}$), RAIN=0 ($0.00\,\text{mm}$), SRAD=2240 ($22.40\,\text{MJ/m}^2$), RH=5500 ($55.0\%$), WIND=320 ($3.20\,\text{m/s}$), Flags `0x0000`, CRC16 `0x...` |
| **2. ingest** | Ingest $\to$ Store | SQL text stream piped via `popen("sqlite3")` | `INSERT OR REPLACE INTO reading (stnid, year, doy, tmax, tmin, rain, srad, rh, wind, flags) VALUES ('GUELPH', 2025, 120, 18.50, 6.20, 0.00, 22.40, 55.00, 3.20, 0);` |
| **3. store** | Store $\to$ Orchestrator | Python `sqlite3` cursor query / row tuples | `SELECT doy, tmax, tmin, rain, srad FROM reading WHERE stnid='GUELPH' AND year=2025 ORDER BY doy ASC;` $\to$ `(120, 18.50, 6.20, 0.00, 22.40)` |
| **4. orchestrator** | Orchestrator $\to$ Model | Column-exact fixed-width ASCII input deck (`MERIDIAN.DAT`) | `GUELPH  2025  43.545`<br>`    0.15 1100.00`<br>`120  18.5   6.2   0.0 22.40`<br>` -1` |
| **5. model** | Model $\to$ Output | Column-exact fixed-width ASCII output file (`MERIDIAN.OUT`) | `MERIDIAN CROPMOD V2.3  STN=GUELPH    YEAR=2025  NDAYS= 180`<br>` DOY      SW      ET   DRAIN    BIOM     LAI`<br>` 120  182.40   2.150   0.000   120.5   0.450`<br>`...`<br>`YIELD      6.425` |
| **6. output** | Output $\to$ Parser | Raw text file / Fixed-column character slices | Column spans `[(0,4), (5,12), (13,20), (21,28), (29,36), (37,44)]` parsed to avoid whitespace split bugs (MRD-143). |
| **7. parser** | Parser $\to$ Store | Parameterized SQL statements via Python `sqlite3` | `INSERT OR REPLACE INTO forecast_daily (stnid, year, doy, sw, et, drain, biom, lai) VALUES ('GUELPH', 2025, 120, 182.40, 2.150, 0.000, 120.5, 0.450);`<br>`INSERT OR REPLACE INTO forecast (stnid, year, run_at, yield_t, ndays, model) VALUES ('GUELPH', 2025, '2025-05-01T04:00:00', 6.425, 180, 'cropmod V2.3');` |
| **8. store** | Store $\to$ API | JSON Array stream piped via `ProcessBuilder("sqlite3", "-json", ...)` | `[{"doy":120,"sw":182.40,"et":2.150,"drain":0.000,"biom":120.5,"lai":0.450}]` |
| **9. API** | API $\to$ Console | HTTP Response (JSON + CORS headers) | HTTP 200 OK (`Content-Type: application/json`, `Access-Control-Allow-Origin: *`)<br>`[{"doy":120,"sw":182.40,"et":2.150,"drain":0.000,"biom":120.5,"lai":0.450}]` |
| **10. console** | Console (Presentation) | DOM Table / Multi-Channel SVG Path | Rendered operator metrics: Soil Water at DOY 120 = **182.40 mm**, Final Yield = **6.425 t/ha**. |

---

## 5. Subsystem Deep Dives: Control & Data Contracts

### 5.1 Subsystem 1: `ingest`
- **Language**: C (C99 / POSIX)
- **Source Files**: [`ingest/src/mrd_ingest.c`](file:///c:/Users/archo/CIS3250/meridian-archaeology/ingest/src/mrd_ingest.c), [`packet.c`](file:///c:/Users/archo/CIS3250/meridian-archaeology/ingest/src/packet.c), [`packet.h`](file:///c:/Users/archo/CIS3250/meridian-archaeology/ingest/src/packet.h), [`decode.c`](file:///c:/Users/archo/CIS3250/meridian-archaeology/ingest/src/decode.c), [`validate.c`](file:///c:/Users/archo/CIS3250/meridian-archaeology/ingest/src/validate.c), [`tsstore.c`](file:///c:/Users/archo/CIS3250/meridian-archaeology/ingest/src/tsstore.c).
- **How Control is Passed**:
  - Called via CLI: `./ingest/mrd_ingest <spoolfile> <store.db>`.
  - In `tsstore.c`, calls `popen("sqlite3 <store.db>", "w")`. It writes SQL transaction commands directly to the `sqlite3` process stdin pipe and calls `pclose()` to commit and terminate the subprocess.
- **How Data is Passed**:
  - **Ingress**: Reads 44-byte binary big-endian frames defined by `struct mrd_packet` in `packet.h`:
    - Bytes 0-3: `uint32_t magic` (`0x4D524431`, ASCII `"MRD1"`).
    - Bytes 4-11: `char stnid[8]` (space-padded station identifier).
    - Bytes 12-13: `uint16_t year`.
    - Bytes 14-15: `uint16_t doy` (day of year).
    - Bytes 16-39: `int32_t value[6]` (weather channels scaled by $\times 100$: TMAX, TMIN, RAIN, SRAD, RH, WIND).
    - Bytes 40-41: `uint16_t flags` (`0x0001` suspect, `0x0002` manual).
    - Bytes 42-43: `uint16_t crc` (CRC16-CCITT checksum over bytes 0..41, poly `0x1021`, init `0xFFFF`).
  - **Egress**: Generates SQL text and pipes to `sqlite3`:
    ```sql
    CREATE TABLE IF NOT EXISTS reading (
      stnid TEXT NOT NULL, year INTEGER NOT NULL, doy INTEGER NOT NULL,
      tmax REAL, tmin REAL, rain REAL, srad REAL, rh REAL, wind REAL,
      flags INTEGER NOT NULL DEFAULT 0,
      PRIMARY KEY (stnid, year, doy)
    );
    INSERT OR REPLACE INTO reading VALUES ('GUELPH', 2021, 105, 14.50, 2.10, 0.00, 18.20, 65.00, 4.20, 0);
    ```

---

### 5.2 Subsystem 2: `analytics`
- **Language**: Python (3.8+)
- **Source Files**: [`analytics/run_forecast.py`](file:///c:/Users/archo/CIS3250/meridian-archaeology/analytics/run_forecast.py), [`orchestrator.py`](file:///c:/Users/archo/CIS3250/meridian-archaeology/analytics/meridian/orchestrator.py), [`deck.py`](file:///c:/Users/archo/CIS3250/meridian-archaeology/analytics/meridian/deck.py), [`parse.py`](file:///c:/Users/archo/CIS3250/meridian-archaeology/analytics/meridian/parse.py), [`scratch.py`](file:///c:/Users/archo/CIS3250/meridian-archaeology/analytics/meridian/scratch.py), [`store.py`](file:///c:/Users/archo/CIS3250/meridian-archaeology/analytics/meridian/store.py).
- **How Control is Passed**:
  - Entry point: `python3 analytics/run_forecast.py --db data/meridian.db`.
  - In `orchestrator.py` (`NightshiftRun.__enter__`), creates an isolated scratch directory using `tempfile.mkdtemp()`.
  - Passes control to `model` by spawning the Fortran binary as a synchronous child process:
    ```python
    proc = subprocess.run([binary], cwd=scratch.path, capture_output=True, text=True, timeout=120)
    ```
    The Python thread blocks until `cropmod` terminates.
  - Upon return, verifies `proc.returncode == 0` and proceeds to `_absorb()` to parse the output and persist to SQLite.
- **How Data is Passed**:
  - **Ingress from SQLite**: Uses Python `sqlite3` connection to query `reading` rows.
  - **Handoff to Model**: Formats and writes `MERIDIAN.DAT` on disk in the scratch directory.
  - **Handoff from Model**: Reads `MERIDIAN.OUT` from the scratch directory. In `parse.py`, uses fixed column slicing (`COLS = [(0, 4), (5, 12), (13, 20), (21, 28), (29, 36), (37, 44)]`) to prevent character overlap bugs when values overflow into adjacent fields (MRD-143).
  - **Egress to SQLite**: Inserts parsed and summarized data into `forecast` and `forecast_daily` tables.

---

### 5.3 Subsystem 3: `model` (`cropmod`)
- **Language**: FORTRAN (FORTRAN 77 driver `cropmod.f`, `waterbal.f`; Fortran 90 modules `canopy.f90`, `growth.f90`, `params.f90`, `partition.f90`, `pet.f90`, `phenology.f90`, `runoff.f90`, `snow.f90`, `soilwat.f90`, `weather.f90`).
- **Source Files**: [`model/cropmod.f`](file:///c:/Users/archo/CIS3250/meridian-archaeology/model/cropmod.f), [`model/waterbal.f`](file:///c:/Users/archo/CIS3250/meridian-archaeology/model/waterbal.f), [`model/growth.f90`](file:///c:/Users/archo/CIS3250/meridian-archaeology/model/growth.f90).
- **How Control is Passed**:
  - Executed by OS process manager as child process of `analytics`.
  - Subroutine `WATBAL` executes snowpack accumulation/melt, runoff, Priestley-Taylor PET, and cascading 5-layer soil water infiltration.
  - Subroutine `GROWTH` executes phenology thermal time accumulation, canopy expansion/senescence, light interception, dry-matter partitioning, and grain yield accumulation.
  - Terminates execution via Fortran `STOP`, returning status code 0 to the parent OS process.
- **How Data is Passed**:
  - **Ingress (`MERIDIAN.DAT`)**:
    - Record 1 (`A8, I4, F8.3`): Station ID, Calendar Year, Latitude.
    - Record 2 (`F8.2, F8.2`): Available Water Capacity (`AWC` fraction), Rooting Depth (mm).
    - Daily Records (`I3, F6.1, F6.1, F6.1, F6.2`): DOY, TMAX, TMIN, RAIN, SRAD.
    - Terminator: `" -1"` in DOY column.
  - **Egress (`MERIDIAN.OUT`)**:
    - Line 1: `MERIDIAN CROPMOD V2.3  STN=GUELPH    YEAR=2021  NDAYS= 180`
    - Line 2: ` DOY      SW      ET   DRAIN    BIOM     LAI`
    - Daily records (`I4, 1X, F7.2, 1X, F7.3, 1X, F7.3, 1X, F7.1, 1X, F7.3`): DOY, Soil Water (mm), Evapotranspiration (mm), Drainage (mm), Biomass (g/m²), Leaf Area Index.
    - Final Line: `YIELD     6.425` (grain yield in t/ha).

---

### 5.4 Subsystem 4: `services`
- **Language**: Java (JDK 8+ standard library, `com.sun.net.httpserver.HttpServer`)
- **Source Files**: [`services/src/ca/meridian/api/ApiServer.java`](file:///c:/Users/archo/CIS3250/meridian-archaeology/services/src/ca/meridian/api/ApiServer.java), [`Db.java`](file:///c:/Users/archo/CIS3250/meridian-archaeology/services/src/ca/meridian/db/Db.java), [`StationHandler.java`](file:///c:/Users/archo/CIS3250/meridian-archaeology/services/src/ca/meridian/api/StationHandler.java), [`ForecastHandler.java`](file:///c:/Users/archo/CIS3250/meridian-archaeology/services/src/ca/meridian/api/ForecastHandler.java), [`SeriesHandler.java`](file:///c:/Users/archo/CIS3250/meridian-archaeology/services/src/ca/meridian/api/SeriesHandler.java).
- **How Control is Passed**:
  - Long-running HTTP service daemon listening on port 8081. Handlers run concurrently on a fixed thread pool of 8 worker threads (`Executors.newFixedThreadPool(8)`).
  - To fulfill database queries, `Db.java` spawns a child process:
    ```java
    ProcessBuilder pb = new ProcessBuilder("sqlite3", "-json", dbPath, sql);
    ```
    It waits synchronously with `p.waitFor()` and reads the JSON stdout output.
- **How Data is Passed**:
  - **Ingress**: HTTP GET requests (`/api/stations`, `/api/forecast?station=...`, `/api/series?station=...&year=...`, `/api/health`).
  - **Egress**: Streams UTF-8 JSON payloads with `Access-Control-Allow-Origin: *` headers over TCP sockets:
    ```json
    [{"doy":105,"sw":182.40,"et":2.150,"drain":0.000,"biom":120.5,"lai":0.450}]
    ```

---

### 5.5 Subsystem 5: `dashboard`
- **Language**: JavaScript (ES5 / browser vanilla JS + jQuery), HTML5, CSS3, served by Node.js (`server.js`)
- **Source Files**: [`dashboard/server.js`](file:///c:/Users/archo/CIS3250/meridian-archaeology/dashboard/server.js), [`js/app.js`](file:///c:/Users/archo/CIS3250/meridian-archaeology/dashboard/js/app.js), [`js/api.js`](file:///c:/Users/archo/CIS3250/meridian-archaeology/dashboard/js/api.js), [`js/charts/barchart.js`](file:///c:/Users/archo/CIS3250/meridian-archaeology/dashboard/js/charts/barchart.js).
- **How Control is Passed**:
  - `server.js` serves static assets on HTTP port 8080.
  - In the browser, control is event-driven via `$(window).on('hashchange', onHashChange)` in `app.js`.
  - Asynchronously requests data from `services` using `jQuery.ajax()` inside `api.js`.
  - Polling timer runs every 15 seconds to check `/api/health`.
- **How Data is Passed**:
  - **Ingress**: Consumes JSON from Java API.
  - **Client-Side Derivation**: Computes anomaly suspect flags (`deriveFlags`) by evaluating thresholds (`sw`, `et`, `drain`, `lai`, `biom` drop).
  - **Egress**: Renders HTML tables and custom DOM-based bar charts (positioning `<div>` elements, since no charting library was bundled; MRD-181).

---

### 5.6 Subsystem 6: `ui-next`
- **Language**: TypeScript, React 18, CSS Modules (bundled with Vite 5)
- **Source Files**: [`ui-next/src/api/client.ts`](file:///c:/Users/archo/CIS3250/meridian-archaeology/ui-next/src/api/client.ts), [`src/hooks/useAsync.ts`](file:///c:/Users/archo/CIS3250/meridian-archaeology/ui-next/src/hooks/useAsync.ts), [`src/screens/ForecastScreen.tsx`](file:///c:/Users/archo/CIS3250/meridian-archaeology/ui-next/src/screens/ForecastScreen.tsx), [`src/components/SeriesChart.tsx`](file:///c:/Users/archo/CIS3250/meridian-archaeology/ui-next/src/components/SeriesChart.tsx).
- **How Control is Passed**:
  - Modern Single Page Application (SPA) driven by React Router 6.
  - Custom data hooks (`useStations`, `useForecast`, `useSeries`) manage async lifecycle and `AbortSignal` cancellation.
  - State synchronized with URL search parameters (`?station=GUELPH&year=2021`) allowing deep-linking.
- **How Data is Passed**:
  - **Ingress**: Strongly-typed `fetch()` calls in `client.ts` (`StationRow`, `ForecastRow`, `SeriesRow`).
  - **Egress**: Renders responsive UI components and a hand-crafted multi-channel SVG chart (`SeriesChart.tsx`) plotting Soil Water, ET, Drainage, Biomass, and LAI curves over DOY; enables CSV file download on the client.

---

### 5.7 Subsystem 7: `reporting`
- **Language**: Python (3.8+ / CLI with dataclasses)
- **Source Files**: [`reporting/reporting/cli.py`](file:///c:/Users/archo/CIS3250/meridian-archaeology/reporting/reporting/cli.py), [`reporting/reporting/db.py`](file:///c:/Users/archo/CIS3250/meridian-archaeology/reporting/reporting/db.py), [`reporting/reporting/format/layout.py`](file:///c:/Users/archo/CIS3250/meridian-archaeology/reporting/reporting/format/layout.py), [`reporting/reporting/reports/season.py`](file:///c:/Users/archo/CIS3250/meridian-archaeology/reporting/reporting/reports/season.py).
- **How Control is Passed**:
  - Invoked from CLI: `python -m reporting <report> [options]`.
  - Opens the SQLite database in strict **read-only mode** (`file:<path>?mode=ro`, `uri=True`), ensuring no write-lock interference with the batch pipeline.
  - Generates reports synchronously and writes output to `sys.stdout` or an output file (`-o`).
- **How Data is Passed**:
  - **Ingress**: Read-only SQL queries (`SELECT ...`) mapped into typed frozen dataclasses (`Reading`, `Forecast`, `DailyRow`).
  - **Egress**: Formats data into column-exact 80-column (greenbar) or 132-column (wide fan-fold) plain text with page headers, footers, and form-feed characters (`\f`), right-justifying numeric values and replacing overflow numbers with asterisks (`*****`), or CSV output.

---

## 6. Standalone PlantUML Diagrams

The repository includes standalone PlantUML diagram sources ready for rendering:
- Component Diagram: [`docs/uml/subsystems_component.puml`](file:///c:/Users/archo/CIS3250/meridian-archaeology/docs/uml/subsystems_component.puml)
- Sequence Diagram: [`docs/uml/pipeline_sequence.puml`](file:///c:/Users/archo/CIS3250/meridian-archaeology/docs/uml/pipeline_sequence.puml)

To render these diagrams with PlantUML:
```bash
plantuml docs/uml/subsystems_component.puml
plantuml docs/uml/pipeline_sequence.puml
```
This produces high-resolution PNG/SVG vector diagrams illustrating the seven subsystems.

---

## 7. Programmatic Verification Tool

To programmatically verify and demonstrate the exact contract execution and control/data handoffs across all seven subsystems, see:
[`tools/trace_subsystems.py`](file:///c:/Users/archo/CIS3250/meridian-archaeology/tools/trace_subsystems.py).

Run the verification tool with:
```bash
python tools/trace_subsystems.py
```
