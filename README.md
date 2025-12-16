# Deep Learning Homework (vitmav45)

## Csapat
- **Csapatnév:** MarketPredict 
- **Név:** Sinkó Viktor Péter
- **Neptun-kód:** XCT9YC
- **Cél:** aláírás szerzése a tárgyból

---

## Projekt rövid leírása

**Cél:** egy meglévő day-trading stratégiám (NinjaTrader / NinjaScript) belépéseinek **automatikus szűrése** egy deep learning osztályozóval. A modell a belépés pillanatában elérhető piaci kontextusból **megbecsüli, hogy a trade várhatóan nyereséges vagy veszteséges lenne**, és csak a „jó” setupokat engedi át. Ezzel szeretném **növelni a winrate-et, csökkenteni a drawdown-t**, és összességében **növelni a profitot** a vizsgált időszakon.

**Instrumentum & időszak:**  
- Eszköz: **MNQ futures (Micro E-mini Nasdaq-100)**  
- Időszak: **2020.01.01 – 2025.11.13.**  
- Időkeret: a stratégia **5 perces** charton fut, több időkeretes kontextus (15m, 60m) és indikátorok kerülnek be a feature-ök közé.  

**Adatforrás:**  
- **NinjaTrader** (NinjaScript). A stratégiába beépített logger két CSV-be ír: külön **LONG** és külön **SHORT** belépésekhez.  
- A NinjaScript forrást **nem** töltöm fel a GitHub-ra; a repo a rövidített adatfájlokat, az adatelőkészítő és a tanító/tesztelő Python kódot tartalmazza .ipynb formátumban.

---

## Futtatás

### 1) Függőségek telepítése
Minimálisan szükséges csomagok:
- `pip install numpy pandas scikit-learn torch matplotlib`

> Megjegyzés: GPU nem kötelező, CPU-n is fut.

---

### 2) Teljes pipeline futtatása (adat-előkészítés + tanítás + kiértékelés)

1. **(Opcionális, teljes adathalmaz esetén)** tedd a teljes nyers CSV-ket a `data/` mappába:
   - `STDL_LongLog.csv`
   - `STDL_ShortLog.csv`

   A repóban alapból csak rövidített minta fájlok vannak, így a teljes futtatáshoz kérd el a teljes adathalmazt.

2. Futtasd a notebookot:
   - `data_prep.ipynb`
   
   Ez létrehozza a `prepared/` mappában a train/valid/test splitet és a scaler metaadatokat.

3. Futtasd a notebookot:
   - `model_trainer.ipynb`
   
   Ez betanítja a modellt, kiértékeli (valid/test), és elmenti a modellt a `models/` mappába.

---

### 3) Csak a teszt futtatása (előre betanított modellel, TRAIN/VALID nélkül)

Ha csak a kész modellen szeretnél **TEST kiértékelést** futtatni, használd a külön notebookot:
- `test_model.ipynb`

**Ehhez nem kell** a nyers CSV és nem kell a train/valid NPZ sem, csak az alábbi fájlok:

**Szükséges fájlok:**
- `prepared/long_test.npz`
- `prepared/short_test.npz`
- `prepared/scaler_unified.json`
- `models/unified_long_short_classifier.pt`
- `test_model.ipynb`

**Futtatás:**
- Nyisd meg és futtasd: `test_model.ipynb`

**Mit számol ki a teszt notebook?**
- Klasszifikációs metrikák a TEST halmazon:
  - confusion matrix
  - classification report
  - ROC-AUC + ROC görbe
  - predikciós valószínűség hisztogramok
- PnL alapú szűrési szimuláció TEST-en:
  - baseline (minden trade)
  - `p(win) >= 0.5` küszöb
  - (opcionális) küszöb-sweep és „best threshold” keresés **TEST-en** (LONG/SHORT külön), majd ennek PnL összevetése

> Fontos: a küszöb-sweep, ha a TEST-en történik, optimista becslést ad (elemzésre jó), „valódi” modellezésben ezt VALID-en érdemes hangolni.

---

## Repo tartalma

- `README.md` – ez a fájl
- `data_prep.ipynb` – adatelőkészítő notebook
- `model_trainer.ipynb` – tanító + (valid/test) kiértékelő notebook
- `test_model.ipynb` – **különálló, csak-TEST** kiértékelő notebook (előre mentett modell betöltésével)
- `data/` – két nyers (rövidített) CSV minta:
  - `STDL_LongLog_20_row_sample.csv`
  - `STDL_ShortLog_20_row_sample.csv`
- `prepared/` – a `data_prep.ipynb` által előállított tanításhoz kész be/kimenetek:
  - `long_train.npz`, `long_valid.npz`, `long_test.npz`
  - `short_train.npz`, `short_valid.npz`, `short_test.npz`
  - `scaler_unified.json` (snapshot normalizálók + meta)
- `models/` – mentett modellek:
  - `unified_long_short_classifier.pt`

---

## Adatforrás és adatgyűjtés

- **Platform:** NinjaTrader 8  
- **Logger:** a stratégia belépési triggert kiváltó bar zárásakor rögzíti:
  - **pillanatfelvételi** (snapshot) mezők,
  - **idősoros** jellemzők (utolsó 160 bar × 18 csatorna),
  - **célérték/label** a trade tényleges kimeneteléből (mi zárta, PnL).  
- **Kimenet:** két külön CSV:
  - `STDL_LongLog.csv` — csak long belépések
  - `STDL_ShortLog.csv` — csak short belépések

> A LONG és SHORT fájlok **egy közös (unified) modell** tanítására szolgálnak. A vesztes és nyertes kereskedések aránya mindkét oldalon **kb. 50–50%**.

---

## A CSV-k oszlopai (mezők magyarázata)

### Meta + snapshot (egy sor = 1 belépés)
- `sample_id`: egyedi azonosító (a belépéshez kötött).
- `timestamp`: a belépési **döntés** időpontja (ISO-8601).
- `instrument`: instrumentum neve (pl. `MNQ 12-25`).
- `session_date`: kereskedési nap (YYYY-MM-DD).
- `direction`: `LONG` vagy `SHORT` (külön fájlokban is szerepel).
- `order_offset_ticks`: a limit order ára hány tickre van a záróártól a trigger pillanatában.
- `minutes_from_open`: percek a mai session nyitása óta.
- `dow_mon` … `dow_fri`: hétfő–péntek one-hot jelölők (0/1).
- `vol_regime20`: utolsó 20 bar loghozamainak szórása (volatilitás).
- `ret_since_session_open_pct`: az ár elmozdulása a mai nyitáshoz képest (%).
- `bars_since_last_st_flip`: hány bar telt el az **utolsó SuperTrend áttörés** óta.
- `st_flips_today`: **mai napon eddig** hány ST flip történt.
- **15 perces kontextus:**
  - `htf15_dist_ema50_atr`, `htf15_dist_ema200_atr`, `htf15_rsi14_scaled`
- **60 perces kontextus:**
  - `htf60_dist_ema50_atr`, `htf60_dist_ema200_atr`, `htf60_rsi14_scaled`

### Idősoros csatornák (160 bar × 18 csatorna)
Minden csatorna 160 időlépésen szerepel, suffix-szel: `_1 … _160` (ahol `_1` a legfrissebb zárt bar).

1. `series_log_ret_*`
2. `series_tr_range_pct_*`
3. `series_vol_rel20_*`
4. `series_atr14_pct_*`
5. `series_dist_rma1_atr_*`
6. `series_dist_rma2_atr_*`
7. `series_rma1_slope_norm_*`
8. `series_rma2_slope_norm_*`
9. `series_dist_st_upper_atr_*`
10. `series_dist_st_lower_atr_*`
11. `series_st_bandwidth_atr_*`
12. `series_rsi14_scaled_*`
13. `series_macd_hist_*`
14. `series_body_to_range_*`
15. `series_dist_ema20_atr_*`
16. `series_dist_ema50_atr_*`
17. `series_ema20_slope_norm_*`
18. `series_ema50_slope_norm_*`

### Label / kimeneti mezők (a tényleges eredmény)
- `filled` (0/1): betöltődött-e az order.
- `exit_reason`: mi zárta a pozíciót (`TP`, `SL`, `StrategyClose`, `SessionClose`, `NotFilled`).
- `entry_price`, `exit_price`: belépési/kilépési átlagár.
- `quantity`: kontraktszám.
- `pnl_ticks_gross`, `pnl_usd_gross`: bruttó eredmény tickben / USD-ben.
- `pnl_ticks_net`, `pnl_usd_net`: nettó eredmény (fix $1 költség levonva).

**Célváltozó a tanításhoz:** a `data_prep.ipynb` a nettó PnL-t (`pnl_usd_net`) adja át `y`-ként.  
A klasszifikációs címkét a tanító kód képezi: `y > 0` → „nyerő”, különben „vesztő”.

---

## Adataim rövid feltárása

- **Időszak:** 2020-01-01 – 2025-11-13 (MNQ).  
- **Sorok száma (teljes nyers CSV-kből):**
  - LONG: **2,308** sor
  - SHORT: **2,317** sor
- **Idősoros tensor méret:** mindkét oldalnál **(N, 160, 18)**  
- **Snapshot vektor méret:** mindkét oldalnál **(N, 17)**  
- A long/short állományokban a nyerő/vesztő arány **kb. fele–fele**.

---

## `data_prep.ipynb` – mit csinál és mit eredményez?

Feladata: a két CSV betöltése, tisztítása, a feature-k szétszedése **idősoros** (X_series) és **snapshot** (X_snapshot) komponensekre, majd a cél (`y = pnl_usd_net`) kinyerése. Ezután standardizálja a snapshot jellemzőket (idősoros csatornákat nem), és három részre oszt: **train / valid / test**. Végül mindent **NPZ** fájlokba ment a `prepared/` mappába, és eltárolja a normalizálás metaadatait (`scaler_unified.json`).

**Eredményfájlok (`prepared/`):**
- `long_train.npz`, `long_valid.npz`, `long_test.npz`
- `short_train.npz`, `short_valid.npz`, `short_test.npz`
- `scaler_unified.json`

Az `.npz` fájlok tartalma:
- `X_series`: `(N, 160, 18)`
- `X_snapshot`: `(N, 17)` (normalizálva)
- `y`: `(N,)` nettó PnL (USD)
- meta mezők: `sample_id`, `timestamp`, `direction`, `exit_reason`

---

## Modellezés (jelenlegi)

**Megközelítés:** egy közös (unified) bináris osztályozó, amely LONG+SHORT mintákat együtt tanul.
- Idősor ág: **TCN (Temporal Convolutional Network)**
- Snapshot ág: **MLP**
- Összefűzés után közös fej (sigmoid)

**Directional információ:** a snapshot vektorhoz futás közben hozzáadódik egy `is_short` flag (0=LONG, 1=SHORT), hogy a modell tanulhassa az oldal-specifikus eltéréseket.

**Célfüggvény:** bináris keresztentrópia a `sign(y)` alapján.

---

## Következő lépések / továbbfejlesztési irányok

1. **Keresztvalidáció időben robusztus módon**
   - Klasszikus random K-fold helyett **walk-forward / rolling window** validáció javasolt (idősoros adatnál nem szabad “időben keverni”).
   - Cél: stabilabb általánosíthatóság, kisebb “szerencse” a split kiválasztásában.

2. **Új adatok létrehozása / adatnövelés (data augmentation)**
   - Snapshot zajosítása (kis Gaussian noise) csak a nem-bináris, folytonos mezőkre.
   - Idősorokon kis amplitúdójú perturbációk, vagy kisebb időablak-variálás (pl. 140–160 bar random crop).
   - Label-robosztusság: “near-zero PnL” zóna külön kezelése (pl. 3-osztály: loss / flat / win), vagy marginos binarizálás (pl. y>+x).

3. **Hiperparaméter-optimalizáció**
   - Grid/Random search vagy Bayes-opt (pl. Optuna) a fő paraméterekre:
     - TCN rétegszám, hidden méret, kernel size, dropout
     - MLP hidden méret, dropout
     - tanulási ráta, batch size, weight decay
     - class weighting / focal loss kipróbálása
   - Cél: jobb ROC-AUC / PnL szűrési teljesítmény validon (nem teszten).

4. **Küszöb-hangolás helyesen (VALID-on)**
   - A PnL maximalizáló küszöböt **VALID-on** érdemes megtalálni (külön LONG/SHORT), és csak utána egyszer kiértékelni a TEST-en.
   - A jelenlegi `test_model.ipynb` sweep-je kifejezetten elemző jellegű (optimista, mert testre hangol).

5. **Out-of-sample robusztusság**
   - Szétválasztott időszakok (pl. 2020–2023 train, 2024 valid, 2025 test) kipróbálása.
   - Cél: regime-váltások elleni stabilitás.

---
