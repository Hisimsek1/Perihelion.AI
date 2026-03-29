# Perihelion.ai Dokümantasyon Indeksi

Projeyle ilgili tüm dökümanlar aşağıda listelenmiştir.

## 1. Başlangıç

- **[README.md](../README.md)** - Proje genel bakışı, kurulum, çalıştırma adımları
  - Sistem mimarisi diyagramı
  - Klasör yapısı
  - Veri hattı açıklaması
  - API özeti
  - Sorun giderme

## 2. Teknik Dokümantasyon

### [TECHNICAL_REPORT.md](TECHNICAL_REPORT.md) — Kapsamlı Teknik Rapor

Aşağıdaki başlıkları içerir:

1. **Yönetici Özeti** - Proje hedeflerinin kısa özeti
2. **Giriş ve Arka Plan** - Uzay hava ve operasyon koşulları
3. **Sistem Mimarisi** - Bileşen diyagramı ve tasarım prensipleri
4. **Veri İşleme Hattı**
   - Veri çekme (`src/data/fetch.py`)
   - Özellik mühendisliği (`src/features/build_features.py`)
   - Veri kalitesi metrikleri
5. **Makine Öğrenmesi Modeli**
   - LightGBM hiperparametreleri
   - Train/test ayrımı
   - Değerlendirme metrikleri
   - Tahmin fonksiyonları
6. **Backend Mimarisi**
   - Flask yapısı ve state yönetimi
   - API uç noktaları (endpoint) detayları
   - Telemetri simülasyonu
   - Logging
7. **Frontend Mimarisi**
   - Teknoloji stack'i
   - JavaScript modülleri
   - Chart.js telemetri grafiği
   - CSS tema ve efektler
8. **API Spesifikasyonu** - OpenAPI 3.0 taslağı
9. **Performans ve Ölçeklenebilirlik**
   - Aşama bazlı performans metrikleri
   - API throughput
   - Ölçeklenebilirlik sınırları
10. **Güvenlik**
    - Veri güvenliği
    - API güvenliği
    - Kod güvenliği
11. **Hata Yönetimi ve Logging**
12. **Test Stratejisi**
    - Birim testleri
    - Entegrasyon testleri
    - Veri kalitesi testleri
    - Performans testleri
13. **Dağıtım**
    - Geliştirme ortamı setup
    - Docker
    - CI/CD
    - Sistem gereksinimleri
14. **Bilinen Sınırlamalar**
15. **Gelecek İyileştirmeler** (kısa, orta, uzun vadeli)
16. **Kaynakça**

### [ARCHITECTURE.md](ARCHITECTURE.md) — Detaylı Sistem Mimarisi Diyagramları

Sistem bileşenlerini Mermaid diyagramlarla gösteren kapsamlı mimari dokumen:

- Üst düzey sistem mimarisi
- Veri işlem hattı (pipeline)
- Model katmanı & hyperparametreler
- Backend API mimarisi & state machine
- Frontend UI panel yapısı & polling döngüsü
- Veri akış şeması
- Dosya yapısı ve ilişkileri
- Dağıtım ortamı (Dev vs. Production)
- İletişim protokolleri (sequence diagram)
- Performans katmanlı mimarisi
- Sistem prensipleri tablosu

### [RAPOR.md](RAPOR.md) — Proje Özet Raporu

Jüri / takım iletişimi için kısa özet:

- Problem tanımı
- Çözüm yaklaşımı
- Veri kaynağı
- Model ve değerlendirme
- Demo API vs. gerçek model ayrımı
- Teknik yığın
- Kaynakça

## 3. Kod Yapısı Referansı

```
src/
├── api/
│   ├── app.py          # Flask uygulaması
│   └── predict.py      # Model yükleme ve tahmin
├── data/
│   ├── fetch.py        # NOAA API veri çekme
│   └── preprocess.py   # Veri temizleme (şu anda boş)
└── features/
    └── build_features.py  # Özellik mühendisliği

frontend/
├── index.html          # DOM yapısı
├── main.js             # Lojik ve render
├── styles.css          # Stil ve tema
├── logo/               # Logo dosyaları
└── assets/             # Tekstürler (3D)

docs/
├── INDEX.md            # Bu dosya
├── TECHNICAL_REPORT.md # Detaylı teknik rapor
└── RAPOR.md            # Özet rapor

main.py                # Model training
requirements.txt       # Python bağımlılıkları
Makefile              # Komut kısayolları
```

## 4. Dosya Açıklamaları

| Dosya | Amaç | İçerik |
|-------|------|--------|
| `src/data/fetch.py` | NOAA'dan veri çekme | JSON indirme, CSV kaydetme |
| `src/features/build_features.py` | Özellik üretimi | Lag, rolling, ratio, etiket labelleme |
| `main.py` | Model training | LightGBM eğitimi, joblib save |
| `src/api/predict.py` | Tahmin servisi | Model yükleme, kapsülleme |
| `src/api/app.py` | Backend | Flask, CORS, telemetri endpoints |
| `frontend/index.html` | Desktop yapısı | Paneller, kartlar, timeline |
| `frontend/main.js` | Frontend lojik | API polling, Chart.js, UI render |
| `frontend/styles.css` | Stil | CSS Grid, backdrop blur, animasyonlar |

## 5. Hızlı Başlangıç Komutları

```bash
# Veri çekme
python -m src.data.fetch

# Özellik üretimi
python src/features/build_features.py

# Model training
python main.py

# API başlatma
python src/api/app.py
# http://127.0.0.1:5050
```

## 6. Sık Sorulan Sorular

**S: Teknik rapor ne kadar detaylı?**  
C: Çok detaylı! Sistem mimarisi, hiperparametreler, API sözleşmesi, güvenlik, performans, test stratejisi ve dağıtım adımlarını kapsar.

**S: Frontend ve backend uyumlu mu?**  
C: Evet. Frontend otomatik olarak backend URL'sini bulur (localhost:5050, LAN IP, veya custom URL).

**S: Model güvenilir mi?**  
C: Demo amaçlıdır. Çok sınırlı veri (~100 gözlem) ve yapay eşikle eğitilmiş. Üretim için daha fazla veri ve fiziksel hedef gerekir.

**S: Ölçeklenebilir mi?**  
C: Şu anda değil. Tek Python prosesi, in-memory state. Gunicorn + Redis ile geliştirilebilir.

## 7. Test Etme

```bash
# Terminal 1: API başlat
python src/api/app.py

# Terminal 2: Sağlık kontrolü
curl http://127.0.0.1:5050/health

# Tarayıcı: Dashboard
http://127.0.0.1:5050
```

## 8. Destek ve Kontakt

- **Sorunlar:** GitHub Issues açın
- **Geri Bildirim:** Pull Request gönderin
- **Dokumanlar:** `docs/` klasöründe tüm dosyalar

---

**Güncelleme Tarihi:** 29 Mart 2026  
**Dokümantasyon Versiyonu:** 1.0
