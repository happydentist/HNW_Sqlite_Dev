# SQLITE 經典的「動態樞紐分析(Dynamic Pivot)」或「元編程(Metaprogramming)」技巧

## 1. 核心 SQL 轉置原理
在 SQLite 中，我們必須使用條件式聚合（Conditional Aggregation）來取代 PIVOT：
```sql
-- 靜態結構示意
SELECT 
    Product,
    SUM(CASE WHEN Year = 2024 THEN Amount ELSE 0 END) AS,
    SUM(CASE WHEN Year = 2025 THEN Amount ELSE 0 END) AS [2025]
FROM Sales
GROUP BY Product;
```
## 2.SQLite 純 SQL 的極限替代方案
如果不想用外部程式，且在某些特定終端機介面下，SQLite 唯一的純 SQL 歪招是利用 printf 和 group_concat 拼出 SQL 字串，但你仍得手動複製該字串去執行：
```sql
-- 這段 SQL 會「輸出」另一段 SQL 程式碼
SELECT printf(
    'SELECT Product, %s FROM Sales GROUP BY Product;',
    GROUP_CONCAT(printf('SUM(CASE WHEN Year = %d THEN Amount ELSE 0 END) AS "%d"', Year, Year))
)
FROM (SELECT DISTINCT Year FROM Sales ORDER BY Year);
```

注意：此方法受限於 group_concat 的字串長度限制（預設 1,000,000 字元），且無法直接在同一個查詢中動態執行。

## 3.搭配 Sqlean 的 eval()

## 1. 核心的「彎道超車」技巧
因為 eval() 執行動態語法時返回的是字串，直接拿來作 SELECT 查詢在處理多個回傳列與欄位時不太直覺。因此，最優雅、最經典的解法是：利用 eval() 動態建立一個「檢視表（View）」。每一次查詢時，我們先叫 eval() 去跑一段自動偵測欄位、組裝、並覆蓋舊 View 的 DDL 語句，接著直接去讀取該 View 即可。

eval() 函數在本質上是一個純量函數（Scalar Function），它預設只會將動態執行的結果轉成「單一字串」或單一欄位回傳。如果直接用它來跑複雜的 SELECT * 多欄位、多列報表，結果會全部縮在一起，完全失去樞紐分析表（Matrix/Grid）應有的欄位結構。因此，「動態覆蓋 View（檢視表）」，正是將元編程（Metaprogramming）與動態 SQL 發揮到極致的大師級經典解法。

## 2. 實作範例
核心策略：兩步流（Two-Step Pipeline）
* 第一步（元編程重新整理）：利用 eval() 執行 DROP VIEW 與 CREATE VIEW，每次執行時，它會自動抓取當前最新的資料列，動態生成定義好的 View。
* 第二步（直覺查詢）：直接對該 View 進行 SELECT * 查詢，這時它已經是擁有正確欄位結構的標準表格了。100% 純 SQLite 實作語法
```sql
-- 【準備工作】載入 sqlean 擴充功能
.load ./eval

-- =================================================================
-- 步驟一：元編程核心 —— 動態建立/覆蓋檢視表 (Dynamic View Recreation)
-- =================================================================
SELECT eval(
    (
        SELECT printf(
            'DROP VIEW IF EXISTS v_dynamic_pivot; ' ||
            'CREATE VIEW v_dynamic_pivot AS ' ||
            'SELECT Product, %s FROM Sales GROUP BY Product;',
            group_concat(
                printf('SUM(CASE WHEN Year = ''%s'' THEN Amount ELSE 0 END) AS "%s"', Year, Year)
            )
        )
        FROM (SELECT DISTINCT Year FROM Sales ORDER BY Year)
    )
);

-- =================================================================
-- 步驟二：直覺式查詢 —— 這時候欄位已經完美展開，且具備正確的資料型態
-- =================================================================
SELECT * FROM v_dynamic_pivot;
```

## 為什麼這個解法被奉為經典？（優勢剖析）
### * 解耦多列與多欄的限制：
  eval() 在第一步只負責執行 DDL 語句（CREATE VIEW）。對 DDL 而言，它不需要回傳任何資料列，完美避開了 eval() 無法直覺回傳二維表格的硬傷。
### * 完美的客戶端相容性：
  一旦 View 被動態建立完成，不論你後端是用 Python、Node.js、C#，還是任何簡單的報表 UI 工具，都只需要下一行最直覺的 SELECT * FROM v_dynamic_pivot 即可，完全不需要在後端程式碼裡處理噁心的字串拼接。
### * 自動適應資料變更：
  每當資料庫新增了新的年份（例如多出了 2026, 2027），後端只需要在查詢前重新呼叫一次「步驟一」的 SQL，View 就會自動與時俱進，完成自動化元編程的閉環。「透過 eval() 驅動 DDL 來重建 View」的思維，確實是 SQLite 圈子裡處理動態樞紐分析最漂亮、含金量最高的操作。

---

# 動態產生「排除特定欄位」的 SELECT 語句

## SQLite 預設不支援 SELECT * EXCLUDE (欄位)。
如果一張表有 50 個欄位，您只想排除掉 password 和 secret_key 兩個敏感欄位，可以這樣寫：
```sql
SELECT printf(
    'SELECT %s FROM users;', 
    group_concat(name, ', ')
) AS dynamic_select_sql
FROM pragma_table_info('users')
WHERE name NOT IN ('password', 'secret_key'); -- 這裡輸入要排除的欄位
```
* 動態生成結果：
SELECT id, username, email, created_at FROM users;
