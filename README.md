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

為了徹底解放後端，我們可以建立 **Trigger（觸發器）**。每當 `Sales` 表有任何影響欄位結構的異動（`INSERT`、`UPDATE`、`DELETE`）時，資料庫會自動執行 `refresh_pivot()`，實現真正**無感（Transparent）的自動化元編程**。

```sql
-- 1. 當有新銷售紀錄，或年份被修改時，觸發自動刷新
CREATE TRIGGER IF NOT EXISTS trg_sales_auto_pivot_insert
AFTER INSERT ON Sales
BEGIN
    SELECT refresh_pivot();
END;

CREATE TRIGGER IF NOT EXISTS trg_sales_auto_pivot_update
AFTER UPDATE OF Year ON Sales -- 只有年份欄位被修改時才觸發，優化效能
BEGIN
    SELECT refresh_pivot();
END;

CREATE TRIGGER IF NOT EXISTS trg_sales_auto_pivot_delete
AFTER DELETE ON Sales
BEGIN
    SELECT refresh_pivot();
END;
```

### 🎯 自動化閉環流暢體驗
有了 Trigger 之後，後端開發者的工作流將會簡化到不可思議：
```sql
-- 後端直接寫入全新年份的資料（例如目前表內只有 2024, 2025，現在寫入 2026）
INSERT INTO Sales (Product, Year, Amount) VALUES ('Apple', 2026, 500);

-- 什麼都不用做，直接查詢 View，這時候欄位已經完美多出了 "2026" 的實體欄位！
SELECT * FROM v_dynamic_pivot;
```

---

### ⚠️ 開發注意事項與架構優勢

#### 架構優勢
1. **零後端介入（Zero-Backend Overhead）**：透過 Trigger，後端完全不需要維護「何時該刷新 Pivot」的邏輯。不管是資料清洗、定時排程寫入，View 永遠處於最新狀態。
2. **消滅後端 SQL 字串地獄**：不需要在 Python/Node.js 程式碼中用多行字串去硬拼接 DDL，資料庫邏輯完美留在資料庫內。
3. **完美的相容性**：一旦 View 建立完成，任何簡單的 UI 報表工具或後端框架都可以用最直覺的 `SELECT * FROM v_dynamic_pivot` 讀取資料。

#### 注意事項與效能優化
* **高併發寫入效能**：由於 Trigger 會在每次 `INSERT` 時重新跑一次 `eval()` 與 `CREATE VIEW`（涉及系統表暫時鎖定），若系統有短時間內「萬級、百萬級列高頻寫入」的場景，**強烈建議不要用 Trigger**，應改為在批次寫入（Batch Import）完成後，手動呼叫一次 `SELECT refresh_pivot();` 以維護效能。
* **記憶體限制**：SQLite 的 `group_concat` 預設字串長度上限通常為 1,000,000 字元。若轉置後的欄位高達數萬個，需注意是否觸發上限。
* **安全性風險**：若參與轉置的資料（如 Year）包含使用者輸入的內容，必須防範 SQL 注入。此技巧建議僅用於**系統內部、資料受控**的分析報表場景。

## 六.define() 功能與  0x09/sqlite-statement-vtab 比較
將 sqlean 的 define()（本質上基於內置純量/表值函數擴充）與 0x09 大神的 sqlite-statement-vtab 進行比較，可以說是觸及了 SQLite 元編程的「兩大門派」。
事實上，sqlean-define 模組的底層在開發時，很大一部分靈感就是借鑑並改編自 0x09 的 sqlite-statement-vtab。兩者雖然都能在 SQLite 中實現類似預存程序的功能，但其核心機制、欄位定義時機以及對動態樞紐分析（Dynamic Pivot）的適用性有著本質上的區別。

---
### A.一張圖表看懂核心差異

| 特性比較 | sqlean 的 define() | 0x09 的 sqlite-statement-vtab |
|---|---|---|
| 技術本質 | 標量函數 (Scalar) / 表值函數庫 | 虛擬表機制 (Virtual Table) |
| 語法結構 | SELECT define('name', 'body') | CREATE VIRTUAL TABLE name USING statement(...) |
| 欄位結構決定時機 | 動態（編譯時才決定） | 靜態（虛擬表建立時就必須宣告） |
| 參數化查詢 | 支援（使用具名綁定變數 :param） | 支援（可直接作為 TVF 傳入參數） |
| 動態 DDL 執行 | 支援（透過內置 eval() 跑 DDL） | 不支援（僅限 Prepared Statement 查詢） |
| 對動態 Pivot 的難易度 | 極其簡單（配合 View 與 Trigger 閉環） | 極其困難（甚至無法原生做到） |

------------------------------
### B.為什麼在「動態 Pivot」場景，sqlean 的 define() 完勝？
關鍵在於 「欄位名稱與數量的決定時機」。
#### 1. 0x09 sqlite-statement-vtab 的限制：
statement-vtab 的本質是建立一個虛擬表（Virtual Table）。SQLite 的虛擬表在宣告的那一刻（CREATE VIRTUAL TABLE），就必須明確告訴 SQLite 引擎這張表有哪些欄位、欄位名稱是什麼。
當你嘗試用它來做動態 Pivot 時，因為你的年份（如 2024, 2025, 2026...）是隨著資料動態增長的，statement-vtab 無法在不重新建立虛擬表的情況下，動態增加回傳的欄位數量。它適合的是「參數化查詢（帶參數的 View）」，例如：
```sql
-- statement-vtab 的典型強項：參數化查詢，但欄位（Product, Amount）是寫死的
CREATE VIRTUAL TABLE get_sales_by_year USING statement((
    SELECT Product, Amount FROM Sales WHERE Year = :year
));
-- 查詢時帶入參數
SELECT * FROM get_sales_by_year(2026);
```
#### 2. sqlean 的 define() + eval() 的降維打擊：
誠如您前面所提到的靈感，sqlean 之所以能搞定動態 Pivot，是因為它跳脫了查詢限制，改用語法去「驅動 DDL 來重建實體 View」。 

* define() 裡面包的是 eval()。
* eval() 跑的是 DROP VIEW 與 CREATE VIEW 這種會重塑資料庫結構的指令。
* 重新建立的實體 View，其欄位是交由 SQLite 的 SQL 解析器在執行 DDL 時當場計算出來的。 [4] 

這使得 sqlean 的方案具有極致的動態性，欄位能隨著資料橫向無限延伸。

---
### C.技術選型建議：兩者該怎麼選？
0x09 的 sqlite-statement-vtab 是「無法」用來建立純量（Scalar）SQL 函數的。

這是由它的技術本質所決定的。
#### 核心原因：它是「虛擬表（Virtual Table）」，不是「函數」
在 SQLite 的底層架構中，擴充功能主要分為兩大派系：
* **1.使用者自訂函數 (UDF)：**
  像是 sqlean-define 建立的 calculate_discount()。它輸入一個或多個參數，運算後**回傳單一數值（純量）**。
* **2.虛擬表 (Virtual Table)：**
  這正是 0x09 sqlite-statement-vtab 的技術基礎。在 SQLite 中，虛擬表的本質**永遠是一張二維表格（Table）**，它必須透過 FROM 子句來呼叫。
#### 語法上的致命限制因為
sqlite-statement-vtab 的語法是 CREATE VIRTUAL TABLE ...，就算您用它來包裝一段只會回傳「單一數值」的 SELECT 語句，它在 SQLite 眼中依然是一張 **「只有 1 列、1 欄的表格」。**

這會導致在實際撰寫 SQL 時非常不直覺：
##### ❌ 0x09 辦不到的「純量函數」寫法：
您無法像內建函數（如 ABS()、UPPER()）那樣，直接把它塞在 SELECT 的欄位清單中：
```sql
-- 這是錯誤語法！SQLite 會報錯，因為 get_discount 是個虛擬表，不能當純量函數用
SELECT Product, get_discount(Category, Amount) FROM Sales;
```
##### ⚠️ 0x09 必須妥協的「子查詢」寫法：
如果您硬要用 0x09 實現純量運算，您必須強行使用關聯子查詢（Correlated Subquery）或 JOIN，語法會變得極度臃腫：
```sql
-- 必須用 FROM 呼叫，並用子查詢包裝，效能與可讀性極差
SELECT 
    Product, 
    (SELECT discount FROM get_discount_vtab(Sales.Category, Sales.Amount)) AS Discount
FROM Sales;
```
#### 什麼時候該用 0x09 的 sqlite-statement-vtab？

* 不需要變動欄位數量：只是想做類似 PostgreSQL 或 SQL Server 的「表值函數（Table-Valued Function）」。
* 需要極致的查詢效能：因為它是將 SQL 編譯為 Prepared Statement 快取在連線中，如果是單純的固定欄位參數化查詢，它的執行速度與記憶體效率會比頻繁拼接字串的 eval() 高非常多。 [1] 
* 不想動到 Schema：不想在資料庫裡频繁 DROP/CREATE 任何 View，希望保持 MetaData 的乾淨。

#### 什麼時候必須用 sqlean 的 define() + eval()？

* 欄位會隨著資料列改變（即本主題的 Dynamic Pivot）。
* 需要執行非 SELECT 語句：例如要在自訂函數中順便執行 INSERT、UPDATE、或動態建表（DDL）。
* 需要打造一個通用型的 SQL 產生器（如前面示範的傳入表名、欄位名就能自動組裝結構的工具）。

### D.總結
0x09 的 sqlite-statement-vtab 是一個優雅的參數化查詢封裝器；而 sqlean 的 define() 搭配 eval() 則是一個真正的動態程式碼執行引擎（元編程核心）。要解決「動態樞紐分析」這種欄位不確定的硬骨頭，只有後者這種具備動態重塑結構能力的工具才能完美勝任。 
#### 選型決策
1. **必須選 `sqlean-define` 的場景**：需要處理 **Dynamic Pivot** 等欄位數量不固定的場景，或是函數內部需要動態組裝字串、執行 `INSERT/UPDATE/DDL` 等非唯讀操作。
2. **可以選 `statement-vtab` 的場景**：純粹做固定欄位的「參數化查詢（如上述的表值函數）」


## 七.define() 功能續談
是的，這正是 define() 最迷人的地方。**它可以用「純 SQL」把一段複雜的 SQL 邏輯封裝成一個自訂函數（UDF），在 SQLite 中完美扮演了「預存程序（Stored Procedure）」或「自訂函式（Function）」的角色。**

雖然 SQLite 官方因為輕量化設計，一直不肯加入 CREATE FUNCTION 或 CREATE PROCEDURE 語法，但 sqlean-define 透過 define() 函數，以一種非常巧妙的方式補齊了這個遺憾。

以下為您說明它如何包裝純 SQL，以及它的兩種包裝形態（純量函數與表值函數）：
### 1. 包裝「純量函數」（回傳單一數值）
假設你有一段複雜的純 SQL 商業邏輯，例如「根據產品類別與銷售量計算折扣率」，原本每次查詢都要寫一長串 CASE WHEN，現在可以直接包裝起來：
```sql
-- 封裝純 SQL 邏輯（內部使用標準 SQL 語法）SELECT define(
    'calculate_discount', -- 函數名稱
    'CASE 
        WHEN :category = "Electronics" AND :amount > 1000 THEN 0.15
        WHEN :category = "Electronics" THEN 0.05
        ELSE 0.02
     END'                  -- 純 SQL 運算邏輯，`:變數` 為傳入參數
);
-- 後端或查詢時，直接像內建函數一樣使用它：SELECT Product, calculate_discount(Category, Amount) AS DiscountFROM Sales;
```
---
### 2. 包裝「表值函數」（Table-Valued Function，回傳一張表）
define() 更強大的地方在於，它可以用純 SQL 包裝一整段 SELECT 查詢，並讓它表現得像一張動態的資料表。這可以完美替代「帶參數的 View（Parameterized View）」：
```sql
-- 封裝一個純 SQL 的 SELECT 查詢SELECT define(
    'get_high_sales', -- 函數名稱
    'SELECT Product, Amount 
     FROM Sales 
     WHERE Year = :target_year AND Amount >= :min_amount'
);
-- 查詢時，直接把函數當成資料表（Table）來用，並傳入參數：SELECT * FROM get_high_sales(2026, 500);
```
---
### 與常規元編程的本質區別
在前面聊到「動態 Pivot」時，我們是在 define() 裡面包了 eval()，那是因為 Dynamic Pivot 本質上需要**動態拼接字串並執行 DDL。**

但如果只是普通的商務邏輯封裝（如上述的折扣計算、參數化篩選），define() **完全不需要 eval() 的介入。**它就是純粹地把那段「純 SQL 語句」編譯並快取起來。
### 總結
define() 的出現，讓 SQLite 擁有了與大型資料庫（TSQL、PL/SQL）平起平坐的封裝能力：

* 以前：這些邏輯必須寫在 Python / Node.js 等後端程式碼裡。
* 現在：你可以用「純 SQL」將邏輯直接寫在資料庫裡，打包成一個個乾淨的函數。

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
