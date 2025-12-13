# Dokumentasi Pemrosesan Data Penelitian

> **Analisis Mobilitas Vertikal Menggunakan IFLS 2014 dan 2007**

Dataset ini merupakan hasil penggabungan dan pembersihan data dari **Indonesia Family Life Survey (IFLS)** wave 2007 dan 2014 untuk menganalisis mobilitas vertikal ekonomi dan hubungannya dengan latar belakang pendidikan (khususnya pendidikan Islam).

---

## 📁 Struktur Dataset

### Versi Dataset
Tersedia **dua versi** dataset hasil pemrosesan:

| Versi | Jumlah Observasi | Deskripsi |
|-------|------------------|-----------|
| **`processed_data.csv`** | ~10.878 | Dataset lengkap dengan semua variabel |
| **`processed_data.dta`** | ~10.878 | Dataset lengkap dengan semua variabel |

### Format File
- **CSV**: Delimiter `;` (semicolon)
- **Stata (.dta)**: Format Stata 13+
- **Encoding**: UTF-8

---

## Alur Pemrosesan Data

### 1. **Modul Identitas & Demografi (B3A_COV)**
**Sumber:** `b3a_cov.dta`

**Variabel yang diambil:**
- `pidlink`: Personal ID (unique identifier)
- `age`: Usia responden
- `sex`: Jenis kelamin

**Proses:**
```python
df_cov = pd.read_stata("b3a_cov.dta")
df_cov = df_cov[['pidlink', 'age', 'sex']]
```

---

### 2. **Modul Pendidikan (B3A_DL1, DL2, DL4)**

#### A. **Riwayat Pendidikan Formal (DL2 & DL4)**
**Sumber:** `b3a_dl2.dta`, `b3a_dl4.dta`

**Variabel kunci:**
- `dl2type`: Jenis pendidikan (SD/SMP/SMA/Universitas)
- `dl10`: Jenis sekolah
- `dl11`: Status sekolah (Negeri/Swasta/Islam)
- `dl11a`: Tahun masuk sekolah
- `dl11f`: Tahun lulus
- `number_grade_fail`: Jumlah tahun tidak naik kelas

**Proses Perhitungan Lama Pendidikan:**

| Jenjang | Formula |
|---------|---------|
| SD | `6 + number_grade_fail` |
| SMP | `3 + number_grade_fail` |
| SMA | `3 + number_grade_fail` |
| Universitas | `abs(dl11f - dl11a)` |

**Identifikasi Sekolah Islam:**
```python
islamic_school = ['2:Public religious', '4:Private islam']
df['SD_Islam'] = np.where(df['dl11'].isin(islamic_school), 1, 0)
```

**Variabel output:**
- `long_year_sd`, `long_year_smp`, `long_year_sma`, `long_year_univ`
- `sd_islam`, `smp_islam`, `sma_islam`, `univ_islam`
- `long_year_acad` = Total tahun pendidikan formal
- `long_year_acad_islam_educ` = Total tahun di institusi Islam

#### B. **Pendidikan Tertinggi & Pesantren (DL1)**
**Sumber:** `b3a_dl1.dta`

**Variabel:**
- `dl06x`: Pernah mondok di pesantren (Ya/Tidak)
- `highest_educ_pesantren`: Pesantren sebagai pendidikan tertinggi

**Agregasi Latar Belakang Islam:**
```python
df['islamic_educ_background'] = np.where(
    (df['long_year_acad_islam_educ'] > 0) | 
    (df['highest_educ_pesantren'] == 1), 1, 0
)
```

---

### 3. **Modul Pekerjaan & Pendapatan (BK_AR1, B3A_TK3)**

#### A. **Pendapatan Individu**
**Sumber:** `bk_ar1.dta` (2014 dan 2007)

**Variabel:**
- `ar15b`: Pendapatan tahunan

**Deflasi Pendapatan:**
```python
# Deflasi 2014 ke harga konstan 2007
df_k['ar15b'] = df_k['ar15b'] * 100 / 150.47  # IHK 2014 = 150.47
```

**Variabel output:**
- `current_income`: Pendapatan 2014 (ter-deflasi)
- `past_income`: Pendapatan 2007
- `increase_income`: Selisih pendapatan (`current_income - past_income`)

#### B. **Jenis & Status Pekerjaan**
**Sumber:** `b3a_tk3.dta`

**Variabel:**
- `tk31a`: Sektor pekerjaan (pertanian, manufaktur, jasa, dll.)
- `tk33`: Status pekerjaan (wiraswasta, PNS, swasta, buruh)

**Mapping status pekerjaan:**
```python
dict_tk33 = {
    '1:Self-employed': 'Wiraswasta',
    '4:Government worker': 'Pegawai Pemerintah',
    '5:Private worker': 'Pegawai Swasta',
    '7:Casual worker in agriculture': 'Buruh',
    '8:Casual worker not in agriculture': 'Buruh',
    '6:Unpaid family worker': 'Missing/Tidak Bekerja'
}
```

---

### 4. **Karakteristik Kepala Keluarga (2007)**

#### A. **Identifikasi Kepala Keluarga**
**Filter:** `ar02b == 1.0` (status kepala rumah tangga)

**Proses:**
1. Filter individu dengan status KK di 2007
2. Agregasi pendapatan per household ID (`hhid07_9`)
3. Mapping ke anggota keluarga berdasarkan `hhid`

#### B. **Pendidikan Orang Tua**
**Sumber:** File eksternal `data_pendidikan_KK.csv`

**Variabel:**
- `longYearAcad_KK`: Total pendidikan formal KK
- `longYearAcadIslamEduc_KK`: Total pendidikan Islam KK
- `islamicEducBackground_KK`: Latar belakang pendidikan Islam KK

**Catatan:** Data pendidikan KK hanya tersedia untuk subset observasi (~1.213 obs)

#### C. **Pekerjaan Orang Tua**
**Mapping dari `tk33` kepala keluarga:**
```python
def decode_pekerjaan_ortu(encode):
    if encode in [1.0, 2.0, 3.0]: return "Wiraswasta"
    elif encode == 4.0: return "Pegawai Pemerintah"
    elif encode == 5.0: return "Pegawai Swasta"
    return "Missing/Tidak Bekerja"
```

---

### 5. **Karakteristik Wilayah (BK_SC1)**
**Sumber:** `bk_sc1.dta` (2014 dan 2007)

**Variabel:**
- `sc05`: Urban/Rural
- `sc01`: Kode provinsi
- `sc02`: Kode kabupaten/kota

**Output:**
- `urban_or_rural_2014/2007`
- `province_code_2014/2007`
- `city_code_2014/2007`

---

## 📋 Deskripsi Variabel Final

### **Identitas & Demografi**
| Variabel | Tipe | Deskripsi |
|----------|------|-----------|
| `pidlink` | String | Unique identifier individu |
| `hhid` | String | Household ID (dari `hhid14_9`) |
| `age` | Kategori | Usia responden |
| `sex` | String | Jenis kelamin (Male/Female) |

### **Pendidikan Individu**
| Variabel | Tipe | Deskripsi |
|----------|------|-----------|
| `long_year_sd` | Float | Lama pendidikan SD (tahun) |
| `sd_islam` | Integer (0/1) | Sekolah Islam di jenjang SD |
| `long_year_smp` | Float | Lama pendidikan SMP (tahun) |
| `smp_islam` | Integer (0/1) | Sekolah Islam di jenjang SMP |
| `long_year_sma` | Float | Lama pendidikan SMA (tahun) |
| `sma_islam` | Integer (0/1) | Sekolah Islam di jenjang SMA |
| `long_year_univ` | Float | Lama pendidikan universitas (tahun) |
| `univ_islam` | Integer (0/1) | Universitas Islam |
| `long_year_acad` | Float | Total tahun pendidikan formal |
| `long_year_acad_islam_educ` | Float | Total tahun pendidikan Islam |
| `highest_educ_pesantren` | Integer (0/1) | Pesantren sebagai pendidikan tertinggi |
| `islamic_educ_background` | Integer (0/1) | Memiliki latar belakang pendidikan Islam |

### **Pekerjaan & Pendapatan**
| Variabel | Tipe | Deskripsi |
|----------|------|-----------|
| `type_of_work` | String | Sektor/bidang pekerjaan |
| `employment_status` | String | Status pekerjaan (Wiraswasta/PNS/Swasta/Buruh) |
| `current_income` | Float | Pendapatan 2014 (deflated ke 2007) |
| `past_income` | Float | Pendapatan 2007 |
| `increase_income` | Float | **[VARIABEL DEPENDEN]** Kenaikan pendapatan |

### **Karakteristik Kepala Keluarga (2007)**
| Variabel | Tipe | Deskripsi | Ketersediaan |
|----------|------|-----------|--------------|
| `long_year_acad_kk_2007` | Float | Total pendidikan formal KK | Parsial (~833 obs) |
| `long_year_acad_islam_educ_kk_2007` | Float | Total pendidikan Islam KK | Parsial (~833 obs) |
| `islamic_educ_background_kk_2007` | Float (0/1) | Latar belakang pendidikan Islam KK | Parsial (~833 obs) |
| `income_kk_2007` | Float | Pendapatan kepala keluarga 2007 | Lengkap |
| `employment_status_kk_2007` | String | Status pekerjaan KK 2007 | Lengkap (~10.138 obs) |

### **Karakteristik Wilayah**
| Variabel | Tipe | Deskripsi |
|----------|------|-----------|
| `urban_or_rural_2014` | Kategori | Klasifikasi wilayah 2014 |
| `province_code_2014` | Integer | Kode provinsi 2014 |
| `city_code_2014` | Integer | Kode kab/kota 2014 |
| `urban_or_rural_2007` | Integer | Klasifikasi wilayah 2007 |
| `province_code_2007` | Integer | Kode provinsi 2007 |
| `city_code_2007` | Integer | Kode kab/kota 2007 |

---

## 🔍 Handling Data

### **Missing Values**
- Data yang tidak tersedia direpresentasikan sebagai `NaN`
- Variabel pendidikan KK hanya tersedia untuk subset observasi
- Pendapatan bernilai 0 jika tidak bekerja

### **Duplicate Handling**
```python
# Contoh: Deduplikasi data pekerjaan
df_tk = df_tk.drop_duplicates(subset=['pidlink'], keep='last')
```

### **Merge Strategy**
- Primary key: `pidlink` + `hhid14_9`
- Join type: `left` untuk menjaga semua observasi utama
- `inner` untuk matching 2007-2014

---

## Cara Penggunaan

### Python (Pandas)
```python
import pandas as pd

# Membaca CSV
df = pd.read_csv("datasets/processed_data.csv", sep=";")

# Membaca Stata
df = pd.read_stata("datasets/processed_data.dta")

# Preview
print(df.info())
print(df.head())
```

### Stata
```stata
* Import dataset
use "datasets/processed_data.dta", clear

* Deskripsi variabel
describe

* Statistik deskriptif
summarize

* Contoh analisis
reg increase_income long_year_acad islamic_educ_background income_kk_2007
```

### R
```r
library(haven)
library(dplyr)

# Membaca data
df <- read_dta("datasets/processed_data.dta")

# Atau dari CSV
df <- read.csv2("datasets/processed_data.csv")  # read.csv2 untuk sep=";"

# Preview
glimpse(df)
```

---

## 📊 Potensi Analisis

### 1. **Return to Education**
```python
# Estimasi return to Islamic education
model = 'increase_income ~ long_year_acad + long_year_acad_islam_educ + C(sex) + age'
```

### 2. **Intergenerational Mobility**
```python
# Pengaruh pendidikan & pendapatan orang tua
model = '''increase_income ~ long_year_acad + islamic_educ_background + 
           income_kk_2007 + long_year_acad_kk_2007'''
```

### 3. **Gender Gap Analysis**
```python
# Interaksi gender dengan pendidikan Islam
model = 'increase_income ~ long_year_acad * C(sex) + islamic_educ_background * C(sex)'
```

### 4. **Urban-Rural Disparities**
```python
# Perbedaan mobilitas urban vs rural
df.groupby('urban_or_rural_2014')['increase_income'].describe()
```

---

## ⚠️ Catatan Penting

### **Limitasi Data**
1. **Missing data kepala keluarga**: Hanya ~833 observasi memiliki data lengkap pendidikan KK
2. **Panel attrition**: Tidak semua responden 2007 ditemukan di 2014
3. **Deflasi pendapatan**: Menggunakan IHK agregat, bukan regional

### **Rekomendasi Analisis**
- Gunakan `processed_data.csv` untuk analisis umum
- Gunakan `processed_data_2.csv` untuk analisis yang memerlukan data pendidikan orang tua lengkap
- Pertimbangkan **sample selection bias** dalam interpretasi hasil
- Lakukan **sensitivity analysis** untuk asumsi deflasi

---

## 📚 Referensi

- **IFLS Documentation**: [https://www.rand.org/well-being/social-and-behavioral-policy/data/FLS/IFLS.html](https://www.rand.org/well-being/social-and-behavioral-policy/data/FLS/IFLS.html)
- **IHK Data**: Badan Pusat Statistik Indonesia

---

## 📝 Changelog

| Versi | Tanggal | Perubahan |
|-------|---------|-----------|
| 2.0 | 2025-12-14 | Handling Duplikasi data processing |

---

**Catatan Akhir**: Dokumentasi ini dirancang untuk **reproducibility**. Semua langkah pemrosesan dapat ditelusuri melalui notebook `notebook.ipynb`.

---
Ludy Hasby Aulia