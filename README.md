# SQLITE 經典的「動態樞紐分析(Dynamic Pivot)」或「元編程(Metaprogramming)」技巧

## 一. 核心 SQL 轉置原理
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
## 二.SQLite 純 SQL 的極限替代方案
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

## 三.搭配 Sqlean 的 eval()

### 1. 核心的「彎道超車」技巧
因為 eval() 執行動態語法時返回的是字串，直接拿來作 SELECT 查詢在處理多個回傳列與欄位時不太直覺。因此，最優雅、最經典的解法是：利用 eval() 動態建立一個「檢視表（View）」。每一次查詢時，我們先叫 eval() 去跑一段自動偵測欄位、組裝、並覆蓋舊 View 的 DDL 語句，接著直接去讀取該 View 即可。

eval() 函數在本質上是一個純量函數（Scalar Function），它預設只會將動態執行的結果轉成「單一字串」或單一欄位回傳。如果直接用它來跑複雜的 SELECT * 多欄位、多列報表，結果會全部縮在一起，完全失去樞紐分析表（Matrix/Grid）應有的欄位結構。因此，「動態覆蓋 View（檢視表）」，正是將元編程（Metaprogramming）與動態 SQL 發揮到極致的大師級經典解法。

sqlean-eval 允許 SQLite 執行存在字串變數中的 SQL 語句。以下是搭配 eval() 實現動態 Pivot 的方案：
#### 核心實作語法
在 SQLite 中，我們同樣使用 CASE WHEN 進行轉置，並利用 eval() 函數將動態拼接出來的 SQL 字串直接執行：
```sql
-- 載入 sqlean 擴充功能（依環境調整路徑）
.load ./eval

-- 元編程與動態執行核心
SELECT eval(
    -- 1. 使用 printf 與 group_concat 動態拼裝出完整的 SQL 語句
    (
        SELECT printf(
            'SELECT Product, %s FROM Sales GROUP BY Product;',
            group_concat(
                printf('SUM(CASE WHEN Year = ''%s'' THEN Amount ELSE 0 END) AS "%s"', Year, Year)
            )
        )
        FROM (SELECT DISTINCT Year FROM Sales ORDER BY Year)
    )
);
```
#### 語法拆解說明
* 內層查詢 (SELECT DISTINCT...)：
  撈出所有不重複的年份（例如 2024, 2025），確保欄位是動態產生的。
* printf('SUM(CASE...)...')：
  將每個年份轉化為標準的 SQLite 轉置語法。
* group_concat(...)：
  將所有年份的 CASE WHEN 語句用逗號 , 串接在一起，變成一條長字串。
* 外層 printf：
  把串接好的轉置欄位，塞入 SELECT Product, ... FROM Sales GROUP BY Product; 的範本中。
* eval(...)：
  這是最關鍵的一步。它接收內層拼裝出來的完整 SQL 字串，並在 SQLite 引擎中即時編譯並執行，直接輸出最終的樞紐分析報表。

#### 使用此技巧的注意事項
* 記憶體限制：group_concat 預設有字串長度限制（通常為 1,000,000 字元）。如果您的變動欄位（如 Year）高達數萬個，可能會觸發上限。
* 安全風險：若 Year 欄位的資料來自使用者輸入，動態拼接可能引發 SQL 注入（SQL Injection）。請確保參與轉置的欄位資料是乾淨且受控的。

### 2. 實作範例
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

### 為什麼這個解法被奉為經典？（優勢剖析）
#### * 解耦多列與多欄的限制：
  eval() 在第一步只負責執行 DDL 語句（CREATE VIEW）。對 DDL 而言，它不需要回傳任何資料列，完美避開了 eval() 無法直覺回傳二維表格的硬傷。
#### * 完美的客戶端相容性：
  一旦 View 被動態建立完成，不論你後端是用 Python、Node.js、C#，還是任何簡單的報表 UI 工具，都只需要下一行最直覺的 SELECT * FROM v_dynamic_pivot 即可，完全不需要在後端程式碼裡處理噁心的字串拼接。
#### * 自動適應資料變更：
  每當資料庫新增了新的年份（例如多出了 2026, 2027），後端只需要在查詢前重新呼叫一次「步驟一」的 SQL，View 就會自動與時俱進，完成自動化元編程的閉環。「透過 eval() 驅動 DDL 來重建 View」的思維，確實是 SQLite 圈子裡處理動態樞紐分析最漂亮、含金量最高的操作。

## 四.搭配 Sqlean 的 define()
搭配 sqlean 的 define() 擴充功能，會把這套動態 Pivot 的元編程技巧推向另一個完全不同的層次：**將複雜的字串拼接與 DDL 重建邏輯，完美封裝成一個自訂的預存程序（Stored Procedure）或函式**。

在 [sqlean-define](https://github.com/nalgeon/sqlean/blob/main/docs/define.md) 模組中，除了剛剛提到的 eval() 外，最核心的就是 define(NAME, BODY) 函數。它允許我們在 SQLite 中直接用 SQL 語法定義客製化函數（UDF），甚至可以用它來定義**表值函數（Table-Valued Function）**。

以下為您展示如何將「動態覆蓋 View」的精髓，用 define() 漂亮地打包起來：

### 1. 經典封裝：將「重建檢視表」打包成一個命令
我們可以透過 define() 建立一個名為 refresh_pivot() 的自訂函數。未來每次資料有更新，只要下一行簡單的指令，背後噁心的字串拼接與 View 重建就會自動完成。
```sql
-- 【準備工作】載入 sqlean-define 擴充功能
.load ./define
-- =================================================================
-- 步驟一：使用 define() 封裝元編程邏輯
-- =================================================================
SELECT define(
    'refresh_pivot', -- 自訂函數名稱
    'eval(
        SELECT printf(
            ''DROP VIEW IF EXISTS v_dynamic_pivot; '' ||
            ''CREATE VIEW v_dynamic_pivot AS '' ||
            ''SELECT Product, %s FROM Sales GROUP BY Product;'',
            group_concat(
                printf(''SUM(CASE WHEN Year = \''''%s\'''' THEN Amount ELSE 0 END) AS "%s"'', Year, Year)
            )
        )
        FROM (SELECT DISTINCT Year FROM Sales ORDER BY Year)
    )'
);
```
#### 呼叫與查詢：
封裝完成後，您的後端程式碼或報表工具將會變得極度乾淨：
```sql
-- 1. 需要重新整理樞紐結構時，直接呼叫此函數（會自動偵測新舊年份並覆蓋 View）
SELECT refresh_pivot();
-- 2. 直接讀取完美的二維表格
SELECT * FROM v_dynamic_pivot;
```
---
### 2. 進階元編程：打造通用型動態 Pivot 產生器
利用 define() 可以接收參數的特性，我們甚至可以脫離單一資料表，寫出一個通用的「動態 Pivot 工具」。只要傳入「資料表名」與「要轉置的欄位名」，它就能自動幫你生出對應的 View：
```sql
-- 建立通用型動態檢視表產生器
SELECT define(
    'build_generic_pivot',
    'eval(
        SELECT printf(
            ''DROP VIEW IF EXISTS v_generic_pivot; '' ||
            ''CREATE VIEW v_generic_pivot AS '' ||
            ''SELECT Product, %s FROM '' || :table_name || '' GROUP BY Product;'',
            group_concat(
                printf(''SUM(CASE WHEN '' || :pivot_col || '' = \''''%s\'''' THEN Amount ELSE 0 END) AS "%s"'', value, value)
            )
        )
        FROM (SELECT DISTINCT ' || :pivot_col || ' AS value FROM ' || :table_name || ' ORDER BY value)
    )'
);
```
#### 使用方式：
```sql
-- 帶入參數：資料表為 'Sales'，想要轉置的欄位是 'Year'
SELECT build_generic_pivot('Sales', 'Year');
-- 隨時查詢通用 View
SELECT * FROM v_generic_pivot;
```
---
### 3.為什麼加上 define() 才是終極形態？

   **1. 實現 SQLite 缺少的預存程序（Stored Procedure）：**
   
   SQLite 官方因為輕量化設計而不支援 CREATE PROCEDURE。sqlean-define 實質上為 SQLite 補齊了這個高階功能，讓資料庫邏輯可以留在資料庫內。
   
   **2. 消滅後端的 SQL 字串地獄：**
   
   不需要在 Python 或 Node.js 程式碼裡塞一堆多行字串去拼湊 SQLite 的 DDL。後端工程師只需要知道 SELECT refresh_pivot(); 這一行命令，大幅提升代碼的可讀性與可維護性。
   
   **3. 元編程的閉環：**
   define() 負責結構的封裝，eval() 負責字串的動態執行，兩者結合，才算完美在 SQLite 中復刻了 SQL Server sp_executesql 或 Oracle EXECUTE IMMEDIATE 的進階元編程體驗。 [3] 

如果想進一步了解如何將這個機制與 SQLite 的觸發器（Trigger） 綁定，讓它在每次資料 INSERT 時「完全自動化」刷新欄位結構，可以告訴我您的應用場景！

## 五.加入 SQLite Trigger（觸發器）自動自動更新
這部分是非常經典的「數據驅動架構（Data-Driven Architecture）」閉環。透過 **SQLite Trigger（觸發器）**，我們可以做到：每當 Sales 資料表有新資料寫入（INSERT）或年份變更（UPDATE）時，資料庫會**自動在背後觸發元編程邏輯**，重新刷新 v_dynamic_pivot 檢視表的欄位結構。

如此一來，後端或前端報表工具甚至**完全不需要知道 refresh_pivot() 的存在**，只要專心 SELECT *，就能永遠拿到最新、欄位最正確的樞紐分析表！


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
