# SkyPredict — Uçuş Gecikmesi Erken Uyarı ve Finansal Etki Analizi

> 3.3 milyon uçuş kaydını saatlik hava durumu verileriyle birleştirerek uçuş gecikmelerini tahmin eden ve havayoluna sağlayacağı finansal tasarrufu hesaplayan bir makine öğrenmesi projesi.

![Python](https://img.shields.io/badge/Python-3.10+-blue?logo=python&logoColor=white)
![scikit-learn](https://img.shields.io/badge/scikit--learn-1.8-orange?logo=scikit-learn&logoColor=white)
![pandas](https://img.shields.io/badge/pandas-2.0+-green?logo=pandas&logoColor=white)
![License](https://img.shields.io/badge/License-MIT-yellow)

---

## Proje Özeti

Uçuş gecikmeleri havayolları için ciddi bir maliyet kalemidir — tazminat, otel, yemek ve müşteri kaybı. Bu proje, gecikmeleri önceden tahmin ederek proaktif müdahale ile bu maliyetleri azaltmayı hedefler.

Proje kapsamında:

- 3 farklı veri kaynağı (uçuş, hava durumu, havaalanı) birleştirildi
- 5 iş mantığı tabanlı yeni değişken türetildi
- Random Forest modeli ile AUC 0.75 doğruluk elde edildi
- Maliyet/fayda simülasyonu ile **~$1.8M net tasarruf** potansiyeli gösterildi

---

## Temel Sonuçlar

| Metrik | Değer |
|--------|-------|
| Analiz Edilen Uçuş | 3,313,919 |
| Birleştirilen Veri Kaynağı | 3 (uçuş + hava durumu + havaalanı) |
| Türetilen Yeni Özellik | 5 |
| En İyi Model | Random Forest (AUC: 0.75) |
| Optimal Karar Eşiği | 0.28 |
| Net Tasarruf | ~$1.81M |

---

## Proje Mimarisi

```
  Uçuş Verileri          Hava Durumu            Havaalanı Bilgileri
  (5.8M kayıt)       (saatlik: sıcaklık,       (IATA kodu, şehir,
                    rüzgar, nem, basınç)           koordinat)
        │                    │                        │
        └──────────┬─────────┴────────────────────────┘
                   │
            VERİ HARMANLAMA
          (şehir + tarih + saat)
             3.3M kayıt
                   │
          ÖZELLİK MÜHENDİSLİĞİ
          (5 yeni iş değişkeni)
                   │
              MODELLEME
           Random Forest
             AUC: 0.75
                   │
        MALİYET/FAYDA SİMÜLASYONU
         Optimal eşik: 0.28
         Net tasarruf: ~$1.81M
```

---

## Veri Kaynakları

| Kaynak | Boyut | Açıklama |
|--------|-------|----------|
| **Uçuş Verileri** | 5.8M → 3.3M kayıt | ABD Ulaştırma Bakanlığı, 2015 ([Kaggle](https://www.kaggle.com/datasets/usdot/flight-delays)) |
| **Hava Durumu** | 236K saatlik kayıt | NOAA, 27 ABD şehri, 2015 ([Kaggle](https://www.kaggle.com/datasets/selfishgene/historical-hourly-weather-data)) |
| **Havaalanı Bilgileri** | 322 havaalanı | Konum, şehir, eyalet |

Birleştirme yöntemi: Havaalanları IATA kodlarıyla şehirlere eşlenmiş (32 havaalanı → 27 şehir), ardından uçuş ve hava durumu verileri **şehir + tarih + saat** bazında merge edilmiştir.

---

## Özellik Mühendisliği

| # | Özellik | Yöntem | İş Mantığı |
|---|---------|--------|------------|
| 1 | Hava Durumu Şiddet İndeksi | Rüzgar + nem + kötü hava composite skoru | Şiddetli hava gecikme riskini katlıyor |
| 2 | Havaalanı Yoğunluk Oranı | Saatlik uçuş sayısı / havaalanı ortalaması | Yoğun saatlerde pist kapasitesi doluyor |
| 3 | Rota Güvenilirlik Skoru | Rota bazlı tarihsel zamanında kalkış oranı | Bazı rotalar kronik gecikmeli |
| 4 | Kaskad Gecikme Riski | Aynı uçağın önceki bacağındaki gecikme | Gecikme bir sonraki uçuşa sirayet ediyor |
| 5 | Zaman Bazlı Özellikler | Hafta sonu, yoğun saat, döngüsel ay kodlama | Akşam uçuşları gün boyu biriken gecikmeleri taşıyor |

---

## Model Karşılaştırması

| Model | AUC-ROC | F1-Score |
|-------|---------|----------|
| Logistic Regression | 0.720 | 0.434 |
| **Random Forest** | **0.747** | **0.469** |

En etkili özellikler (Feature Importance):

1. **prev_delay** (0.23) — Kaskad gecikme
2. **hour** (0.15) — Kalkış saati
3. **cascade_risk** (0.13) — Önceki uçuş gecikmeli mi
4. **route_ontime_rate** (0.07) — Rota güvenilirliği
5. **temperature** (0.06) — Sıcaklık

---

## Maliyet/Fayda Simülasyonu

Gecikme önceden tahmin edilirse yolcular proaktif rebooking ile yönlendirilir.

**Parametreler:**

| Parametre | Değer | Açıklama |
|-----------|-------|----------|
| Reaktif Maliyet | $500 | Gecikme sonrası tazminat + otel + yemek + imaj kaybı |
| Proaktif Rebooking | $150 | Erken yönlendirme maliyeti |
| Yanlış Alarm | $30 | Gereksiz bildirim maliyeti |

**Neden eşik 0.50 yerine 0.28?**

Bir gecikmeyi kaçırmanın maliyeti ($500) yanlış alarmdan ($30) çok daha yüksek. Bu asimetri nedeniyle daha geniş ağ atıp daha fazla gecikme yakalamak karlı. Optimal eşik bu dengeyi maksimize eder.

**Sonuçlar:**

| Senaryo | Net Tasarruf |
|---------|-------------|
| Model Yok | $0 |
| Standart Eşik (0.50) | ~$1.25M |
| **Optimal Eşik (0.28)** | **~$1.81M** |

---

## Kurulum ve Çalıştırma

### Gereksinimler

```bash
pip install pandas numpy scikit-learn matplotlib seaborn shap
```

### Dosya Yapısı

```
SkyPredict/
├── SkyPredict_Ucus_Gecikmesi_Projesi.ipynb
├── SkyPredict_Yonetici_Ozeti.pdf
├── README.md
└── data/raw/
    ├── flights/
    │   ├── airlines.csv
    │   ├── airports.csv
    │   └── flights_filtered.csv.gz
    └── weather/
        ├── city_attributes.csv.gz
        ├── temperature.csv.gz
        ├── humidity.csv.gz
        ├── wind_speed.csv.gz
        ├── pressure.csv.gz
        └── weather_description.csv.gz
```

### Çalıştırma

```bash
jupyter notebook SkyPredict_Ucus_Gecikmesi_Projesi.ipynb
```

Hücreleri sırayla çalıştırın. Notebook tüm veri yükleme, birleştirme, özellik mühendisliği, modelleme ve maliyet/fayda analizini kapsar. Pandas `.csv.gz` dosyalarını doğrudan okuyabilir.

---

## Teslim Edilenler

| Dosya | Açıklama |
|-------|----------|
| `SkyPredict_Ucus_Gecikmesi_Projesi.ipynb` | Ana analiz notebook'u (8 bölüm) |
| `SkyPredict_Yonetici_Ozeti.pdf` | Yönetici özeti raporu (5 sayfa) |
| `README.md` | Proje açıklama dosyası |

---

## Lisans

Bu proje MIT Lisansı ile lisanslanmıştır.
