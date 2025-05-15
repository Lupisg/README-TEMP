
# 📊 Panduan Implementasi SQL: Tracking Perubahan Kualitas Nasabah

Dokumen ini menjelaskan tahapan implementasi modular untuk mendeteksi dan menganalisis perubahan kualitas (kolektibilitas) kredit nasabah.

---

## 📌 1. Persiapan Function

### ✅ Buat fungsi untuk ekstraksi ID Fasilitas Real

```sql
CREATE FUNCTION dbo.GetBeforeSecondDot (@input VARCHAR(100))
RETURNS VARCHAR(100)
AS
BEGIN
    DECLARE @result VARCHAR(100)
    
    IF @input IS NULL 
        SET @result = NULL
    ELSE IF CHARINDEX('.', @input) = 0 
        SET @result = @input
    ELSE IF CHARINDEX('.', @input, CHARINDEX('.', @input) + 1) = 0
        SET @result = @input
    ELSE
        SET @result = LEFT(
            @input,
            CHARINDEX('.', @input, CHARINDEX('.', @input) + 1) - 1
        )
    
    RETURN @result
END
```

---

## 🛠 2. Modifikasi Tabel Master

### 🔧 Tambahkan kolom computed untuk ID Fasilitas Real

```sql
ALTER TABLE Master_Kredit
ADD IdFasReal AS dbo.GetBeforeSecondDot(Id_Fasilitas) PERSISTED;
```

### 🚀 Buat indeks komposit untuk optimasi query

```sql
CREATE INDEX IX_MasterKredit_TglData_IdFasReal ON Master_Kredit(Tgl_Data, IdFasReal);
```

---

## 📊 3. Pembuatan View

### 🔍 View 1: Rekap harian per nasabah dan fasilitas

```sql
CREATE VIEW vw_RealIdFasilitas AS
SELECT 
    Tgl_data, 
    id_nasabah, 
    IdFasReal, 
    MAX(collect) AS Collect
FROM Master_Kredit
GROUP BY Tgl_data, id_nasabah, IdFasReal;
```

### 🔄 View 2: Perubahan kualitas dari waktu ke waktu

Kompatibel untuk SQL Server 2012 ke bawah (tidak menggunakan `LAG()`):

```sql
CREATE VIEW vw_PerubahanKualitasNasabah AS
WITH BaseData AS (
    SELECT 
        Tgl_data,
        ID_Nasabah,
        IdFasReal,
        Collect,
        ROW_NUMBER() OVER (
            PARTITION BY ID_Nasabah, IdFasReal 
            ORDER BY Tgl_data
        ) AS RowNum
    FROM vw_RealIdFasilitas
),
PrevData AS (
    SELECT 
        a.Tgl_data,
        a.ID_Nasabah,
        a.IdFasReal,
        a.Collect,
        b.Collect AS PrevCollect
    FROM BaseData a
    LEFT JOIN BaseData b ON 
        a.ID_Nasabah = b.ID_Nasabah AND 
        a.IdFasReal = b.IdFasReal AND
        a.RowNum = b.RowNum + 1
)
SELECT
    Tgl_data,
    ID_Nasabah,
    IdFasReal,
    Collect
FROM PrevData
WHERE PrevCollect IS NULL OR Collect <> PrevCollect;
```

---

## 📈 4. Analisis Perubahan Kualitas

### 🔎 Query: Riwayat nasabah NPL per April 2025

```sql
SELECT 
    P.*
FROM vw_PerubahanKualitasNasabah P
INNER JOIN (
    SELECT 
        ID_Nasabah, 
        IdFasReal
    FROM vw_RealIdFasilitas
    WHERE 
        Tgl_data = '2025-04-30' AND
        collect > 2
) Target ON 
    P.ID_Nasabah = Target.ID_Nasabah AND
    P.IdFasReal = Target.IdFasReal
ORDER BY 
    P.ID_Nasabah, 
    P.IdFasReal, 
    P.Tgl_data;
```

---

## ⚠️ Catatan Penting

### 🧭 Urutan Eksekusi

- Jalankan script dari atas ke bawah
- Function harus dibuat sebelum ALTER TABLE

### ⚙️ Kompatibilitas

- SQL Server 2012 ke bawah (tanpa `LAG()`)
- Gunakan `ROW_NUMBER()` untuk identifikasi perubahan

### 🔄 Pemeliharaan

- Rebuild indeks secara berkala
- Pantau ukuran tabel setelah penambahan kolom computed

### 🛠 Kustomisasi

- Sesuaikan `collect > 2` dengan definisi NPL internal
- Periode waktu dapat diganti sesuai kebutuhan analisis

### 🚀 Optimasi

- Gunakan partisi tabel bila data sangat besar
- Pertimbangkan job otomatis untuk update data historis

---

## ✅ Verifikasi & Troubleshooting

### 🔢 Hitung nasabah+fasilitas yang NPL

```sql
SELECT COUNT(DISTINCT ID_Nasabah + '|' + IdFasReal)
FROM vw_RealIdFasilitas
WHERE Tgl_data = '2025-04-30' AND collect > 2;
```

### 📊 Lihat distribusi perubahan

```sql
SELECT 
    ID_Nasabah, 
    IdFasReal, 
    COUNT(*) AS JumlahPerubahan
FROM (
    SELECT P.*
    FROM vw_PerubahanKualitasNasabah P
    INNER JOIN (
        SELECT ID_Nasabah, IdFasReal
        FROM vw_RealIdFasilitas
        WHERE Tgl_data = '2025-04-30'
        AND collect > 2
    ) Target ON P.ID_Nasabah = Target.ID_Nasabah
       AND P.IdFasReal = Target.IdFasReal
) Result
GROUP BY ID_Nasabah, IdFasReal
ORDER BY JumlahPerubahan DESC;
```

### 🔍 Cek riwayat untuk satu nasabah

```sql
SELECT * FROM vw_PerubahanKualitasNasabah
WHERE ID_Nasabah = 'NAS001' AND IdFasReal = 'FAS123';
```

---

## 🧪 Validasi Akhir

Pastikan data konsisten dengan:

```sql
SELECT DISTINCT P.ID_Nasabah, P.IdFasReal
FROM vw_PerubahanKualitasNasabah P
WHERE EXISTS (
    SELECT 1 FROM vw_RealIdFasilitas T
    WHERE T.Tgl_data = '2025-04-30'
      AND T.collect > 2
      AND P.ID_Nasabah = T.ID_Nasabah
      AND P.IdFasReal = T.IdFasReal
);
```

---

Solusi ini mempermudah:
- Identifikasi fasilitas kredit yang stabil
- Deteksi perubahan kolektibilitas
- Monitoring tren nasabah menuju NPL



---
## Analisis Nasabah dengan Riwayat NPL yang Sekarang Sehat

Untuk mengidentifikasi nasabah yang saat ini memiliki kualitas baik (collect ≤ 2) tetapi pernah memiliki riwayat NPL (collect > 2), berikut beberapa solusi SQL yang kompatibel dengan SQL Server 2012:

### Solusi 1: Menggunakan EXISTS untuk Mengecek Riwayat NPL
```sql
WITH NasabahApril2025 AS (
    SELECT DISTINCT ID_Nasabah, IdFasReal
    FROM vw_RealIdFasilitas
    WHERE Tgl_data = '2025-04-30'
      AND collect <= 2
)
SELECT 
    N.ID_Nasabah,
    N.IdFasReal,
    MAX(CASE WHEN V.Collect > 2 THEN 'Pernah NPL' ELSE 'Tidak Pernah NPL' END) AS StatusRiwayat,
    COUNT(CASE WHEN V.Collect > 2 THEN 1 END) AS JumlahKejadianNPL,
    MAX(V.Collect) AS KualitasTerburuk
FROM NasabahApril2025 N
JOIN vw_PerubahanKualitasNasabah V ON 
    N.ID_Nasabah = V.ID_Nasabah AND
    N.IdFasReal = V.IdFasReal
GROUP BY N.ID_Nasabah, N.IdFasReal
HAVING COUNT(CASE WHEN V.Collect > 2 THEN 1 END) > 0
ORDER BY JumlahKejadianNPL DESC;
```

### Solusi 2: Detail Riwayat Perubahan Kualitas
```sql
WITH NasabahSekarangSehat AS (
    SELECT DISTINCT ID_Nasabah, IdFasReal
    FROM vw_RealIdFasilitas
    WHERE Tgl_data = '2025-04-30'
      AND collect <= 2
),
NasabahPernahNPL AS (
    SELECT DISTINCT ID_Nasabah, IdFasReal
    FROM vw_PerubahanKualitasNasabah
    WHERE Collect > 2
)
SELECT 
    V.*,
    CASE WHEN V.Collect > 2 THEN 'NPL' 
         WHEN V.Collect <= 2 THEN 'Sehat'
         ELSE 'Unknown' END AS StatusKualitas
FROM vw_PerubahanKualitasNasabah V
JOIN NasabahSekarangSehat S ON 
    V.ID_Nasabah = S.ID_Nasabah AND
    V.IdFasReal = S.IdFasReal
JOIN NasabahPernahNPL N ON 
    V.ID_Nasabah = N.ID_Nasabah AND
    V.IdFasReal = N.IdFasReal
ORDER BY 
    V.ID_Nasabah, 
    V.IdFasReal, 
    V.Tgl_data;
```

### Solusi 3: Analisis Frekuensi NPL
```sql
SELECT 
    A.ID_Nasabah,
    A.IdFasReal,
    A.Collect AS KualitasTerakhir,
    MAX(B.Collect) AS KualitasTerburuk,
    COUNT(CASE WHEN B.Collect > 2 THEN 1 END) AS FrekuensiNPL,
    MIN(CASE WHEN B.Collect > 2 THEN B.Tgl_data END) AS PertamaKaliNPL,
    MAX(CASE WHEN B.Collect > 2 THEN B.Tgl_data END) AS TerakhirKaliNPL
FROM vw_RealIdFasilitas A
JOIN vw_PerubahanKualitasNasabah B ON 
    A.ID_Nasabah = B.ID_Nasabah AND
    A.IdFasReal = B.IdFasReal
WHERE 
    A.Tgl_data = '2025-04-30' AND
    A.Collect <= 2
GROUP BY 
    A.ID_Nasabah,
    A.IdFasReal,
    A.Collect
HAVING 
    COUNT(CASE WHEN B.Collect > 2 THEN 1 END) > 0
ORDER BY 
    FrekuensiNPL DESC,
    KualitasTerburuk DESC;

```


Interpretasi Hasil
```
FrekuensiNPL: Jumlah kali nasabah pernah NPL
KualitasTerburuk: Nilai collect tertinggi yang pernah dicapai
PertamaKaliNPL: Tanggal pertama kali menjadi NPL
TerakhirKaliNPL: Tanggal terakhir kali menjadi NPL
```

### Analisis Tambahan: Evaluasi Risiko Berdasarkan Proporsi NPL
```sql
WITH CustomerHistory AS (
    SELECT 
        ID_Nasabah,
        IdFasReal,
        Collect,
        COUNT(*) AS Frekuensi
    FROM vw_PerubahanKualitasNasabah
    GROUP BY ID_Nasabah, IdFasReal, Collect
),
CurrentGoodCustomers AS (
    SELECT DISTINCT ID_Nasabah, IdFasReal
    FROM vw_RealIdFasilitas
    WHERE Tgl_data = '2025-04-30'
    AND collect <= 2
),
CustomerPerformance AS (
    SELECT 
        c.ID_Nasabah,
        c.IdFasReal,
        SUM(CASE WHEN ch.Collect = 1 THEN ch.Frekuensi ELSE 0 END) AS frekuensi1,
        SUM(CASE WHEN ch.Collect = 2 THEN ch.Frekuensi ELSE 0 END) AS frekuensi2,
        SUM(CASE WHEN ch.Collect = 3 THEN ch.Frekuensi ELSE 0 END) AS frekuensi3,
        SUM(CASE WHEN ch.Collect = 4 THEN ch.Frekuensi ELSE 0 END) AS frekuensi4,
        SUM(CASE WHEN ch.Collect = 5 THEN ch.Frekuensi ELSE 0 END) AS frekuensi5,
        SUM(ch.Frekuensi) AS total_perubahan,
        MAX(CASE WHEN ch.Collect > 2 THEN 1 ELSE 0 END) AS pernah_npl
    FROM CurrentGoodCustomers c
    JOIN CustomerHistory ch ON c.ID_Nasabah = ch.ID_Nasabah AND c.IdFasReal = ch.IdFasReal
    GROUP BY c.ID_Nasabah, c.IdFasReal
    HAVING MAX(CASE WHEN ch.Collect > 2 THEN 1 ELSE 0 END) = 1
)
SELECT 
    cp.*,
    (frekuensi1 + frekuensi2) AS total_lancar,
    (frekuensi3 + frekuensi4 + frekuensi5) AS total_npl,
    CASE 
        WHEN total_perubahan > 0 
        THEN CAST((frekuensi1 + frekuensi2) * 100.0 / total_perubahan AS DECIMAL(5,2))
        ELSE 0 
    END AS persentase_lancar,
    CASE 
        WHEN total_perubahan > 0 
        THEN CAST((frekuensi3 + frekuensi4 + frekuensi5) * 100.0 / total_perubahan AS DECIMAL(5,2))
        ELSE 0 
    END AS persentase_npl,
    CASE
        WHEN persentase_lancar >= 80 THEN 'Sangat Baik'
        WHEN persentase_lancar >= 60 THEN 'Baik'
        WHEN persentase_lancar >= 40 THEN 'Cukup'
        WHEN persentase_lancar >= 20 THEN 'Risiko Tinggi'
        ELSE 'Risiko Sangat Tinggi'
    END AS analisa_hasil
FROM CustomerPerformance cp
ORDER BY persentase_lancar ASC, total_npl DESC;
```

