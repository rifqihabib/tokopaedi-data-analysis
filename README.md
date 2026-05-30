# tokopaedi-data-analysis
Final Project Data Analytics Bootcamp MySkill
# Tokopaedi Data Analysis - SQL Final Project

Repository ini berisi portofolio proyek akhir (Final Project) analisis data menggunakan SQL, sebagai bagian dari program Data Analyst di MySkill. Analisis ini dilakukan pada dataset e-commerce fiktif bernama **Tokopaedi**.

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
Proyek ini menggunakan 6 tabel utama yang saling berelasi untuk melacak data penjualan, produk, pelanggan, pembayaran, dan performa corong pemasaran (*funnel*):
1. `order_detail`: Mencatat data pesanan pelanggan, kuantitas, harga, diskon, dan status validitas transaksi.
2. `product_detail`: Informasi produk seperti SKU, kategori, brand, nama produk, varian, dan harga modal (COGS).
3. `customer_detail`: Informasi profil pelanggan, tanggal registrasi, channel pendaftaran, dan lokasi provinsi.
4. `payment_detail`: Daftar metode pembayaran yang tersedia.
5. `funnel_detail`: Data rekam jejak aktivitas pengunjung (event) dan sumber trafik.
6. `transaction_detail`: Rincian finansial transaksi, nilai pajak, biaya pengiriman, dan total bersih yang dibayarkan.

---

## 🛠️ Pembahasan Studi Kasus SQL

### Kasus 1: Laporan Total Sales per Bulan (Tahun 2024)
**Pertanyaan:** Buatkan Laporan total sales per bulan untuk tahun 2024, berdasarkan tabel `transaction_detail`.

* **SQL Query:**
```sql
SELECT
    FORMAT_DATE('%B', transaction_date) AS month_name,
    SUM(total_paid) AS total_sales
  FROM
    finpro_sql_myskill.transaction_detail
  WHERE EXTRACT(year FROM transaction_date) = 2024
  GROUP BY month_name
  ORDER BY MIN(transaction_date) ASC;

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

