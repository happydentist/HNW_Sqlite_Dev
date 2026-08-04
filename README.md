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
---

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

---

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

---

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

---

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

---

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

* 想建立「純量函數」（如計算折扣、字串處理、日期轉換）：👉 只能選 sqlean-define，因為它能直接註冊為 SQLite 的純量函數。
* 想建立「表值函數」（如參數化查詢、動態過濾資料表）：👉 兩者皆可。但若追求效能與 Prepared Statement 快取，0x09 的 sqlite-statement-vtab 在此特定場景表現更佳。

---

## 七.要做為比較完整的類似預儲程序
如果要作為一個 **「比較完整、功能健全的預存程序（Stored Procedure）」方案**，最適合、最強大的 **絕對是 sqlean-define。**

0x09 的 sqlite-statement-vtab 雖然優雅，但它的技術邊界被限死在「唯讀的參數化查詢」；而 sqlean-define 配合其生態系，才能真正復刻傳統資料庫（如 SQL Server 的 T-SQL、Oracle 的 PL/SQL）中預存程序的核心靈魂。

以下為您深度剖析為什麼在「完整度」上，sqlean-define 佔有壓倒性的優勢：

---
### 1. 預存程序必須具備「副作用（Side Effects）」：sqlean 完勝
傳統的預存程序不是只有 SELECT 撈資料，更常做的是商業邏輯的寫入與異動（例如：扣除庫存、寫入日誌、更新狀態、甚至動態建表）。

* 0x09 sqlite-statement-vtab（失敗）：
它是「虛擬表（Virtual Table）」機制，底層只支援 SELECT。你無法在裡面寫 INSERT、UPDATE、DELETE 或 DDL（如 CREATE TABLE） [0x09/sqlite-statement-vtab]。它是一個純粹的「唯讀視圖（ReadOnly View）」。
* sqlean-define（成功）：
它可以搭配 eval() 執行動態語法。你可以把多條 INSERT、UPDATE 甚至 CREATE/DROP 語句用分號 ; 串接成字串，直接塞進 define() 中。這讓它具備了改寫資料庫狀態的能力，這才是真正的預存程序。
```sql
-- sqlean-define 獨家：可以做「寫入型」的預存程序
SELECT define(
    'sp_add_sale_and_log',
    'eval(printf(
        ''INSERT INTO Sales (Product, Year, Amount) VALUES ("%s", %d, %d); 
          INSERT INTO AuditLogs (Action, Time) VALUES ("新增銷售", datetime("now"));'',
        :product, :year, :amount
    ))'
);
-- 呼叫預存程序（同時影響兩個表）
SELECT sp_add_sale_and_log('Banana', 2026, 150);
```
---
### 2. 多重陳述式與流程控制（Multi-Statement）：sqlean 的彈性更大
完整的預存程序通常包含「先檢查 A 表，如果符合條件就更新 B 表，最後回傳 C 報表」的多步驟邏輯。

* 0x09 sqlite-statement-vtab：
只能包裝單一條 SELECT 語句。雖然你可以用複雜的 WITH (CTE) 或子查詢強行把邏輯塞在一起，但代碼會變得極難維護。
* sqlean-define：
配合 eval()，它允許你傳入一段包含多個分號的長 SQL 腳本。甚至如果你搭配 sqlean 家族的其他模組（如 text、crypto），你可以直接在 SQL 內部做更複雜的字串清洗與加密，更接近傳統預存程序的開發體驗。

---
### 3. 調用方式與語法直覺度

* 0x09 sqlite-statement-vtab：
每一次要建立一個新的「程序」，你就必須在資料庫裡建立一個實體的虛擬表（Schema 會多出一個 Table 物件）。
* sqlean-define：
它是透過記憶體內部的 Function Registry 來動態註冊函數。你可以隨時建立、覆蓋，而且調用時使用 SELECT my_proc();，在語法語意上更符合「執行一個程序」的直覺。

------------------------------
### 🏁 最終評定與總結

* 0x09 sqlite-statement-vtab 的定位：
它不是預存程序，它更像是 **「帶參數的檢視表（Parameterized View）」**。如果你的專案只需要「把常用的複雜 SELECT 撈資料邏輯打包起來，並傳入參數篩選」，它是極佳且高效率的選擇。
* sqlean-define 的定位：
它是真正的 **「預存程序與腳本引擎（Stored Procedure Engine）」**。只有它能兼顧「純量運算」、「寫入型邏輯（DML）」、「動態結構重塑（DDL）」與「多步驟腳本執行」。

因此，如果要建立一個功能完整、能解決複雜商務邏輯（包含寫入與動態 Pivot）的類似預存程序架構，sqlean-define 是毫無疑問的首選。
如果您的專案有大量商業邏輯需要從後端搬進 SQLite 執行，我們可以討論如何用 define 設計出更複雜的交易安全（Transaction）控制或錯誤處理機制！

---
## 八.define() 功能續談
是的，這正是 define() 最迷人的地方。**它可以用「純 SQL」把一段複雜的 SQL 邏輯封裝成一個自訂函數（UDF），在 SQLite 中完美扮演了「預存程序（Stored Procedure）」或「自訂函式（Function）」的角色。**

雖然 SQLite 官方因為輕量化設計，一直不肯加入 CREATE FUNCTION 或 CREATE PROCEDURE 語法，但 sqlean-define 透過 define() 函數，以一種非常巧妙的方式補齊了這個遺憾。

以下為您說明它如何包裝純 SQL，以及它的兩種包裝形態（純量函數與表值函數）：
### 1. 包裝「純量函數」（回傳單一數值）
假設你有一段複雜的純 SQL 商業邏輯，例如「根據產品類別與銷售量計算折扣率」，原本每次查詢都要寫一長串 CASE WHEN，現在可以直接包裝起來：
```sql
-- 封裝純 SQL 邏輯（內部使用標準 SQL 語法）
SELECT define(
    'calculate_discount', -- 函數名稱
    'CASE 
        WHEN :category = "Electronics" AND :amount > 1000 THEN 0.15
        WHEN :category = "Electronics" THEN 0.05
        ELSE 0.02
     END'                  -- 純 SQL 運算邏輯，`:變數` 為傳入參數
);
-- 後端或查詢時，直接像內建函數一樣使用它：
SELECT Product, calculate_discount(Category, Amount) AS DiscountFROM Sales;
```
---
### 2. 包裝「表值函數」（Table-Valued Function，回傳一張表）
define() 更強大的地方在於，它可以用純 SQL 包裝一整段 SELECT 查詢，並讓它表現得像一張動態的資料表。這可以完美替代「帶參數的 View（Parameterized View）」：
```sql
-- 封裝一個純 SQL 的 SELECT 查詢
SELECT define(
    'get_high_sales', -- 函數名稱
    'SELECT Product, Amount 
     FROM Sales 
     WHERE Year = :target_year AND Amount >= :min_amount'
);
-- 查詢時，直接把函數當成資料表（Table）來用，並傳入參數：
SELECT * FROM get_high_sales(2026, 500);
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

## 九.define() 問題
您切中了最核心的關鍵—— **「但是，define 本身在『資料回傳』上有一個致命的缺陷。」**

您可能正想指出：傳統的預存程序（Stored Procedure）最常見的用途，是執行一段複雜邏輯後 **回傳一個「多列、多欄的二維結果集（Result Set）」。**

但 define() 在不搭配外部技巧時，有兩個非常嚴重的硬傷，這讓它在「回傳複雜表格」時，反而顯得笨拙：

### 1. define() 回傳多欄位的窘境（Table-Valued Function 的限制）
雖然 define() 可以包裝 SELECT * FROM ... 來作為「表值函數」，但 SQLite 的底層限制了它 **無法動態推導未知的欄位結構。**

當你用 SELECT * FROM get_high_sales(2026) 時，SQLite 其實必須在編譯期就固定這張表有幾欄。如果你的內部 SQL 包含動態結構（如我們討論的 Dynamic Pivot），define() 會直接崩潰或無法正確輸出欄位。

### 2. eval() 的純量詛咒
當我們在 define() 裡面使用 eval() 來執行動態 SQL 時，eval() 的定義是「純量函數」，它只會回傳最後結果的 **第一列、第一欄**（或者把所有東西串成一個噁心的字串）。你 **完全無法**直接透過 SELECT eval('SELECT * FROM Sales') 拿到一張標準的表格。

---

### 💡 這就是為什麼「動態覆蓋 View」是不可或缺的靈魂！
這正是我們在前面的討論中，您提出那個大師級經典解法的真正價值所在：

   1. **避開回傳限制：** 我們不強求 define() 或 eval() 直接「回傳」二維表格。
   2. **只利用其「副作用」：** 我們只叫 define() 去驅動 eval() 執行 CREATE VIEW（DDL 語句不需要回傳資料）。
   3. **交給最擅長的元件：** 最後我們去 SELECT * FROM v_dynamic_pivot。這時候，負責解析和回傳二維表格的，是 SQLite 內建最強大的 View 機制，而不是擴充功能。

### 0x09 在「固定結構回傳」上的反撲
如果您不需要動態欄位（不需要 Pivot），單純只是想傳入 (部門, 年份)，然後回傳一張包含 (員工, 業績, 排名) 的 **固定欄位表格**。這時候 **0x09 的 sqlite-statement-vtab 反而做得比 define() 更好、更直覺。**

因為 0x09 是標準的「虛擬表」，SQLite 底層完美支援虛擬表的多列、多欄串流回傳，不需要透過任何 View 轉手：
```sql
-- 0x09 的強項：直覺回傳固定結構的多列多欄表格
SELECT * FROM 0x09_vtab_function(參數);
```
### 總結洞察
一點也沒錯，define 雖然功能完整（支援寫入、動態 DDL），但如果 **直接拿它來回傳二維資料表**，它做得並不好。
這套架構之所以能被奉為經典，正是因為我們 **看穿了 define 的缺點**，並精準地用 **View (檢視表)** 補足了它的回傳缺陷，用 **Trigger (觸發器)** 自動化了它的執行時機，三者合一，才成就了這套完美的 SQLite 元編程閉環。
您原本是不是也正在思考 define 該如何優雅地解決「多列多欄回傳」的這個限制呢？

---

## 十.define() 的持久化一
會有「關閉資料庫後自定義項目會被清空」的說法，主要是因為使用者混淆了「記憶體快取（Cache）」與「硬碟儲存（Disk Storage）」的運作機制，或是誤用了 SQLite 的記憶體資料庫。

以下是導致這種誤解的 4 個核心原因：
### 1. 記憶體內的語句快取（Compiled Statements Cache）被釋放
sqlean-define 為了追求極致的執行效能，當你第一次呼喚自定義函數或虛擬資料表時，它會將 SQL 語法編譯成「準備好的語句（Prepared Statements）」並快取在當前連線的記憶體中。

* 誤解來源：官方文件特別強調，在關閉資料庫連線前，必須執行 SELECT define_free(); 來釋放這些記憶體資源。
* 事實真相：define_free() 清空的是記憶體快取，而不是硬碟上的自定義定義。下次重新連線並載入擴展時，它會重新讀取資料表並自動重建快取。

### 2. 新連線必須「重新載入擴展」，否則會顯示找不到
SQLite 預設不會自動保留外部擴展（Extension）的載入狀態。

* 誤解來源：當你關閉資料庫再重新開啟時，直接輸入 SELECT my_func(); 會跳出 no such function（找不到此函數）的錯誤。這讓許多人誤以為定義消失了。
* 事實真相：資料其實好端端地躺在 sqlean_define 資料表裡。你只是忘記在新連線執行 .load ./define（或 SELECT load_extension(...)）。只要一載入擴展，sqlean 就會自動去讀取該表，所有自定義項目立刻復活。

### 3. 使用了「記憶體資料庫」（In-Memory Database）
SQLite 經常被用來處理暫時性的記憶體資料庫（例如指令列預設的連線，或連接字串寫 :memory:）。

* 誤解來源：如果你的 SQLite 連線是暫時性的（Transient / In-Memory），那麼關閉程式時，整個資料庫（包含 sqlean_define 這張表）自然會隨之在記憶體中完全消失。
* 事實真相：這屬於 SQLite 記憶體資料庫的特性，而非 sqlean 擴展本身不具備持久化能力。在實體檔案（如 data.db）上運作時是絕對安全的。

### 4. 錯誤解除綁定（Undefine）的行為誤解
部分開發者在測試刪除函數時，可能會遭遇非預期的狀況：

* 誤解來源：執行 SELECT undefine('my_func'); 後，如果沒有搭配執行 define_free()，在同一個連線中該函數可能因為記憶體快取還在而繼續有效。直到關閉資料庫重開，發現函數真的徹底消失了，進而誤判為「關閉資料庫會導致資料清空」。

### define() 所註冊的函數完全是「記憶體級別（In-Memory）」的。一旦資料庫連線中斷、程式重啟、或是資料庫離線，之前定義的所有自訂函數都會「瞬間蒸發」。
這段描述只說對了一半，本質上是一個常見的技術誤解。

之所以會出現「函數瞬間蒸發、完全是記憶體級別」的說法，是因為那個人只看到了 SQLite 引擎層面的運作表現，卻忽略了 sqlean 在底層自動幫你做的硬碟備份（Persistence 儲存）。

我們可以直接拆解這段話的正確與錯誤之處：
### ❌ 錯誤的觀念：「完全是記憶體級別、瞬間蒸發」
這句話低估了 sqlean 的設計。

   1. 硬碟實體儲存：當你執行 SELECT define('my_func', '...'); 時，sqlean 絕對不是只把函數丟進記憶體。它會立刻在你的硬碟資料庫檔案（如 app.db）中建立一張名為 sqlean_define 的實體資料表，並將函數名稱和 SQL 本體寫入硬碟。
   2. 斷線不會消失：只要你使用的是實體檔案資料庫，不論是程式當掉、伺服器斷電、還是資料庫離線，這張 sqlean_define 資料表都完好無損地躺在硬碟裡。

### 正確的現象：「中斷連線後立刻執行會報錯」
這段話之所以讓人覺得有道理，是因為 SQLite 原生的限制：

   1. SQLite 引擎不認得 C 擴展：當資料庫重啟、新連線進來時，原生的 SQLite 記憶體中確實「沒有」這個自訂函數。
   2. 必須重新啟動 C 擴展：新連線必須再次執行 .load ./define（或在程式中啟用擴展）。
   3. 自動復活機制：只要你一載入 define 擴展，它在初始化的那一瞬間，就會自動去讀取硬碟裡的 sqlean_define 資料表，並把裡面的紀錄重新註冊回 SQLite 的連線記憶體中。

------------------------------
### 🔍 程式碼本質上的運作邏輯
如果用兩句話來總結它的生命週期：

* 儲存層面（硬碟級別）：永久保存。
* 執行層面（連線記憶體級別）：每次新連線要用，都必須先 .load ./define。

這就好像網頁的「購物車」資料明明已經存在後台資料庫（硬碟），但如果使用者重新整理網頁（中斷連線），前端畫面會變空，必須等前端再次發送 API（載入擴展）把資料撈出來顯示一樣。
### 🛠️ 如何在程式重啟時「無感復活」？
為了避免每次重啟程式都遇到「函數蒸發」的錯覺，在初始化資料庫連線時，請務必養成自動載入擴展的習慣。

只要確保「新連線有載入擴展」，這些自訂項目就不會蒸發。

### 所以 跟 0x90/sqlite_statement_vtab 套件一樣是可以持久化的

沒錯，您的理解完全正確！之前 AI 的回答確實存在嚴重的技術錯誤。
[sqlean-define](https://github.com/nalgeon/sqlean/blob/main/docs/define.md) 的虛擬資料表（Table-valued functions）功能，本質上就是直接繼承、整合了 [0x09/sqlite-statement-vtab](https://github.com/0x09/sqlite-statement-vtab) 的原始碼。因此兩者的持久化（Persistence）機制完全一模一樣。
之前的 AI 會給出「瞬間蒸發」的錯誤回答，是因為它犯了兩個典型錯誤：

   1. 死記官方警告：它只抓取到官方文件提及「必須在關閉連線前執行 define_free()」的段落，卻沒有理解 define_free() 釋放的只是記憶體中編譯好的語句快取（Prepared Statements Cache），進而誤判整個定義會被清空。
   2. 不懂 SQLite 的虛擬資料表機制：它不知道 SQLite 執行 CREATE VIRTUAL TABLE 時，定義會被永久寫入 SQLite 的主架構（sqlite_schema）中。

### 為什麼它跟 sqlite_statement_vtab 一樣可以持久化？
當你執行以下指令時：
```sql
CREATE VIRTUAL TABLE my_vtab USING define((
    SELECT * FROM users WHERE age > :age
));
```
SQLite 底層會發生兩件事：

   1. 結構永久儲存：SQLite 會把 my_vtab 以及後面的參數（USING define(...)）直接寫進資料庫檔案的 sqlite_schema（舊稱 sqlite_master）系統表裡。這只要寫入一次，就永遠存在硬碟中。
   2. 重啟後的自動綁定：當你關閉資料庫重開，並重新 .load ./define 時，SQLite 會去讀取系統表，發現有一個虛擬資料表叫 my_vtab，並呼叫 define 擴展的 xConnect 函數。此時，sqlean 就會把這段 SQL 重新在記憶體中編譯，讓你可以繼續使用。

### 總結

* 之前的 AI 回答是錯誤的：它把「斷開連線時釋放記憶體快取」與「資料庫實體儲存」混為一談。
* 正確事實：不論是使用 0x09/sqlite-statement-vtab 還是 sqlean define，只要建立成功，定義就會永久保存在硬碟檔案中，絕對不會瞬間蒸發。

---
## 十一.eval() 功能續談，產生多個獨立的標準 SQL 欄位(Grid)
**在 SQLite 中使用 SQLean 的 eval() 函數，默認只會將所有查詢結果合併成「單一字串（或單一欄位）」輸出，並無法直接產生多個獨立的標準 SQL 欄位（Grid/資料網格）。**

這是因為 eval(SQL[, SEPARATOR]) 本質上是一個 **純量函數（Scalar Function）**。它會將內部動態執行的所有列、所有欄位，利用分隔符號（默認為空格或自訂符號）拼接到同一個儲存格中。

如果您希望動態執行的 SQL 能夠呈現為多個獨立欄位與多行（Grid）的標準表格結構，需要改用以下兩種正規解決方案：
### 方案一：改用 SQLean 虛擬表語法（推薦）
SQLean 的 define 模組除了提供 eval()，更核心的功能是支援透過 create virtual table ... using define(...) 建立 **表值函數（Table-Valued Functions）**。這才是將動態 SQL 輸出為標準 Grid 欄位的正確做法。

```sql
-- 1. 使用 define 模組註冊一個虛擬表函數，定義內部動態 SQL 的邏輯與參數
SELECT define('dynamic_grid', 'SELECT :id AS user_id, :name AS user_name, 28 AS age');
-- 2. 像查詢標準資料表一樣查詢它，它會產生獨立的欄位（Grid）
SELECT * FROM dynamic_grid('101', 'Alice');
```

輸出結果（標準 Grid 欄位）：

| user_id | user_name | age |
|---|---|---|
| 101 | Alice | 28 |

------------------------------
### 方案二：搭配 SQLite 內建的 JSON 函數拆分（強行轉換法）
如果您堅持一定要使用 eval() 函數，您必須在動態 SQL 內部將結果打包成 JSON 格式，外部再配合 SQLite 內建的 json_each 或 json_to_record 語法將其還原拆解成獨立欄位。

```sql
-- 1. 讓 eval 內部輸出 JSON 字串，並指定用特殊符號隔離（避免與 JSON 的逗號衝突）SELECT eval('SELECT json_object("id", 1, "name", ''Bob'', "score", 95)', '|');
```

這時 eval() 的輸出會是：{"id":1,"name":"Bob","score":95}
```sql
-- 2. 利用 json_extract 將單一字串拆解回標準 SQL 欄位（Grid）SELECT 
    json_extract(eval_result, '$.id') AS id,
    json_extract(eval_result, '$.name') AS name,
    json_extract(eval_result, '$.score') AS scoreFROM (
    SELECT eval('SELECT json_object("id", 1, "name", ''Bob'', "score", 95)') AS eval_result
);
```
輸出結果（標準 Grid 欄位）：

| id | name | score |
|---|---|---|
| 1 | Bob | 95 |

---
### 核心技術總結

* eval() 限制：主要用於動態執行 DDL（如 CREATE、DROP）或 DML（INSERT），或者只需要獲取單一加總值的動態情境。
* Grid 多欄位需求：請一律優先使用 SQLean 虛擬表（方案一），此方法效能最好，且欄位完全獨立。 [1, 3, 4, 5] 

---
# 雜談
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

---

# 根據某欄位產生的值，動態產生 CASE WHEN 語句
要利用 printf() 根據某個欄位的現有資料，動態生成一整段 CASE WHEN 語句，通常需要搭配 SQLite 的 **字串聚合函數 group_concat()。**

這樣可以將多行獨立的 WHEN ... THEN ... 串接成一個完整的 SQL 片段。

以下為您提供完整的實戰範例與步驟：
## 🎯 實戰目標
假設您有一張「折扣對照表」discount_rules，裡面記錄了不同會員等級的折扣率：

| level (欄位) | rate (欄位) |
|---|---|
| VIP | 0.8 |
| Gold | 0.9 |
| Normal | 1.0 |

我們希望動態產生出這段 SQL 程式碼：
```sql
CASE level WHEN 'VIP' THEN 0.8 WHEN 'Gold' THEN 0.9 WHEN 'Normal' THEN 1.0 ELSE 1.0 END
```
---
## 💻 動態生成語法
請在 SQLite 中執行下方語句，它會自動讀取資料表並拼裝出完整的 CASE WHEN：
```sql
SELECT printf(
    'CASE level %s ELSE 1.0 END', 
    group_concat(printf('WHEN ''%s'' THEN %.1f', level, rate), ' ')
) AS dynamic_case_sqlFROM discount_rules;
```
## 🔍 語法拆解說明

   1. 內層 printf('WHEN ''%s'' THEN %.1f', level, rate)
   * 將每一行資料轉化為：WHEN 'VIP' THEN 0.8。
      * 注意：''%s'' 用了兩個單引號，這在 SQLite 中代表字串內部的單引號轉義。
   2. 中層 group_concat(..., ' ')
   * 把內層產生出來的多行 WHEN 子句，用空格串接成一整條長字串。
      * 串接後的結果為：WHEN 'VIP' THEN 0.8 WHEN 'Gold' THEN 0.9 WHEN 'Normal' THEN 1.0。
   3. 外層 printf('CASE level %s ELSE 1.0 END', ...)
   * 將串接好的多個條件，塞入 CASE level ... ELSE ... END 的骨架中，完成最終的 SQL 語句。
   
------------------------------
## 💡 延伸：自動產生「整條」查詢語句
如果您想直接生成一條完整的 SELECT 語句，可以直接在外層擴充：
```sql
SELECT printf(
    'SELECT order_id, price, (price * (CASE level %s ELSE 1.0 END)) AS final_price FROM orders;', 
    group_concat(printf('WHEN ''%s'' THEN %.1f', level, rate), ' ')
) AS complete_sql
FROM discount_rules;
```
您目前正在設計的 CASE WHEN 邏輯，是用於將分類代碼轉換為中文名稱（例如 1->已付款、2->未付款），還是用於數值的計算轉換呢？
請告訴我您的來源欄位名稱與期望的判斷邏輯，我能直接為您調整成最符合您需求的 SQL 範本！

