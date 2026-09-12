# Klasifikasi dan Forecasting Tingkat Kepadatan Lalu Lintas di Jalan Thamrin Menggunakan Machine Learning

Project Based Learning — Data Analyst, PPKD Jakarta Selatan

> ⚠️ Project ini sudah dipresentasikan dan mendapat masukan dari instruktur untuk beberapa perbaikan lanjutan.

## Deskripsi Project

Project ini menganalisis data lalu lintas per jam di ruas Jalan MH Thamrin, Jakarta, sepanjang tahun 2023 (8.760 baris, bersumber dari Dinas Perhubungan DKI Jakarta). Analisis mencakup tiga tahap utama:

1. **Exploratory Data Analysis (EDA)** — memahami pola kepadatan lalu lintas berdasarkan volume kendaraan, waktu (jam & hari), dan kualitas data
2. **Klasifikasi** — memprediksi tingkat kepadatan lalu lintas (*Lancar* / *Cukup Padat*) menggunakan Logistic Regression dan Random Forest
3. **Forecasting** — memprediksi nilai *Volume Capacity Ratio* (VCRatio) 1 jam ke depan menggunakan Random Forest Regressor, dideploy sebagai alat bantu internal untuk Dinas Perhubungan

## Dataset

| Kolom | Keterangan |
|---|---|
| `Date_and_Time` | Timestamp per jam |
| `In_*` / `Out_*` / `Total_*` | Volume kendaraan (Motorcycle, Car, BusTruck, Total), PCE, Velocity, VCRatio — per arah masuk, keluar, dan gabungan |

**Rentang data:** 1 Januari – 31 Desember 2023, per jam (8.760 baris)

## Struktur Analisis

### 1. Exploratory Data Analysis
- Pengecekan struktur data, missing value, dan duplikat
- **Temuan kunci:** missing value pada kolom `In_*` bersifat sistematis — mayoritas terjadi setiap hari Minggu jam 06.00–09.00, mengindikasikan downtime/maintenance sensor terjadwal. Missing value pada kolom `Out_*` bersifat lebih acak (blok pendek yang tersebar)
- Analisis univariate, bivariate, dan multivariate — termasuk korelasi antar fitur, distribusi VCRatio, dan pola temporal (heatmap jam × hari)
- **Temuan pola temporal:** rush hour ganda (pagi jam 7–9, sore jam 17) hanya konsisten muncul di hari kerja; akhir pekan menunjukkan pola yang jauh lebih landai

### 2. Data Cleaning & Preprocessing
- Imputasi missing value `In_*` menggunakan rata-rata per kombinasi hari & jam (dihitung dari data train saja, untuk menghindari data leakage)
- Interpolasi missing value `Out_*` menggunakan pendekatan time-based
- Outlier handling pada `Total_BusTruck` menggunakan IQR capping
- Split data dilakukan **sebelum** proses imputasi berbasis statistik untuk mencegah kebocoran informasi dari data test ke proses training

### 3. Klasifikasi Tingkat Kepadatan Lalu Lintas

Target diklasifikasikan menjadi 2 kelas berdasarkan `Total_VCRatio` (threshold 0.20, mengacu pada standar *Level of Service* PKJI).

Tiga skenario fitur diuji untuk memahami kontribusi masing-masing kelompok fitur:

| Skenario | Fitur | Logistic Regression | Random Forest |
|---|---|---|---|
| **Opsi A** | Seluruh fitur volume kendaraan | 99% | 99% |
| **Opsi C** | Volume BusTruck + waktu | 83% | 86% |
| **Opsi B** | Waktu saja (jam, hari, weekend) | 65% | 76% |

Evaluasi dilakukan menggunakan accuracy, precision, recall, F1-score, confusion matrix, cross-validation 5-fold, dan ROC-AUC. Random Forest secara konsisten unggul atau setara dengan Logistic Regression di seluruh skenario, dengan selisih yang makin besar ketika fitur volume kendaraan tidak tersedia — mengindikasikan Random Forest lebih baik menangkap pola non-linear pada data waktu.

### 4. Forecasting VCRatio

Model forecasting memprediksi `Total_VCRatio` 1 jam ke depan menggunakan Random Forest Regressor, dengan fitur:
- Lag VCRatio 1 jam, 24 jam, dan 168 jam sebelumnya
- Rolling mean 24 jam dan 7 hari terakhir
- Fitur waktu (jam, hari, akhir pekan)

Split data dilakukan secara **kronologis** (bukan acak) untuk mensimulasikan skenario prediksi nyata. Seluruh fitur lag dipastikan valid terhadap horizon prediksi (tidak menggunakan data yang belum tersedia pada waktu prediksi dilakukan).

| Model | MAE | RMSE | R² |
|---|---|---|---|
| Baseline naive (lag 1 jam) | 0.0265 | 0.0376 | 0.8086 |
| **Random Forest Regressor** | **0.0160** | **0.0248** | **0.9172** |

### 5. Deployment

Model forecasting dideploy sebagai aplikasi **Gradio** untuk penggunaan internal Dinas Perhubungan, dengan input kondisi VCRatio 1 jam terakhir serta informasi waktu (jam, hari, akhir pekan). Output berupa kategori tingkat kepadatan beserta nilai prediksi VCRatio.

## Tools & Library

- Python (pandas, numpy)
- scikit-learn (Logistic Regression, Random Forest, evaluasi model)
- feature-engine (encoding)
- matplotlib, seaborn (visualisasi)
- Gradio (deployment)
- joblib (penyimpanan model)

## Limitasi & Rencana Pengembangan

- Model forecasting menunjukkan indikasi overfitting ringan (R² train 0.98 vs test 0.92) — perlu eksplorasi hyperparameter tuning lebih lanjut
- Model belum menangkap anomali akibat event khusus (contoh: penurunan tajam VCRatio pada malam Tahun Baru akibat kemungkinan pengalihan arus lalu lintas)
- Fitur volume kendaraan (`In_PCE`/`Out_PCE`) pada skenario klasifikasi memiliki keterkaitan matematis erat dengan target, sehingga akurasi tinggi pada Opsi A perlu diinterpretasikan dengan hati-hati
- Forecasting masih terbatas pada horizon 1 jam; perluasan ke horizon lebih panjang (24 jam, beberapa hari) memerlukan penyesuaian fitur lag
- Penambahan fitur eksternal (cuaca, hari libur nasional, event khusus) berpotensi meningkatkan akurasi model

## Sumber Data

Dinas Perhubungan DKI Jakarta, data lalu lintas ruas Jalan MH Thamrin tahun 2023.
