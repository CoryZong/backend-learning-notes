# Backend 筆記：API 分頁（offset vs cursor）

## 一句話

當資料有成千上萬筆時，API 不會一次全部回給你，而是「一頁一頁」回——而怎麼決定「下一頁從哪開始」，有兩種主流做法：**offset（用第幾筆）** 和 **cursor（用一個指標）**。

## 我的理解

API 分頁在後端有分兩種做法：

- **offset**：主要用於列表頁換頁時，原理是根據排序去做查詢。有個小缺點是資料如果突然被新增，可能會整個往後移而出現錯位，但在一般列表頁情境通常無傷大雅。
- **cursor**：主要用於無限滾動換頁時，原理是會記住上次查詢最後一筆資料的識別指標，下一次查詢就能快速找到對應位置、繼續提供下一頁資料，而且不受中途新增資料影響。

## 前端類比

- **offset 分頁** ≈ `array.slice(offset, offset + limit)`。你按「位置」切。問題是只要陣列前面被 `unshift()` 插入一筆，你後面切出來的內容就全錯位了——這正是 offset 分頁會重複/漏資料的原因。
- **cursor 分頁** ≈ 你記住「上次處理到的那個物件的 id」，下次從那個 id 之後繼續，而不是記「第幾格」。物件 id 不會因為別人插隊而改變，所以不會錯位。
- 你在前端寫的**無限滾動（infinite scroll）**，後端幾乎都是 cursor 分頁；你寫的**頁碼導覽列（pagination bar，1 2 3 4）**，後端通常是 offset 分頁。

## 實務用途

- **無限滾動的動態牆**（FB、IG、Twitter）：一律 cursor，因為資料一直在最上面新增，offset 會錯位到不能用。
- **後台管理表格**（要顯示「共 3,200 筆、第 17 頁」、要能跳頁）：用 offset，因為使用者需要總頁數和跳頁，且資料量通常沒大到效能爆掉。
- **資料匯出 / 批次同步**（把整張表掃一遍）：用 cursor（常稱 keyset pagination），因為要掃幾百萬筆，offset 到後面會慢到不可接受。
- **公開 API**（GitHub、Stripe、Slack）：幾乎清一色 cursor，並把 cursor 包成一段不透明字串放在回應裡，你只要照著帶下一個 cursor 就好。

## 常見誤解

- **「分頁就是 limit/offset，cursor 是進階花招」**：剛好相反，大流量場景 cursor 才是預設解，offset 反而是會出事的那個。
- **「cursor 一定是時間戳或 id」**：cursor 對使用者來說應該是**不透明（opaque）**的——後端可以把 `(created_at, id)` 編碼成一段字串，你不該去解析它、也不該自己組。後端改排序邏輯時才不會破壞你的程式。
- **「offset 分頁的 total count 很便宜」**：要顯示「共幾頁」就得 `COUNT(*)`，大表上這個 count 本身可能比查資料還慢，常常要另外快取。
- **「cursor 分頁不需要排序」**：cursor 分頁**強依賴一個穩定且唯一的排序鍵**（通常是 `id`，或 `created_at + id` 複合）。如果排序欄位有重複值又沒有 tie-breaker，會漏資料。

## 10 分鐘動手

GitHub 的公開 API 用的就是 cursor 分頁（透過 `Link` header），不用註冊就能打。打開 **Git Bash**，貼這行：

```bash
# 拿最新 2 筆 commit
curl -s "https://api.github.com/repos/torvalds/linux/commits?per_page=2"
```

關鍵觀察：你只說了「一頁幾筆（`per_page`）」，**從沒說「要第幾筆」**。再打一次一模一樣的指令，會發現它給你的是**同樣的 2 筆**——因為 HTTP 是無狀態的，伺服器不會記得你看過哪裡。

接著看伺服器怎麼告訴你「下一頁從哪走」：

```bash
# 加 -D - 把 response header 印出來，找 link: 那行
curl -s -D - "https://api.github.com/repos/torvalds/linux/commits?per_page=2" -o /dev/null
```

在 header 裡找開頭是 `link:` 的那行，裡面 `rel="next"` 前面那串網址會帶著一個 `after=xxx`——那就是 cursor。把那串網址原封不動再打一次，就會拿到下一頁。體會：**你全程沒用到「第幾筆」這種數字，只是照著伺服器給的指標往下帶。**

> Windows 小提醒：PowerShell 的 `curl` 是別名、行為不同，請用 Git Bash，或在 PowerShell 改打 `curl.exe`。

## 回想題

你在做一個聊天室，訊息一直往最新的方向長，使用者往上滑載入更舊的訊息。如果你用 `offset` 分頁，當有人在你往上滑的同時送出一則新訊息，會發生什麼狀況？換成 cursor 又為什麼能避免？
