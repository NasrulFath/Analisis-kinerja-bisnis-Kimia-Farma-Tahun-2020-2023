# Cek jumlah record di masing-masing tabel sumber
SELECT COUNT(*) AS total_rows
FROM `rakamin-kf-analytics-472203.kimia_farma.kf_inventory`;

SELECT COUNT(*) AS total_rows
FROM `rakamin-kf-analytics-472203.kimia_farma.kf_kantor_cabang`;

SELECT COUNT(*) AS total_rows
FROM `rakamin-kf-analytics-472203.kimia_farma.kf_product`;

SELECT COUNT(*) AS total_rows
FROM `rakamin-kf-analytics-472203.kimia_farma.kf_final_transaction`;


# Buat tabel analisis dengan perhitungan nett_sales, nett_profit, dan gross laba %
CREATE OR REPLACE TABLE `rakamin-kf-analytics-472203.kimia_farma.kf_analysis` AS
SELECT
  t.transaction_id,
  DATE(t.date) AS date,
  t.branch_id,
  c.branch_name,
  c.kota,
  c.provinsi,
  c.rating AS rating_cabang,
  t.customer_name,
  t.product_id,
  p.product_name,
  CAST(t.price AS FLOAT64) AS actual_price,
  CAST(t.discount_percentage AS FLOAT64) AS discount_percentage,

 # Mengitung gross laba % berdasarkan price
  CASE
    WHEN t.price <= 50000 THEN 10
    WHEN t.price <= 100000 THEN 15
    WHEN t.price <= 300000 THEN 20
    WHEN t.price <= 500000 THEN 25
    ELSE 30
  END AS persentase_gross_laba,

   # Menghitung Nett sales setelah diskon
  ROUND(t.price * (1 - t.discount_percentage/100), 2) AS nett_sales,

 # Menghitung Nett profit berdasarkan nett_sales x persentase laba
  ROUND(
    ROUND(t.price * (1 - t.discount_percentage/100), 2) *
    (CASE
      WHEN t.price <= 50000 THEN 0.10
      WHEN t.price <= 100000 THEN 0.15
      WHEN t.price <= 300000 THEN 0.20
      WHEN t.price <= 500000 THEN 0.25
      ELSE 0.30
    END)
  ,2) AS nett_profit,

  t.rating AS rating_transaksi

FROM `rakamin-kf-analytics-472203.kimia_farma.kf_final_transaction` t
LEFT JOIN `rakamin-kf-analytics-472203.kimia_farma.kf_product` p
  ON t.product_id = p.product_id
LEFT JOIN `rakamin-kf-analytics-472203.kimia_farma.kf_kantor_cabang` c
  ON t.branch_id = c.branch_id;


# Cek isi tabel analisis
SELECT *
FROM `rakamin-kf-analytics-472203.kimia_farma.kf_analysis`
LIMIT 20;

# Top 10 produk dengan penjualan tertinggi
SELECT 
  product_name,
  SUM(nett_sales) AS total_sales
FROM `rakamin-kf-analytics-472203.kimia_farma.kf_analysis`
GROUP BY product_name
ORDER BY total_sales DESC
LIMIT 10;

#Top 5 cabang dengan profit tertinggi
SELECT 
  branch_name,
  kota,
  provinsi,
  SUM(nett_profit) AS total_profit
FROM `rakamin-kf-analytics-472203.kimia_farma.kf_analysis`
GROUP BY branch_name, kota, provinsi
ORDER BY total_profit DESC
LIMIT 5;

#Rata-rata rating per cabang
SELECT 
  branch_name,
  AVG(rating_transaksi) AS avg_rating
FROM `rakamin-kf-analytics-472203.kimia_farma.kf_analysis`
GROUP BY branch_name
ORDER BY avg_rating DESC;

#Top 10 cabang berdasarkan total_profit
WITH per_branch AS (
  SELECT branch_id, branch_name, kota, provinsi,
         SUM(nett_sales) AS total_sales,
         SUM(nett_profit) AS total_profit,
         COUNT(1) AS transaksi
  FROM `rakamin-kf-analytics-472203.kimia_farma.kf_analysis`
  GROUP BY branch_id, branch_name, kota, provinsi
)
SELECT * FROM per_branch
ORDER BY total_profit DESC
LIMIT 10;

# Trend bulanan (sales & profit)
WITH monthly AS (
  SELECT
    EXTRACT(YEAR FROM date) AS year,
    EXTRACT(MONTH FROM date) AS month,
    SUM(nett_sales) AS total_sales,
    SUM(nett_profit) AS total_profit
  FROM `rakamin-kf-analytics-472203.kimia_farma.kf_analysis`
  WHERE date IS NOT NULL
  GROUP BY year, month
)
SELECT
  year,
  month,
  total_sales,
  total_profit
FROM monthly
ORDER BY year, month;

# Produk best-seller per kota (top 1 product per kota)
WITH city_prod AS (
  SELECT kota, product_id, product_name, SUM(nett_sales) AS total_sales
  FROM `rakamin-kf-analytics-472203.kimia_farma.kf_analysis`
  GROUP BY kota, product_id, product_name
),
ranked AS (
  SELECT *, ROW_NUMBER() OVER (PARTITION BY kota ORDER BY total_sales DESC) AS rn
  FROM city_prod
)
SELECT kota, product_id, product_name, total_sales
FROM ranked
WHERE rn = 1
ORDER BY total_sales DESC;




