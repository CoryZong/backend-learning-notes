# Backend 筆記：Cache Invalidation

## 一句話

Cache Invalidation（快取失效）是在主資料改變後，讓舊快取被刪除、更新或自然過期，避免使用者一直讀到已經不正確的資料。

## 我的理解

Cache Invalidation 是原始資料成功變更後，主動讓所有相關的舊快取失效；TTL 則是不論原始資料是否改變，時間一到就讓資料自動過期、不再視為有效。前者用來及時同步資料，後者用來限制舊資料最久能被使用多久，並作為失效機制的安全網。

## 前端類比

它很像 React Query 的 `invalidateQueries`：後端資料被修改後，前端不能繼續把舊查詢結果當成最新資料，所以要把它標成失效，下一次讀取時再重新取得。

| 前端 | 後端快取 |
|---|---|
| `queryClient.invalidateQueries()` | 刪除或標記 Redis key 失效 |
| 重新呼叫 API | 重新查詢主資料庫 |
| stale time | Redis TTL |
| 畫面短暫顯示舊資料 | API 暫時回傳舊快取 |

重點不是把主資料刪掉，而是承認「這份副本可能過時了」，讓下一次讀取回到真正的資料來源。

## 實務用途

- **商品資料更新**：修改價格或庫存後刪除 `product:42`，下一個請求會查資料庫並建立新快取，避免繼續顯示舊價格
- **文章與個人資料**：編輯完成後讓詳細頁快取失效，因為使用者通常預期剛儲存的內容立刻可見
- **列表與統計資料**：新增訂單時，不只訂單明細會改變，訂單列表與今日營收也可能要失效，因為同一筆寫入可能影響多個快取 key
- **多服務系統**：由更新資料的服務發出事件，通知其他服務清除相關快取，因為它們可能各自保存同一份資料的副本

常見的 Cache-Aside 流程是：讀取時先查快取，沒有才查資料庫並回填；寫入時先成功更新資料庫，再刪除舊快取。這樣主資料庫仍是唯一依據，下一次讀取才會產生新快取。

## 常見誤解

- **「設定 TTL 就不用管失效」**：TTL 只是最後的安全網；若價格設成一小時後才過期，使用者仍可能看一小時的錯誤價格
- **「資料更新後再更新快取就一定正確」**：同時有多個請求時，更新順序可能交錯，較舊的值反而最後寫入；直接刪除通常比較容易維持一致性
- **「刪掉一個明細 key 就完成了」**：同一筆商品也可能出現在搜尋結果、分類列表與統計資料中，必須先知道哪些快取依賴它
- **「資料不一致代表 Redis 壞了」**：Redis 可能只是忠實回傳先前存入的值；真正漏掉的通常是寫入資料後的失效規則
- **「最安全就是每次清空全部快取」**：這會讓大量請求同時打回資料庫，造成 cache stampede；應盡量只失效受影響的 key

## 10 分鐘動手

建立 `cache-demo.js`，用兩個 `Map` 模擬資料庫與快取：

```javascript
const db = new Map([
  ['product:42', { name: '鍵盤', stock: 5 }],
]);
const cache = new Map();

function getProduct(key) {
  if (cache.has(key)) {
    console.log('CACHE HIT');
    return cache.get(key);
  }

  console.log('CACHE MISS');
  const value = { ...db.get(key) };
  cache.set(key, value);
  return value;
}

function updateProduct(key, patch, invalidate = true) {
  db.set(key, { ...db.get(key), ...patch });
  if (invalidate) cache.delete(key);
}

console.log(getProduct('product:42')); // MISS，stock: 5
console.log(getProduct('product:42')); // HIT，stock: 5

updateProduct('product:42', { stock: 4 }, false);
console.log(getProduct('product:42')); // HIT，仍是錯的 stock: 5

cache.delete('product:42');
console.log(getProduct('product:42')); // MISS，取得正確的 stock: 4
```

執行：

```bash
node cache-demo.js
```

接著把 `false` 移除再執行一次。你會看到更新資料庫後刪除快取，下一次讀取會出現 `CACHE MISS` 並取得新庫存；這就是最基本的 Cache-Aside invalidation。

## 回想題

1. 為什麼資料庫更新成功後，舊快取不能繼續被當成正確資料？
2. TTL 已設為一小時，商品價格卻必須立即生效時，還需要做什麼？
3. 新增一筆訂單後，除了訂單明細，還可能有哪些相關快取需要失效？
4. 為什麼「精準刪除受影響的 key」通常比「清空全部快取」更安全？
