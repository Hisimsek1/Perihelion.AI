# Perihelion.ai — Detaylı Sistem Mimarisi

Bu belge, Perihelion.ai sisteminin bileşenlerini, veri akışını, ve katmanlı mimarisini detaylı olarak gösterir.

---

## 1. Üst Düzey Sistem Mimarisi

```mermaid
graph TB
    subgraph External["Dış Kaynaklar"]
        NOAA["NOAA SWPC<br/>X-ray JSON API"]
    end
    
    subgraph DataLayer["Veri Katmanı"]
        Fetch["fetch.py<br/>Veri Çekme"]
        RawData["data/raw/<br/>xray_flux.csv"]
    end
    
    subgraph ProcessingLayer["İşleme Katmanı"]
        Features["build_features.py<br/>Özellik Mühendisliği"]
        ProcessedData["data/processed/<br/>features.csv"]
    end
    
    subgraph ModelLayer["Model Katmanı"]
        Train["main.py<br/>Model Training"]
        Model["models/<br/>storm_lgbm.joblib"]
    end
    
    subgraph PredictionLayer["Tahmin Servisi"]
        PredictModule["predict.py<br/>Tahmin işleme"]
    end
    
    subgraph APILayer["API Katmanı"]
        API["Flask API<br/>app.py"]
        Health["GET /health"]
        Predict["GET /api/predict"]
        Mode["POST /api/mode"]
    end
    
    subgraph FrontendLayer["Frontend Katmanı"]
        HTML["index.html<br/>DOM Yapısı"]
        JS["main.js<br/>Lojik & Render"]
        CSS["styles.css<br/>Stil & Tema"]
    end
    
    NOAA --> Fetch
    Fetch --> RawData
    RawData --> Features
    Features --> ProcessedData
    ProcessedData --> Train
    Train --> Model
    Model --> PredictModule
    PredictModule --> API
    API --> Health
    API --> Predict
    API --> Mode
    API --> HTML
    HTML --> JS
    HTML --> CSS
    JS -.->|Fetch| Predict
    JS -.->|Fetch| Mode
    JS -.->|Poll| Health
```

---

## 2. Veri İşlem Hattı (Data Pipeline)

```mermaid
graph LR
    A["NOAA API<br/>JSON"] 
    B["urllib.request<br/>& SSL"]
    C["JSON Parse<br/>& DataFrame"]
    D["DateTime<br/>Conversion"]
    E["Raw CSV<br/>~2-5 MB"]
    F["Load CSV"]
    G["Lag Features<br/>lag1,3,6"]
    H["Ratio & Diff<br/>flux_ratio, diff"]
    I["Rolling Stats<br/>mean_3, mean_6"]
    J["Label Creation<br/>threshold"]
    K["Drop NaN<br/>Rows"]
    L["Processed CSV<br/>~50 KB"]
    
    A -->|HTTPS| B
    B --> C
    C --> D
    D --> E
    E --> F
    F --> G
    G --> H
    H --> I
    I --> J
    J --> K
    K --> L
    
    style A fill:#1e88e5,stroke:#0d47a1,color:#fff
    style E fill:#ffa726,stroke:#f57c00,color:#fff
    style L fill:#66bb6a,stroke:#43a047,color:#fff
```

### Ayrıntılı Adımlar:

**1. Veri Çekme (fetch.py)**
```
Input:  NOAA SWPC URL
        ↓
Process: • SSL/TLS bağlantısı (certifi)
         • JSON indir
         • Parse et
         • Pandas DataFrame
         • time_tag → UTC datetime
        ↓
Output:  data/raw/xray_flux.csv
         (100+ gözlem)
```

**2. Özellik Üretimi (build_features.py)**
```
Input:   data/raw/xray_flux.csv
         ↓
Process: • flux_lag1 = shift(1)
         • flux_lag3 = shift(3)
         • flux_lag6 = shift(6)
         • flux_ratio = lag1 / (lag3 + 1e-9)
         • rolling_mean_3 = rolling(3).mean()
         • rolling_mean_6 = rolling(6).mean()
         • flux_diff = diff()
         • Label: future_flux > threshold
        ↓
Output:  data/processed/features.csv
         (85 gözlem, balanced labels)
```

**3. Model Training (main.py)**
```
Input:   data/processed/features.csv
         ↓
Process: • X/y split
         • Train/Test: 80/20 stratified
         • LightGBM fitting
         • Classification report
         • Joblib serialize
        ↓
Output:  models/storm_lgbm.joblib
         (~50 KB)
```

---

## 3. Model Katmanı (ML Component)

```mermaid
graph TB
    subgraph Training["Eğitim Aşaması"]
        Features["Features<br/>7 sütun"]
        Split["Train/Test<br/>80/20 Stratified"]
        LGB["LightGBM<br/>200 trees"]
        Eval["Evaluation<br/>F1, Precision, Recall"]
        Save["Joblib<br/>Serialize"]
    end
    
    subgraph Inference["Tahmin Aşaması"]
        Load["Load Model"]
        Validate["Feature Check"]
        Predict["Predict Proba"]
        Risk["Risk Status<br/>HIGH/MEDIUM/LOW"]
    end
    
    Features --> Split
    Split --> LGB
    LGB --> Eval
    Eval --> Save
    
    Save --> Load
    Load --> Validate
    Validate --> Predict
    Predict --> Risk
    
    style Training fill:#bbdefb,stroke:#1976d2
    style Inference fill:#c8e6c9,stroke:#388e3c
```

### Hiperparametreler:

```python
{
    'class_weight': 'balanced',    # ← Sınıf dengesizliği
    'n_estimators': 200,           # ← Ağaç sayısı
    'max_depth': 8,                # ← Max derinlik
    'num_leaves': 48,              # ← Yaprak sayısı
    'learning_rate': 0.05,         # ← Öğrenme oranı
    'subsample': 0.9,              # ← Veri sub-sample
    'colsample_bytree': 0.9,       # ← Özellik sub-sample
    'reg_lambda': 0.1,             # ← L2 regularization
}
```

### Risk Scoring:

```
Probability → Status
P > 0.6  → HIGH RISK   (Kritik)
P > 0.3  → MEDIUM RISK (Uyarı)
P ≤ 0.3  → LOW RISK    (Normal)
```

---

## 4. Backend Mimarisi (Flask API)

```mermaid
graph TB
    subgraph Server["Flask Server<br/>Port 5050"]
        CORS["CORS Enabled"]
        
        subgraph Routes["API Routes"]
            R1["GET /health"]
            R2["GET /api/predict"]
            R3["POST /api/mode"]
            R4["GET / (static)"]
        end
        
        subgraph State["Global State"]
            MODE["MODE<br/>calm/storm"]
            WEIGHT["storm_weight<br/>[0.0, 1.0]"]
            DATA["Telemetry Data<br/>wind, kp, bz..."]
        end
        
        subgraph Logic["İşlem Modülleri"]
            SIM["Simulasyon<br/>MetricCalc"]
            RAMP["Smooth Ramp<br/>90 sn geçiş"]
            LOG["Logger<br/>Requests"]
        end
    end
    
    CORS --> Routes
    Routes --> State
    State --> Logic
    
    R2 -->|calls| SIM
    SIM -->|updates| DATA
    DATA -->|smooth| RAMP
    RAMP -->|logs| LOG
    
    style Server fill:#fff3e0,stroke:#e65100
    style Routes fill:#f0f4c3,stroke:#827717
    style State fill:#e1f5fe,stroke:#0277bd
    style Logic fill:#f3e5f5,stroke:#512da8
```

### Endpoints Detaylı:

**1. GET /health**
```json
Request:  GET http://127.0.0.1:5050/health
Response: {
  "status": "ok",
  "mode": "calm",
  "intensity": 0.1234
}
Status:   200 OK
```

**2. GET /api/predict** (Demo Telemetri)
```json
Request:  GET /api/predict
Response: {
  "time": "2026-03-29T12:00:00Z",
  "windSpeed": 323.5,
  "protonDensity": 3.2,
  "kpIndex": 1.7,
  "aiPredictionKp": 1.8,
  "bz": 4.3,
  "electronFlux": 1780
}
Logic:    • Smooth ramp active
          • Simulated values based on MODE
          • ~50ms response time
```

**3. POST /api/mode**
```json
Request:  POST /api/mode
          { "mode": "storm" }

Response: { "ok": true, "mode": "storm" }

Behavior: • MODE global değişkeni güncelle
          • storm_weight smooth geçiş başla
          • 90 sn içinde calm→storm
```

### State Machine:

```mermaid
stateDiagram-v2
    [*] --> Calm
    
    Calm -->|POST /api/mode<br/>{"mode":"storm"}| Ramping_Up
    Ramping_Up -->|90 sec elapsed| Storm
    Storm -->|POST /api/mode<br/>{"mode":"calm"}| Ramping_Down
    Ramping_Down -->|90 sec elapsed| Calm
    
    Ramping_Up: Smooth ramp<br/>0.0 → 1.0
    Ramping_Down: Smooth ramp<br/>1.0 → 0.0
    
    note right of Ramping_Up
        Telemetri değerleri
        gradually değişir
        (lerp kullanılır)
    end note
```

---

## 5. Frontend Mimarisi (Web UI)

```mermaid
graph TB
    subgraph Browser["Tarayıcı"]
        DOM["HTML DOM"]
        
        subgraph Pages["Sayfa Bölümleri"]
            Header["Header<br/>Logo + Status"]
            PanelL["Panel Left<br/>Telemetri Kartları"]
            Scene["Scene Area<br/>3D Render"]
            PanelR["Panel Right<br/>Risk Analysis"]
            Timeline["Timeline<br/>Event Flow"]
        end
        
        subgraph Logic["JavaScript Logic"]
            State["State Mgmt<br/>sim object"]
            Poll["Polling<br/>1 sec interval"]
            Render["Render Loop<br/>requestAnimFrame"]
        end
        
        subgraph Graphics["Grafik Kütüphaneleri"]
            Chart["Chart.js<br/>Telemetri"]
            Three["Three.js<br/>3D Scene"]
            CSS["CSS3<br/>Animations"]
        end
    end
    
    DOM --> Pages
    Pages --> Logic
    Logic --> Graphics
    
    Poll -->|fetch| APIPredict["/api/predict"]
    Poll -->|fetch| APIMode["/api/mode"]
    Render -->|update| Chart
    Render -->|update| Three
    Render -->|update| CSS
    
    style Browser fill:#e8f5e9,stroke:#2e7d32
    style Graphics fill:#f1f8e9,stroke:#558b2f
```

### UI Panel Yapısı:

```
┌─────────────────────────────────────────────────────┐
│  HEADER: Logo + Status Indicator                   │
├─────┬─────────────────────────────────┬─────────────┤
│     │                                 │             │
│ L   │   3D Scene                      │ R           │
│     │   (Scene Area:                  │             │
│ PANEL│  Three.js Render)              │ PANEL       │
│     │   [800x650]                     │             │
│     │                                 │             │
├─────┴─────────────────────────────────┴─────────────┤
│  TIMELINE: Event Progression (w/ progress bar)      │
└─────────────────────────────────────────────────────┘

Panel Left (438px):              Panel Right (438px):
├ Wind Speed (gauge)            ├ Alert Card
├ Proton Density (gauge)        ├ Kp Index (scale 1-9)
├ Mini-stats (Bz, e-flux)       ├ Impact List
├ Telemetry Chart               └ Controls
└ Connect Button
```

### Polling Döngüsü:

```mermaid
sequenceDiagram
    autonumber
    participant Browser
    participant API
    participant Model
    
    loop Every 1 second
        Browser->>API: GET /api/predict
        API->>Model: Load predictions (cached)
        Model-->>API: telemetry payload
        API-->>Browser: JSON response
        Browser->>Browser: applyDataPoint(data)
        Browser->>Browser: updateUI()
        Browser->>Browser: Render chart & 3D
    end
```

---

## 6. Veri Akış Şeması

```mermaid
graph LR
    NOAA["NOAA API"]
    
    NOAA -->|6h intervals| Fetch["fetch.py"]
    Fetch -->|urllib.request| RawCSV["raw CSV<br/>~100 rows"]
    
    RawCSV -->|Read| Features["features.py"]
    Features -->|7 derived<br/>features| ProcessedCSV["processed CSV<br/>~85 rows"]
    
    ProcessedCSV -->|80/20 split| Train["main.py"]
    Train -->|LightGBM| Model["joblib"]
    
    Model -->|predict.py| PredService["Predictions"]
    
    PredService -->|JSON| API["Flask API"]
    API -->|REST endpoint| Frontend["Dashboard"]
    
    Frontend -->|Chart.js<br/>Three.js| Browser["Tarayıcı"]
    
    style NOAA fill:#1e88e5,stroke:#0d47a1,color:#fff
    style Model fill:#7cb342,stroke:#558b2f,color:#fff
    style Browser fill:#ff6f00,stroke:#e65100,color:#fff
```

---

## 7. Dosya Yapısı ve İlişkileri

```mermaid
graph TB
    Root["."]
    
    Root -->|Giriş| Main["main.py<br/>(Training)"]
    Root -->|Config| Req["requirements.txt"]
    Root -->|Komutlar| Make["Makefile"]
    Root -->|Doküman| README["README.md"]
    
    Root -->|src/| SrcRoot["src/"]
    Root -->|frontend/| FrontRoot["frontend/"]
    Root -->|docs/| DocsRoot["docs/"]
    
    SrcRoot -->|api/| ApiDir["app.py<br/>predict.py"]
    SrcRoot -->|data/| DataDir["fetch.py<br/>preprocess.py"]
    SrcRoot -->|features/| FeatDir["build_features.py"]
    
    FrontRoot -->|HTML| HTML["index.html"]
    FrontRoot -->|JS| JS["main.js"]
    FrontRoot -->|CSS| CSS["styles.css"]
    FrontRoot -->|logo/| Logo["perihelionlogo.svg"]
    FrontRoot -->|assets/| Assets["8k_*.jpg"]
    
    DocsRoot -->|Rapor| Rapor["RAPOR.md"]
    DocsRoot -->|TECHNICAL| Tech["TECHNICAL_REPORT.md"]
    DocsRoot -->|Index| Index["INDEX.md"]
    DocsRoot -->|Architecture| Arch["ARCHITECTURE.md"]
    
    ApiDir -->|imports| PredDir["predict.py"]
    PredDir -->|loads| Model[["models/storm_lgbm.joblib"]]
    
    JS -->|calls| ApiDir
    DataDir -->|creates| Raw["data/raw/<br/>xray_flux.csv"]
    FeatDir -->|creates| Proc["data/processed/<br/>features.csv"]
    Raw -->|uses| FeatDir
    Proc -->|uses| Main
    
    style Root fill:#f5f5f5,stroke:#000
    style SrcRoot fill:#bbdefb,stroke:#1976d2
    style FrontRoot fill:#c8e6c9,stroke:#388e3c
    style DocsRoot fill:#ffe0b2,stroke:#e65100
```

---

## 8. Dağıtım Ortamı (Deployment)

```mermaid
graph TB
    subgraph Dev["Geliştirme"]
        DevPy["Python 3.10+"]
        DevPip["pip install -r requirements.txt"]
        DevRun["python src/api/app.py"]
    end
    
    subgraph Prod["Üretim"]
        Docker["Docker Container"]
        Gunicorn["Gunicorn<br/>Multi-worker"]
        Nginx["Nginx<br/>Reverse Proxy"]
        Redis["Redis<br/>Caching"]
        Postgres["PostgreSQL<br/>Data"]
    end
    
    subgraph Monitor["İzleme"]
        Prom["Prometheus"]
        Grafana["Grafana Dashboard"]
        Logs["Centralized Logs"]
    end
    
    Dev -->|Containerize| Docker
    Docker -->|Orchestrate| Gunicorn
    Gunicorn -->|Route| Nginx
    Nginx -->|Cache layer| Redis
    Redis -->|Data| Postgres
    
    Gunicorn -->|Metrics| Prom
    Prom -->|Visualize| Grafana
    Gunicorn -->|Logs| Logs
    
    style Dev fill:#f3e5f5,stroke:#512da8,color:#000
    style Prod fill:#ffccbc,stroke:#d84315,color:#000
    style Monitor fill:#fff9c4,stroke:#f57f17,color:#000
```

---

## 9. İletişim Protokolleri

```mermaid
sequenceDiagram
    participant Client as Client<br/>(Browser)
    participant Backend as Backend<br/>(Flask)
    participant ML as ML Model
    participant NOAA as NOAA API

    Client->>Backend: GET /health
    Backend-->>Client: {"status":"ok",...}
    
    Client->>Backend: GET /api/predict
    Backend->>ML: Load cached model
    ML-->>Backend: Predictions ready
    Backend-->>Client: {"windSpeed":...}
    
    Client->>Backend: POST /api/mode<br/>{"mode":"storm"}
    Backend->>Backend: Update MODE global<br/>Start ramp
    Backend-->>Client: {"ok":true,"mode":"storm"}
    
    Backend->>NOAA: (background) Periodic fetch
    NOAA-->>Backend: X-ray flux JSON
    Backend->>Backend: Process & cache
```

---

## 10. Performans Katmanlı Mimarisi

```mermaid
graph TB
    subgraph Perf["Performans Hedefleri"]
        API_P["API Latency<br/>< 50ms"]
        FE_P["Frontend Render<br/>60 FPS"]
        Poll_P["Polling Interval<br/>1 sec"]
        Train_P["Training<br/>< 20 sec"]
    end
    
    subgraph Scale["Ölçeklenebilirlik"]
        Cache["In-Memory Cache<br/>(Model)"]
        Queue["Job Queue<br/>(long tasks)"]
        DB["Database<br/>(persistence)"]
        Load["Load Balancer<br/>(requests)"]
    end
    
    API_P -->|Achieved by| Cache
    FE_P -->|Achieved by| RequestAnimFrame["requestAnimationFrame"]
    Poll_P -->|Achieved by| SetInterval["setInterval(1000ms)"]
    Train_P -->|Achieved by| Parallel["Parallel processing"]
    
    style Perf fill:#c8e6c9,stroke:#388e3c
    style Scale fill:#fff3e0,stroke:#e65100
```

---

## Özet: Sistem Mimarisi Prensipierleri

| Prensip | Uygulama | Fayda |
|---------|----------|-------|
| **Modülerlik** | Bileşenler bağımsız | Test ve geliştirme kolay |
| **Veri Yalıtımı** | Ayrı klasörler (raw/processed/models) | Veri akışı izlenebilir |
| **Şeffaflık** | Demo vs. Gerçek model ayrımı | Bilimsel bütünlük |
| **Ölçeklenebilirlik** | Stateless API, data-driven config | Üretim hazır |
| **CORS Desteği** | Frontend/backend farklı ports | Geliştirme esnekliliği |

---

**Son Güncelleme:** 29 Mart 2026  
**Mimari Versiyon:** 1.0  
**Dil:** Türkçe (Akademik)
