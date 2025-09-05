# Chicago Taxi + Weather Data Pipeline

**Cél**: Chicago taxi fuvarozási adatok és időjárási viszonyok összekapcsolása AWS-re optimalizált data engineering pipeline-nal.

## 📊 Projekt Áttekintés

Ez a projekt a Chicago közlekedési és időjárási adatok közötti összefüggéseket vizsgálja egy komplex ETL pipeline segítségével.

### Adatforrások
- **Chicago Open Data API**: `ajtu-isnz` dataset - taxi fuvarok GPS koordinátákkal
- **Open-Meteo API**: Óránkénti időjárási adatok Chicago koordinátáira (41.85, -87.65)

### Kimenetek
- **Fact table**: Normalizált taxi+weather adatok
- **Master tables**: payment_type_master, company_master
- **AWS-optimalizált**: Memória ~15% csökkentés adattípus optimalizálással

## 🔄 Pipeline Folyamata

### 1. Adatgyűjtés
```
04_get_weather_data.ipynb → Időjárási adatok (hőmérséklet, szél, csapadék)
03_get_taxi_data_v2.ipynb → Taxi fuvarok (trip_id, időpontok, helyek, árak)
```

### 2. Adatintegráció & Modeling  
```
06_chicago_data_to_mapping.ipynb → Taxi+weather join + normalizálás
```

**Kulcs logika**:
- Taxi `trip_start_timestamp` → órára kerekítés → weather `datetime` join
- Duplikált/hibás oszlopok eltávolítása
- Master táblák létrehozása (payment_type, company)
- AWS storage optimalizálás (int8, float32 konverziók)

### 3. Output Struktúra
```
csv/processed/
├── chicago_taxi_weather_fact.csv    # Fő adattábla
├── payment_type_master.csv          # Fizetési módok
└── company_master.csv               # Taxi cégek
```

## 💡 Technikai Megoldások

### Időjárási Korreláció
```python
# Taxi időpontok órára kerekítése weather join-hoz
taxi_trips["datetime_for_weather"] = pd.to_datetime(
    taxi_trips["trip_start_timestamp"]
).dt.floor("h")
```

### AWS Optimalizálás
```python
# Memória csökkentés adattípus optimalizálással
optimized_types = {
    "pickup_community_area_id": "int8",     # 255-ig elegendő
    "fare": "float32",                      # dupla pontosság nem kell
    "temperature": "float32"                # időjárási adatok
}
```

### Normalizálás
- **Denormalized**: "Credit Card", "Sun Taxi" stringek minden sorban
- **Normalized**: payment_type_id=1, company_id=7 → tárhelymegtakarítás

## 🚀 Használat

### Futtatási Sorrend
1. `04_get_weather_data.ipynb` - Időjárás letöltés
2. `03_get_taxi_data_v2.ipynb` - Taxi adatok letöltés  
3. `06_chicago_data_to_mapping.ipynb` - Integráció & export

### Környezeti Változók
```bash
export CHICAGO_API_TOKEN="your_token_here"  # Opcionális, de ajánlott
```

### AWS Deployment Ready
- **Lambda Layers**: `packages_for_lambda/`, `pandas_layer/` előkészítve
- **CSV Output**: S3 upload-ra kész formátum
- **Memória-optimalizált**: Költséghatékony Athena/Redshift query-k

## 📈 Business Value

### Elemzési Lehetőségek
- Időjárás hatása taxi kereslet-re
- Csapadékos vs száraz napok fuvarszáma  
- Hőmérséklet vs bevétel korreláció
- Szeles időben utazási mintázatok

### AWS Architektúra
```
API Sources → Lambda ETL → S3 Storage → Athena/QuickSight Analytics
```

Komplett, production-ready data engineering megoldás Chicago közlekedési analytics-hez.