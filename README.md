# Market-Competitiveness-and-Pricing-Analysis-of-Indian-Pharmaceutical-Products
This project analyzes the pricing and market competitiveness of Indian pharmaceutical products using SQL. It explores trends in drug formulations, manufacturers, and therapeutic classes to identify pricing gaps, assess affordability, and provide insights for healthcare decision-making and pharma market strategy.

/* ================================================================
Project: Market Competitiveness and Pricing Analysis of Indian Pharmaceutical Products
Author: Debosmita Sikdar
Goal: Analyze pricing, formulations, and manufacturer competition 
using SQL for actionable pharma market insights.
Dataset: https://www.kaggle.com/datasets/rishgeeky/indian-pharmaceutical-products
=================================================================== */


/* ---------------------- A. Database & Table ---------------------- */
CREATE DATABASE IF NOT EXISTS pharma_analysis;
USE pharma_analysis;

DROP TABLE IF EXISTS pharma_products;
CREATE TABLE pharma_products ( 
product_id INT, 
brand_name TEXT, 
manufacturer TEXT, 
price_inr DECIMAL(10,2), 
is_discontinued BOOLEAN, 
dosage_form TEXT, 
pack_size INT, 
pack_unit TEXT, 
num_active_ingredients INT, 
primary_ingredient TEXT, 
primary_strength TEXT, 
active_ingredients TEXT, 
therapeutic_class TEXT, 
packaging_raw TEXT, 
manufacturer_raw TEXT);

/* ---------------------- B. Import data (CSV) ---------------------- */

/* Use MySQL Workbench's Table Data Import Wizard */

/* ---------------------- C. Data cleaning-Handle NULLs & invalid prices ---------------------- */

/* 1) Quick counts to see whether import worked */
SELECT COUNT(*) AS total_rows, 
       COUNT(price_inr) AS non_null_prices,
       SUM(CASE WHEN price_inr IS NULL OR price_inr = 0 THEN 1 ELSE 0 END) AS invalid_price_count
FROM pharma_products;

/* 2) Remove rows with no price (or decide to keep with price=0 based on business logic) */
SET SQL_SAFE_UPDATES=0;
DELETE FROM pharma_products
WHERE price_inr IS NULL OR price_inr <= 0;

/* 3) Standardize text columns: trim whitespace and normalize case for grouping */
UPDATE pharma_products
SET brand_name = TRIM(brand_name),
    manufacturer = TRIM(manufacturer),
    manufacturer_raw = TRIM(manufacturer_raw),
    dosage_form = LOWER(TRIM(dosage_form)),
    therapeutic_class = TRIM(therapeutic_class),
    packaging_raw = TRIM(packaging_raw),
    primary_ingredient = TRIM(primary_ingredient),
    primary_strength = TRIM(primary_strength);
    
/* ---------------------- D. Add two additional columns---------------------- */
ALTER TABLE pharma_products
ADD COLUMN price_per_unit DECIMAL(12,4),
ADD COLUMN primary_strength_mg DECIMAL(12,4);

/* ---------------------- E. Compute price per unit ---------------------- */
/* Use pack_size when it's numeric (pack_size already exists in file) */
/* If pack_unit is 'strip' and pack_size is number of tablets: price per tablet = price_inr / pack_size */
UPDATE pharma_products
SET price_per_unit = ROUND(price_inr / NULLIF(pack_size,0), 4)
WHERE pack_size IS NOT NULL AND pack_size > 0;

/* For any remaining NULL price_per_unit set to price_inr (fallback) */
UPDATE pharma_products
SET price_per_unit = price_inr
WHERE price_per_unit IS NULL;

/* ---------------------- F. Convert primary_strength to numeric mg (best-effort) */
/* This attempts to parse values like '500mg', '0.5 g', '500 mcg', '5 ml' into mg or NULL */
UPDATE pharma_products
SET primary_strength_mg =
  CASE
    WHEN LOWER(primary_strength) RLIKE '^[0-9]+\\s*mg' THEN CAST(REGEXP_REPLACE(primary_strength, '[^0-9]', '') AS DECIMAL)
    WHEN LOWER(primary_strength) RLIKE '^[0-9]+\\.?[0-9]*\\s*g' THEN CAST(REGEXP_REPLACE(primary_strength, '[^0-9\\.]', '') AS DECIMAL) * 1000
    WHEN LOWER(primary_strength) RLIKE '^[0-9]+\\s*mcg' THEN CAST(REGEXP_REPLACE(primary_strength, '[^0-9]', '') AS DECIMAL) / 1000
    WHEN LOWER(primary_strength) RLIKE '^[0-9]+\\s*ug' THEN CAST(REGEXP_REPLACE(primary_strength, '[^0-9]', '') AS DECIMAL) / 1000
    WHEN LOWER(primary_strength) RLIKE '^[0-9]+\\s*ml' THEN NULL  /* ml not convertible to mg generically */
    ELSE NULL
  END
WHERE primary_strength IS NOT NULL AND primary_strength <> '';

/* Check conversion results */
SELECT COUNT(*) AS rows_with_strength, COUNT(primary_strength_mg) AS numeric_strength_count
FROM pharma_products;

/* ---------------------- G. Create normalised view for analysis ---------------------- */
DROP VIEW IF EXISTS v_pharma_products_normalized;
CREATE VIEW v_pharma_products_normalized AS
SELECT
  product_id,
  brand_name,
  manufacturer,
  price_inr,
  is_discontinued,
  dosage_form,
  pack_size,
  pack_unit,
  num_active_ingredients,
  primary_ingredient,
  primary_strength,
  primary_strength_mg,
  active_ingredients,
  therapeutic_class,
  packaging_raw,
  manufacturer_raw,
  price_per_unit
FROM pharma_products;

/* Quick peek */
SELECT * FROM v_pharma_products_normalized LIMIT 10;

/* ---------------------- H. Exploratory & Competitive Queries ---------------------- */

/* 1) Top dosage forms by product count */
SELECT dosage_form, COUNT(*) AS product_count
FROM v_pharma_products_normalized
GROUP BY dosage_form
ORDER BY product_count DESC
LIMIT 20;

/* 2) Price stats by dosage form */
SELECT dosage_form,
       ROUND(AVG(price_inr),2) AS avg_price,
       ROUND(MIN(price_inr),2) AS min_price,
       ROUND(MAX(price_inr),2) AS max_price,
       ROUND(STDDEV_POP(price_inr),2) AS sd_price
FROM v_pharma_products_normalized
GROUP BY dosage_form
ORDER BY avg_price DESC;

/* 3) Top manufacturers by product count */
SELECT manufacturer, COUNT(*) AS num_products, ROUND(AVG(price_inr),2) AS avg_price
FROM v_pharma_products_normalized
GROUP BY manufacturer
ORDER BY num_products DESC
LIMIT 20;

/* 4) Top 20 most expensive products (by unit price) */
SELECT product_id, brand_name, manufacturer, price_inr, price_per_unit, dosage_form
FROM v_pharma_products_normalized
ORDER BY price_per_unit DESC
LIMIT 20;

/* 5) Combination vs single ingredient pricing (use num_active_ingredients) */
SELECT CASE WHEN num_active_ingredients > 1 THEN 'combination' ELSE 'single' END AS comp_type,
       COUNT(*) AS n,
       ROUND(AVG(price_inr),2) AS avg_price
FROM v_pharma_products_normalized
GROUP BY comp_type;

/* ---------------------- I. Outlier detection (form-wise z-score) ---------------------- */
/* Compute form-wise mean & sd, then show top outliers */
WITH form_stats AS (
  SELECT dosage_form, AVG(price_inr) AS mean_p, STDDEV_POP(price_inr) AS sd_p
  FROM v_pharma_products_normalized
  GROUP BY dosage_form
)
SELECT v.product_id, v.brand_name, v.manufacturer, v.dosage_form, v.price_inr,
       ROUND((v.price_inr - f.mean_p) / NULLIF(f.sd_p,0),2) AS z_score
FROM v_pharma_products_normalized v
JOIN form_stats f ON v.dosage_form = f.dosage_form
ORDER BY z_score DESC
LIMIT 30;

/* ---------------------- J. Small therapeutic mapping to get class-level stats ---------------------- */
/* If you want, expand this table with more keywords */
DROP TABLE IF EXISTS drug_class_keywords;
CREATE TABLE drug_class_keywords (
  id INT AUTO_INCREMENT PRIMARY KEY,
  keyword VARCHAR(255),
  therapeutic_class VARCHAR(255)
);

INSERT INTO drug_class_keywords (keyword, therapeutic_class) VALUES
('amox', 'antibiotic'),
('azith', 'antibiotic'),
('paracetamol', 'analgesic'),
('ibuprofen', 'analgesic'),
('metformin', 'antidiabetic'),
('insulin', 'antidiabetic'),
('atorvastatin', 'cardiovascular'),
('amlodipine', 'cardiovascular'),
('omeprazole', 'gastro'),
('pantoprazole', 'gastro');

/* Join using LIKE to approximate category assignment */
SELECT k.therapeutic_class,
       COUNT(*) AS num_products,
       ROUND(AVG(p.price_inr),2) AS avg_price,
       ROUND(STDDEV_POP(p.price_inr),2) AS price_sd
FROM v_pharma_products_normalized p
JOIN drug_class_keywords k ON LOWER(p.active_ingredients) LIKE CONCAT('%', k.keyword, '%')
GROUP BY k.therapeutic_class
ORDER BY num_products DESC;

/* ---------------------- K. Competitive band for a sample composition (e.g., amoxicillin) ---------------------- */
SELECT manufacturer,
       ROUND(MIN(price_inr),2) AS min_price,
       ROUND(AVG(price_inr),2) AS avg_price,
       ROUND(MAX(price_inr),2) AS max_price,
       COUNT(*) AS product_count
FROM v_pharma_products_normalized
WHERE LOWER(active_ingredients) LIKE '%amox%'
GROUP BY manufacturer
ORDER BY avg_price;

/* ---------------------- L. Summary view for portfolio deliverable ---------------------- */
DROP VIEW IF EXISTS v_market_summary;
CREATE VIEW v_market_summary AS
SELECT
  COALESCE(k.therapeutic_class, 'unknown') AS therapeutic_class,
  dosage_form,
  COUNT(*) AS total_products,
  ROUND(AVG(price_inr),2) AS avg_price,
  ROUND(MIN(price_inr),2) AS min_price,
  ROUND(MAX(price_inr),2) AS max_price,
  ROUND(STDDEV_POP(price_inr),2) AS price_stddev
FROM v_pharma_products_normalized p
LEFT JOIN drug_class_keywords k ON LOWER(p.active_ingredients) LIKE CONCAT('%', k.keyword, '%')
GROUP BY k.therapeutic_class, dosage_form;

/* Inspect the summary */
SELECT * FROM v_market_summary ORDER BY avg_price DESC LIMIT 50;

/* End of script */

