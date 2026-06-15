# Backend 筆記：Database Index

## 一句話

Database index（資料庫索引）用額外的資料結構記住「欄位值對應到哪些資料位置」，讓資料庫不必逐筆掃描整張表，也能快速找到符合條件的資料。

## 我的理解

假設 `users` 表有一百萬筆資料，而我們要用 email 找某個使用者。沒有 index 時，資料庫可能要從第一筆一路比對到最後一筆；如果 `email` 有 index，資料庫可以先在索引中找到 email 對應的資料位置，再讀取真正的資料。

使用者不需要知道目標資料是第幾筆，也不需要在 SQL 裡手動指定索引。當 `WHERE`、`JOIN` 或 `ORDER BY` 使用到適合的索引欄位時，資料庫的查詢規劃器會判斷是否使用索引。

索引會加快特定查詢，但不是免費的。它需要額外儲存空間，而且新增、更新、刪除資料時，資料庫也必須同步維護索引。

## 前端類比

Database index 不是前端 array index。

- Array index 是「位置 -> 資料」，例如 `users[10]`，呼叫端已經知道位置。
- Database index 比較像 `Map`，例如「email -> 資料位置」，呼叫端只需要知道查詢值。
- 沒有索引的查詢，像是在 array 裡逐筆執行 `find()`。
- 有適合索引的查詢，像先用 Map 找到位置，再取得完整物件。

## 實務用途

Index 常用在以下情境：

- `WHERE email = 'user@example.com'`：快速找出特定使用者。
- `JOIN`：加速兩張表用關聯欄位配對。
- `ORDER BY created_at`：在索引順序符合查詢時，減少額外排序成本。
- 唯一索引：除了加速查詢，也可防止 email 等欄位出現重複值。

`email` 通常辨識度高，而且常用來登入或查詢，因此適合建立 index。`gender` 只有少數幾種值，單一值可能對應大量資料，辨識度低，資料庫即使有 index 也不一定使用。

## 常見誤解

- Index 不是越多越好；每個 index 都會增加寫入與儲存成本。
- 有 index 不代表每次查詢一定會使用；資料庫會依資料量與查詢成本決定。
- 複合索引的欄位順序會影響哪些查詢能有效使用它。
- Index 不會改變 SQL 查詢結果，只會影響資料庫取得結果的方式與速度。
- 小表或會讀取大部分資料的查詢，直接掃描整張表可能反而更快。

## 10 分鐘動手

使用 SQLite 建立一張表，插入一些測試資料，再比較建立索引前後的查詢計畫：

```sql
CREATE TABLE users (
  id INTEGER PRIMARY KEY,
  email TEXT NOT NULL,
  gender TEXT
);

INSERT INTO users (email, gender) VALUES
  ('amy@example.com', 'F'),
  ('bob@example.com', 'M'),
  ('cory@example.com', 'M');

EXPLAIN QUERY PLAN
SELECT * FROM users WHERE email = 'cory@example.com';

CREATE INDEX idx_users_email ON users(email);

EXPLAIN QUERY PLAN
SELECT * FROM users WHERE email = 'cory@example.com';
```

觀察建立 index 前後，查詢計畫是否從掃描資料表改成使用 `idx_users_email`。

## 回想題

為什麼 `email` 通常適合建立 index，而 `gender` 即使建立 index，也可能無法明顯加快查詢？

