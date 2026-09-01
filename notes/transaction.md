# Backend 筆記：Transaction

## 一句話

Transaction（交易）把一連串資料庫操作視為同一件事：全部成功才算完成，只要其中一步失敗，就把整組操作取消，避免資料停在做一半的狀態。

## 我的理解

Transaction 通常用於一次會修改多筆互相關聯的資料，而且不允許只成功一部分的情況。像金錢轉帳時，扣款與入帳必須一起成功；註冊時，如果需要同時建立帳號、個人資料或權限，其中一步失敗就應全部取消。

## 前端類比

它像前端送出結帳表單：建立訂單、扣除庫存與記錄付款不是三個互不相關的按鈕，而是一個完整流程。若付款失敗，畫面不能假裝訂單已完成；資料庫 transaction 讓後端也能守住這種「一起成功或一起失敗」的界線。

## 實務用途

- 轉帳：扣除 A 帳戶餘額與增加 B 帳戶餘額必須一起完成，否則金額可能憑空消失或增加。
- 電商結帳：建立訂單與扣除庫存需要一致，避免有訂單卻沒有成功保留商品。
- 批次更新：多筆相依資料修改到一半發生錯誤時，可以整批回復，避免人工清理半成品。

Transaction 的重點不是讓程式永遠不失敗，而是讓失敗發生時，資料仍維持可理解、可繼續處理的狀態。

## 常見誤解

- Transaction 不等於「所有 API 操作都能自動回復」；它通常只保護同一資料庫可控制的操作，寄信或呼叫外部付款 API 需要另外設計補償與重試。
- 呼叫 `COMMIT` 前的修改不代表其他連線一定看得到，實際可見性取決於隔離等級。
- Transaction 包得越大不一定越安全；持有鎖太久可能讓其他請求等待，甚至造成 deadlock。
- `ROLLBACK` 能取消 transaction 內的資料庫變更，但不能撤銷已經送出的外部副作用。

## 10 分鐘動手

使用 SQLite 建立帳戶資料，先完成一次成功轉帳，再刻意回復一次失敗轉帳：

```sql
CREATE TABLE accounts (
  id INTEGER PRIMARY KEY,
  balance INTEGER NOT NULL
);

INSERT INTO accounts (id, balance) VALUES (1, 1000), (2, 500);

BEGIN;
UPDATE accounts SET balance = balance - 200 WHERE id = 1;
UPDATE accounts SET balance = balance + 200 WHERE id = 2;
COMMIT;

BEGIN;
UPDATE accounts SET balance = balance - 300 WHERE id = 1;
UPDATE accounts SET balance = balance + 300 WHERE id = 2;
ROLLBACK;

SELECT * FROM accounts;
```

確認第一次轉帳後兩個帳戶總額不變，第二次因 `ROLLBACK` 而完全沒有生效。接著想想：若第二個帳戶不存在，程式應在哪一步檢查並回復？

## 回想題

1. 為什麼轉帳的扣款與入帳不能拆成兩個獨立 transaction？
2. 如果資料庫已 `COMMIT`，但寄送通知失敗，為什麼單靠 `ROLLBACK` 無法解決？
3. Transaction 開啟太久，可能如何影響同時操作相同資料的其他請求？
