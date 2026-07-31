# SQLITE 經典的「動態樞紐分析(Dynamic Pivot)」或「元編程(Metaprogramming)」技巧

SQLite 純 SQL 的極限替代方案如果你堅持不想用外部程式，且在某些特定終端機介面下，SQLite 唯一的純 SQL 歪招是利用 printf 和 group_concat 拼出 SQL 字串，但你仍得手動複製該字串去執行：
```sql
-- 這段 SQL 會「輸出」另一段 SQL 程式碼
SELECT printf(
    'SELECT Product, %s FROM Sales GROUP BY Product;',
    GROUP_CONCAT(printf('SUM(CASE WHEN Year = %d THEN Amount ELSE 0 END) AS "%d"', Year, Year))
)
FROM (SELECT DISTINCT Year FROM Sales ORDER BY Year);
```

注意：此方法受限於 group_concat 的字串長度限制（預設 1,000,000 字元），且無法直接在同一個查詢中動態執行。
