# 🛍️ Tokopaedi Data Analysis — SQL Final Project

Repository ini berisi portofolio proyek akhir (*Final Project*) analisis data menggunakan SQL, sebagai bagian dari program Data Analyst di MySkill. Analisis dilakukan pada dataset e-commerce fiktif bernama **Tokopaedi**.

---

## 👥 Anggota Kelompok 4B
* Muhammad Fathi Rizqi
* Ajeng Listya Devani
* Brigita Adven Novani
* Farid Ilham Nugrahatta
* Godefriedo Dimas Putra Brata
* Rifqi Habib
* Anugerah Putra Pratama
* Dini Desita
* Imam Afdol Hakiki Bakri
* Naura Fatin
* Yohanes Dimas Pratama
* Muhammad Hibban
* Siti Aisyah Bintang Larasati

---

## 📊 Ringkasan Dataset & Schema
Proyek ini menggunakan 6 tabel utama yang saling berelasi untuk melacak data penjualan, produk, pelanggan, pembayaran, dan performa corong pemasaran:

1. **`order_detail`** : Mencatat data pesanan pelanggan, kuantitas, harga, diskon, dan status validitas transaksi.
2. **`product_detail`** : Informasi produk seperti SKU, kategori, brand, nama produk, varian, dan harga modal (COGS).
3. **`customer_detail`** : Informasi profil pelanggan, tanggal registrasi, channel pendaftaran, dan lokasi provinsi.
4. **`payment_detail`** : Daftar metode pembayaran yang tersedia.
5. **`funnel_detail`** : Data rekam jejak aktivitas pengunjung (event) dan sumber trafik.
6. **`transaction_detail`** : Rincian finansial transaksi, nilai pajak, biaya pengiriman, dan total bersih yang dibayarkan.

---

## 🛠️ Pembahasan Studi Kasus SQL

```SQL
### 📌 Kasus 1: Laporan Total Sales per Bulan (Tahun 2024)
* **Pertanyaan:** Buatkan Laporan total sales per bulan untuk tahun 2024, berdasarkan tabel `transaction_detail`.
* **Syntax SQL:**
SELECT
  FORMAT_DATE('%B', transaction_date) AS month_name,
  SUM(total_paid) AS total_sales
FROM
  finpro_sql_myskill.transaction_detail
WHERE EXTRACT(year FROM transaction_date) = 2024
GROUP BY month_name
ORDER BY MIN(transaction_date) ASC;

---

### 📌 Kasus 2: Volume Penjualan per Kategori (2020 - 2024)
* **Pertanyaan:** Tampilkan volume (quantity) terjual per kategori setiap tahun dari 2020 s.d. 2024.
* **Syntax SQL:**
SELECT
  EXTRACT(YEAR FROM od.order_date) AS year,
  pd.category,
  SUM(od.quantity) AS total_quantity_sold
FROM finpro_sql_myskill.order_detail od
JOIN finpro_sql_myskill.product_detail pd
  ON od.sku_id = pd.sku_id
WHERE EXTRACT(YEAR FROM od.order_date) BETWEEN 2020 AND 2024
  AND od.is_valid = 1
GROUP BY year, pd.category
ORDER BY year ASC, total_quantity_sold DESC;

---

### 📌 Kasus 3: Performa Channel Penjualan (2024 vs 2023)
* **Pertanyaan:** Analisis performa channel (Web, App, Offline) di 2024 berdasarkan total orders dan revenue per bulan vs 2023.
* **Keterangan:** Kasus ini dianalisis secara komprehensif menggunakan perbandingan metrik performa channel penjualan langsung pada tabel multi-join untuk melihat pergeseran profitabilitas antara tahun 2023 dan 2024.
* **Syntax SQL:**
WITH monthly_metrics AS (
  SELECT
    EXTRACT(MONTH FROM order_date) AS month_num,
    FORMAT_DATE('%B', order_date) AS month_name,
    EXTRACT(YEAR FROM order_date) AS year,
    CASE 
      WHEN channel_type IN ('app store', 'play store') THEN 'App'
      WHEN channel_type = 'web' THEN 'Web'
      WHEN channel_type = 'offline' THEN 'Offline'
      ELSE 'Other'
    END AS channel,
    COUNT(DISTINCT order_id) AS total_orders,
    SUM(after_discount) AS revenue
  FROM finpro_sql_myskill.order_detail
  WHERE EXTRACT(YEAR FROM order_date) IN (2023, 2024)
    AND is_valid = 1
  GROUP BY month_num, month_name, year, channel
),
data_2023 AS (
  SELECT month_num, month_name, channel, total_orders, revenue
  FROM monthly_metrics WHERE year = 2023
),
data_2024 AS (
  SELECT month_num, month_name, channel, total_orders, revenue
  FROM monthly_metrics WHERE year = 2024
)
SELECT
  curr.month_name,
  curr.channel,
  curr.total_orders AS total_orders_2024,
  curr.revenue AS revenue_2024,
  prev.revenue AS revenue_2023,
  ROUND(SAFE_DIVIDE((curr.revenue - prev.revenue), prev.revenue) * 100, 2) AS yoy_growth_pct
FROM data_2024 curr
LEFT JOIN data_2023 prev
  ON curr.month_num = prev.month_num AND curr.channel = prev.channel
ORDER BY curr.month_num ASC, curr.channel ASC;
---

### 📌 Kasus 4: Kinerja Corong Pemasaran (Organic Funnel Event 2024)
* **Pertanyaan:** Laporan Kinerja Funnel Event "Organic" Periode 1 Januari - 31 Desember 2024 berdasarkan total jumlah event organic, total unique order_id, dan conversion rate.
* **Syntax SQL:**
SELECT
  channel_source,
  COUNT(*) AS total_events,
  COUNT(DISTINCT order_id) AS total_orders,
  ROUND(COUNT(DISTINCT order_id) / COUNT(*) * 100, 2) AS conversion_rate_pct
FROM finpro_sql_myskill.funnel_detail
WHERE event = 'Organic'
  AND funnel_date BETWEEN '2024-01-01' AND '2025-01-01'
GROUP BY channel_source
ORDER BY channel_source;
---

### 📌 Kasus 5: Analisis Kecepatan Onboarding Pelanggan Baru (2024)
* **Pertanyaan:** Laporan Registrasi & Rata-rata Waktu ke Pembelian Pertama Pelanggan Baru di Tahun 2024.
* **Syntax SQL:**
WITH first_order AS (
  SELECT customer_id,
    MIN(DATE(order_date)) AS first_order_date
  FROM finpro_sql_myskill.order_detail
  WHERE is_valid = 1
    AND is_net = 1
  GROUP BY customer_id
)
SELECT 
  FORMAT_DATE('%B', DATE(c.registration_date)) AS bulan,
  c.registration_channel,
  COUNT(DISTINCT c.customer_id) AS total_customer,
  ROUND(AVG(DATE_DIFF(f.first_order_date, DATE(c.registration_date), DAY)), 1) AS avg_days_to_buy
FROM finpro_sql_myskill.customer_detail c
JOIN first_order f
  ON c.customer_id = f.customer_id
WHERE EXTRACT(year FROM DATE(c.registration_date)) = 2024
GROUP BY EXTRACT(MONTH FROM DATE(c.registration_date)), bulan, c.registration_channel
ORDER BY EXTRACT(MONTH FROM DATE(c.registration_date)) ASC;
---

## 💡 Rekomendasi Strategis Bisnis

1. **Optimalisasi High Season & Forecasting:** Memaksimalkan perencanaan stok (*inventory*) dan kesiapan finansial menjelang kuartal akhir (Q4) guna menangkap lonjakan pasar secara maksimal.
2. **Taktik Cross-Selling:** Memasangkan produk populer (seperti *F&B* atau *Fashion*) dengan kategori yang bervolume rendah (*Sports Equipment* dan *Home Decor*) lewat promo kombo untuk mendongkrak penjualan silang.
3. **Pemberian Voucher Onboarding Berbatas Waktu:** Memberikan insentif berupa voucer diskon belanja pertama dengan masa kedaluwarsa ketat (misal: berlaku maksimal 7 hari setelah registrasi) guna memangkas durasi konversi pengguna baru.
4. **Audit UX/UI Web & Sistem Pengingat Keranjang:** Memperbaiki alur navigasi halaman pembayaran pada situs web dan mengaktifkan notifikasi otomatis (*push notification*) jika ada barang yang tertinggal di keranjang belanja.

---
*Catatan: Detail visualisasi grafik lengkap dan dokumen laporan resmi dapat diakses secara langsung melalui file berkas PDF kelompok kami yang terlampir di dalam repositori ini.*
