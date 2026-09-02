# Backend 筆記：ACID

## 一句話

Transaction 決定「哪些操作要綁成一組」，ACID 則描述資料庫執行這組操作時應提供的四種保證：不能只完成一半、結果必須合法、同時操作不能互相破壞，以及提交後不能無故消失。

## 我的理解

Transaction 將多個資料庫操作包成同一組，而 Atomicity 保證這組操作即使失敗，也不會留下只完成一部分的結果。ACID 除了 Atomicity，還包含 Consistency，確保操作前後符合資料規則；Isolation，避免同時執行的 transaction 讀取或干擾尚未完成的狀態；以及 Durability，確保資料在 `COMMIT` 後，即使系統重啟也不會無故遺失。

## 前端類比

可以把 Transaction 想成前端的一次「送出訂單」：建立訂單、扣除庫存與寫入付款紀錄被包成同一組操作。ACID 則是這次送出必須符合的四項驗收條件：

- Atomicity（原子性）：三個操作必須全部成功或全部取消，不能只建立訂單卻沒有扣庫存。
- Consistency（一致性）：完成前後都要符合資料規則，例如庫存不能是負數、訂單必須屬於存在的使用者。
- Isolation（隔離性）：兩個人同時搶最後一件商品時，不能都讀到「還有 1 件」並各自成功結帳。
- Durability（持久性）：後端回覆結帳成功後，即使服務重啟，已提交的訂單仍然存在。

因此，上一篇 Transaction 主要回答「為什麼要把多個操作包在一起，以及如何 `COMMIT` 或 `ROLLBACK`」；這篇 ACID 則回答「包起來之後，資料庫還需要守住哪些性質」。兩篇會在 Atomicity 上重疊，但 ACID 還包含 Consistency、Isolation 與 Durability。

## 實務用途

- Atomicity 用在轉帳：扣款成功但入帳失敗時要整組取消，否則錢會憑空消失。
- Consistency 用在資料規則：`CHECK`、`UNIQUE`、外鍵及應用程式驗證共同避免負數庫存、重複帳號或不存在的關聯。
- Isolation 用在並發操作：多人同時購買、搶票或更新同一筆資料時，需要避免超賣與互相覆蓋。
- Durability 用在故障復原：資料庫一旦回覆提交成功，就要透過交易日誌與持久化儲存保留結果。

`CHECK (balance >= 0)` 只是在示範 Consistency 的其中一種工具，不是 ACID 的全部。ACID 的重點是從四個角度檢查一筆 transaction 是否可靠。

## 常見誤解

- ACID 不等於 Transaction；Transaction 是操作邊界，ACID 是這個邊界內應具備的保證。
- `CHECK` 只負責阻擋特定非法資料，不能提供 Atomicity、Isolation 或 Durability。
- Atomicity 與上一篇 Transaction 高度相關，但不能因此把 ACID 簡化成 `COMMIT` 與 `ROLLBACK`。
- Consistency 不是指多台資料庫副本會立刻得到相同內容；ACID 的一致性重點是 transaction 前後都符合資料庫規則與業務約束。
- Isolation 不一定代表所有 transaction 完全依序執行；不同隔離等級會在正確性、並發量與等待時間之間取捨。
- ACID 通常只保護資料庫內的操作；已寄出的 email 或已呼叫的第三方付款不會跟著資料庫 `ROLLBACK`。

## 10 分鐘動手

先在終端機執行 `sqlite3 acid-demo.db`，開啟一個檔案型資料庫並建立帳戶：

```sql
CREATE TABLE accounts (
  id INTEGER PRIMARY KEY,
  balance INTEGER NOT NULL CHECK (balance >= 0)
);

INSERT INTO accounts (id, balance) VALUES (1, 100), (2, 50);
```

接著分別觀察四個特性。

### A：Atomicity

先修改兩個帳戶，再取消整組 transaction：

```sql
BEGIN;
UPDATE accounts SET balance = balance - 40 WHERE id = 1;
UPDATE accounts SET balance = balance + 40 WHERE id = 2;
ROLLBACK;

SELECT * FROM accounts;
```

結果仍是 `100` 與 `50`，因為 `ROLLBACK` 取消了整組操作，而不是只取消其中一行。

### C：Consistency

嘗試直接產生不合法的餘額：

```sql
UPDATE accounts SET balance = -1 WHERE id = 1;
```

這行會因 `CHECK (balance >= 0)` 失敗，表示資料庫拒絕違反規則的結果。`CHECK` 在這裡只示範 Consistency。

### I：Isolation

保留目前的終端機 A，再到終端機 B 執行 `sqlite3 acid-demo.db`，讓兩邊開啟同一個資料庫。在終端機 A 執行下列操作，但先不要提交：

```sql
BEGIN;
UPDATE accounts SET balance = balance - 10 WHERE id = 1;
```

接著在終端機 B 查詢：

```sql
SELECT * FROM accounts;
```

終端機 B 不會把 A 尚未提交的餘額 `90` 當成正式結果。最後在終端機 A 執行 `ROLLBACK`。

### D：Durability

完成一次真正的轉帳並提交：

```sql
BEGIN;
UPDATE accounts SET balance = balance - 40 WHERE id = 1;
UPDATE accounts SET balance = balance + 40 WHERE id = 2;
COMMIT;
```

關閉並重新開啟 SQLite，再執行 `SELECT * FROM accounts;`，結果仍應是 `60` 與 `90`。這是在觀察已提交資料的 Durability。

## 回想題

1. Transaction 與 ACID 分別回答什麼問題？
2. `CHECK (balance >= 0)` 只是在示範 ACID 的哪一項特性？
3. 兩個人同時購買最後一件商品時，為什麼只有 Atomicity 還不夠？
4. 資料庫回覆 `COMMIT` 成功後重新啟動，資料仍應存在，這是哪一項特性？
