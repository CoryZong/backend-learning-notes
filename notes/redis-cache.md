# Backend 筆記：Redis 快取

## 一句話

Redis 是一個跑在記憶體裡的 key-value 資料庫，讓你把「常讀但少改」的資料暫存起來，讓後端不必每次都去查速度慢的主資料庫。

## 我的理解

Redis 是專給伺服器使用的快取服務，不是伺服器內建功能，需要額外啟動服務才能使用。

快取資料是所有使用者共用的，key 的命名方式決定了快取是共享還是私有。例如電商平台在搜尋商品後，可以把結果存進 Redis 並設定 60 秒 TTL：第一個使用者搜尋「iPhone」觸發資料庫查詢並寫入快取，後續 60 秒內搜尋相同關鍵字的使用者都直接從 Redis 拿資料，資料庫完全不需要被打到。

## 前端類比

Redis 就像伺服器端的 localStorage——前端用 localStorage 快取 API 回應，避免重複請求；Redis 在伺服器端做同樣的事，但是所有使用者共用同一份快取。

| 前端 | 後端（Redis） |
|---|---|
| `localStorage.setItem(key, val)` | `SET key val` |
| `localStorage.getItem(key)` | `GET key` |
| sessionStorage（關分頁消失） | `EXPIRE key 3600`（TTL 到期刪除） |
| `localStorage.removeItem(key)` | `DEL key` |

差別在於：localStorage 只有那個使用者的那個瀏覽器看得到；Redis 是共享的，所有使用者的所有請求都共用同一個快取。

## 實務用途

- **API 回應快取**：商品頁不會每秒都改，把 JSON 存進 Redis，TTL 設 60 秒，同一分鐘的請求都打快取
- **Session 存放**：把使用者 session 放進 Redis，水平擴充多台伺服器時各台都能讀同一份 session
- **Rate Limiting**：每個 IP 在 60 秒內最多打 100 次，用 `INCR` 加計數、`EXPIRE` 設重置時間
- **排行榜**：Sorted Set 天生就是排行榜結構，`ZADD` 加分、`ZRANK` 查名次

## 常見誤解

- **「Redis 可以當主資料庫用」**：預設重開機資料就消失，主資料還是得放在磁碟型資料庫
- **「快取越多越好」**：改了資料庫但 Redis 還是舊的，叫做 cache invalidation，是後端最頭痛的問題之一
- **「SET 了就永遠在」**：不設 TTL 的 key 永遠不消失，記憶體爆掉才會被自動驅逐，生產環境幾乎都應該設 TTL
- **「Redis 可以下 SQL 查詢」**：Redis 是 key-value，沒有 JOIN，只能用精確的 key 取資料

## 10 分鐘動手

```bash
docker run --name myredis -p 6379:6379 -d redis
docker exec -it myredis redis-cli
```

```bash
SET greeting "hello backend"
GET greeting

SET counter 0
INCR counter
INCR counter
INCR counter
GET counter

SET session:abc123 "user_id=42" EX 30
TTL session:abc123
# 等 30 秒後再執行：
GET session:abc123
```

觀察：`INCR` 是原子操作，不會有 race condition；TTL 到期後 `GET` 回傳 `(nil)`，key 自動消失。

## 回想題

你的電商平台快取了「iPhone 搜尋結果」，但商品剛上架了一個新 iPhone。這 60 秒內使用者看到的會是舊資料——你會怎麼設計來處理這個問題？
