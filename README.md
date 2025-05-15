
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
