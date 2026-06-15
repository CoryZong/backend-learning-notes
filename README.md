# Backend Learning Notes

這個 repository 用來整理我學習後端、資料庫、系統設計、Linux、部署與架構時的筆記。

我的目標不是把內容寫成完整教科書，而是把每個真正理解過的小知識，整理成之後可以快速回想、複習、延伸的知識卡片。

---

## 為什麼建立這個筆記

我目前主要背景是前端工程，正在逐步補強後端與系統設計能力。

在學習後端知識時，我發現單純看文章或影片很容易「當下懂，過幾天又忘」。
所以這個 repository 的重點是：

* 用自己的話整理理解
* 用前端熟悉的概念做類比
* 補上實務情境
* 記錄容易誤解的地方
* 留下回想題，方便之後複習

---

## 筆記原則

每篇筆記盡量包含以下結構：

```md
# Backend 筆記：主題名稱

## 一句話

用一到兩句話說明這個概念解決什麼問題。

## 我的理解

用自己的話寫下真正理解的版本，不直接抄定義。

## 前端類比

用前端工程師熟悉的概念類比，例如 array、Map、cache、component、API call、state management 等。

## 實務用途

說明這個概念通常在哪些後端或系統場景會用到。

## 常見誤解

整理容易搞錯的地方。

## 10 分鐘動手

設計一個小練習，讓自己不是只看過，而是真的摸過一次。

## 回想題

留一題之後可以不用看筆記回答的問題。
```

---

## 目錄

### Database

* [Database Index](notes/database-index.md)
* SQL vs NoSQL
* Transaction
* ACID
* Normalization
* Query Optimization

### Cache

* Redis
* Cache Invalidation
* TTL
* Session Store

### API Design

* RESTful API
* Authentication
* Authorization
* Rate Limit
* Pagination

### Linux / Deployment

* Linux Basic Commands
* Docker
* Nginx
* Caddy
* CI/CD

### System Design

* Load Balancer
* Queue
* Message Broker
* Logging
* Monitoring
* Scalability

---

## 我的學習流程

每次學完一個小知識，先用自己的話寫下：

```md
## 我理解的是

## 我還卡住的是

## 我想到的前端類比
```

接著再整理成正式筆記。

這樣可以避免只是在看別人的說明，而是讓知識真的經過自己的理解。

---

## Commit 命名習慣

建議使用簡單明確的 commit message：

```txt
Add database index note
Add SQL vs NoSQL note
Update README structure
Refine Redis cache note
```

原則是讓自己之後看 git log 時，能快速知道每次更新了什麼。

---

## 目標

這個 repository 最終希望能變成我的後端知識地圖。

不是為了追求一次學完，而是持續累積：

```txt
小知識
→ 自己理解
→ 實作練習
→ 筆記整理
→ 回想複習
→ 形成系統化知識
```

重點是長期穩定累積，而不是短時間大量塞知識。
