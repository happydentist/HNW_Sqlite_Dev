sqlpage show grid data and push one data column and show relative another table


在 SQLPage 中，如果你想要展示一個主資料表（Grid/Table），並在點擊其中一個資料欄位（或按鈕）時，將該列的 ID 傳遞（Push）出去，並在同一個頁面上動態顯示另一個關聯資料表的詳細內容，最標準且優雅的做法是利用網址參數（URL Parameters / Query Strings）搭配 SQL 的 JOIN 或 WHERE 子句。 [1, 2] 
以下為您示範如何使用 SQLPage 內建的 table 元件實作這個「主從式表單（Master-Detail）」功能。

------------------------------
## 實作邏輯

   1. 主表格（Master Grid）： 顯示第一張表的資料（例如：customers 顧客表）。我們利用 link 參數將顧客的 ID 帶入網址（例如：?selected_id=1）。
   2. 條件判斷： 檢查網址有沒有帶 $selected_id 參數。
   3. 副表格（Detail Table）： 如果有參數，就在下方或旁邊渲染第二個 table 元件（例如：orders 訂單表），並透過 WHERE customer_id = $selected_id 過濾出關聯資料。

------------------------------
## 完整 SQLPage 程式碼範例
將以下程式碼儲存為您的 .sql 檔案（例如 index.sql）：

   1. 設定頁面標題
   ```
   select 'list' as component, '主從式資料展示範例' as title;
   ```
   2. 渲染主表格：展示顧客清單
   ```
   select 'table' as component, 
          '顧客清單 (點擊查看詳情)' as title,
           true as search,   -- 啟用 SQLPage 內建搜尋功能
           true as sort;     -- 啟用內建排序功能
   ```
   在這裡，我們把 '查看訂單' 欄位包裝成一個連結，點擊時會重整頁面並帶入 ?selected_id=...
   ```
   select id as "顧客編號",
          name as "姓名",
          email as "電子郵件",
          '查看訂單 ➔' as "操作",
          -- 使用 sqlpage.current_page() 保持在同網頁，並附加網址參數
          sqlpage.current_page() || '?selected_id=' || id as "操作_link"from customers;
   ```
   3. 檢查是否有動態點擊傳遞過來的 $selected_id-- 如果網址有 ?selected_id=XX，則觸發渲染第二張關聯表select 'table' as component,
       '該顧客的歷史訂單紀錄' as titlewhere $selected_id is not null; -- 只有當參數存在時才顯示此元件
   ```
   select o.order_id as "訂單編號",
          o.order_date as "下單日期",
          o.total_amount as "總金額",
          o.status as "訂單狀態"from orders owhere o.customer_id = cast($selected_id as integer) -- 透過網址參數過濾關聯資料order by o.order_date desc;
   ```
------------------------------
## 💡 關鍵語法解析

* "欄位名稱_link" 機制：在 SQLPage 的 table 元件中，如果您有一個欄位叫作 操作，只要在 SQL 中同時選取一個名為 操作_link 的虛擬欄位，SQLPage 就會自動將「操作」欄位變成可以點擊的超連結。
* $selected_id：這是 SQLPage 自動從網址列抓取變數的魔法語法。當使用者點擊 ?selected_id=5 時，底下的副表格就會自動重新載入並只撈取 customer_id = 5 的資料。
* sqlpage.current_page()：確保不論你的檔案叫什麼名字，點擊時都會刷新在當前頁面，體驗非常流暢。 [3] 
* 

如果您想要調整版面配置，我可以為您提供更進階的排版方式：

* 是否希望將兩個表格改成**左右並排（利用 card 或遮罩層）**而非上下排列？
* 您的資料庫使用的是哪一種（如 SQLite、PostgreSQL 或 MySQL），有需要協助處理特定的資料型態轉換嗎？


[1] [https://stackoverflow.com](https://stackoverflow.com/questions/14653366/how-to-connect-column-in-datagridview-to-different-sql-table)
[2] [https://stackoverflow.com](https://stackoverflow.com/questions/54594765/how-to-display-data-in-sql-from-multiple-tables-but-only-if-one-column-data-mat)
[3] [https://sql-page.com](https://sql-page.com/documentation.sql)
