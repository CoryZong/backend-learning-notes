# Backend 筆記：RESTful API

## 一句話

RESTful API 是用「資源」來設計網址，並用 HTTP 方法與狀態碼清楚表達要做的事，讓前後端不用猜彼此的意圖。

## 我的理解

RESTful API 是一種以資源為中心的介面設計方式，透過 HTTP 方法、資源路徑與狀態碼，讓串接端能清楚理解 API 的用途與執行結果。

POST 方法本身不保證冪等，因為相同的建立請求重複送出時，可能產生多筆內容相同但 ID 不同的新資源。

常見狀態碼：

- `200 OK`：請求成功，通常會回傳內容。
- `201 Created`：成功建立新資源。
- `204 No Content`：請求成功，但不回傳內容。
- `400 Bad Request`：請求格式或語法有誤。
- `422 Unprocessable Content`：格式可以解析，但欄位內容或業務驗證未通過。
- `404 Not Found`：找不到指定的 API 路徑或資源。
- `405 Method Not Allowed`：API 路徑存在，但不支援這次使用的 HTTP 方法。

## 前端類比

可以把 API 想成前端的元件介面：元件名稱、props 與事件如果一致又可預測，使用者就不必閱讀內部實作。

例如操作使用者資源時，用 `GET /users/42` 讀取、`PATCH /users/42` 局部更新，比 `/getUser?id=42`、`/changeUserName` 這類動詞網址更容易推測。網址負責指出「哪個資源」，HTTP method 負責表達「要做什麼」。

## 實務用途

- `GET /orders/123`：讀取訂單，成功通常回 `200 OK`。
- `POST /orders`：建立訂單，成功通常回 `201 Created`，並回傳新資源或其位置。
- `PATCH /orders/123`：只更新有提供的欄位，避免把未傳欄位誤當成要清空。
- `DELETE /orders/123`：刪除訂單，成功且不回內容時可用 `204 No Content`。
- 找不到資源回 `404`，輸入不合法可回 `400` 或依契約使用 `422`；前端便能依語意顯示不同訊息。

這些慣例讓 Web、App、後台與第三方整合都能使用同一套可預測的介面，也更容易產生文件、測試與監控規則。

## 常見誤解

- RESTful 不等於「回傳 JSON」；JSON 只是常見資料格式，重點是資源、HTTP 語意與一致的介面。
- `GET` 不應產生訂單、扣款或改變業務狀態，因為瀏覽器、快取或爬蟲可能重送讀取請求。
- `PATCH` 的重點是局部更新語意，不保證一定比 `PUT` 更快；兩者如何使用要由 API 契約明確定義。
- 不是所有成功都回 `200`，也不應把所有錯誤包成 `200` 再塞一個自訂錯誤碼，否則通用工具難以判斷結果。
- RESTful 只描述介面設計，不會自動解決登入與權限；Authentication 與 Authorization 仍需另外設計。

## 10 分鐘動手

替待辦事項設計一組 API，不必真的寫後端：

1. 寫出建立、讀取單筆、局部修改與刪除待辦的 method 與 path。
2. 為每個成功情境選一個 HTTP 狀態碼。
3. 為「找不到待辦」與「title 為空」各選一個錯誤狀態碼。
4. 寫出局部完成待辦的 request body。

參考答案：

```http
POST   /tasks          -> 201 Created
GET    /tasks/42       -> 200 OK
PATCH  /tasks/42       -> 200 OK
DELETE /tasks/42       -> 204 No Content
```

```http
PATCH /tasks/42
Content-Type: application/json

{"completed": true}
```

找不到資源可回 `404 Not Found`；`title` 為空可依既定契約回 `400 Bad Request` 或 `422 Unprocessable Content`。重點不是背唯一答案，而是整個 API 採用一致且有文件的規則。

## 回想題

1. 為什麼 `GET` 不應拿來建立訂單或觸發扣款？
2. `POST /tasks` 成功建立資料時，`201` 比一律回 `200` 多表達了什麼？
3. 更新一個欄位時，為什麼通常會考慮 `PATCH`？它是否保證效能更好？
4. API 已符合 RESTful，就代表它已經處理好使用者身分與操作權限嗎？為什麼？
