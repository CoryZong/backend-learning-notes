# Backend 筆記：環境變數（Environment Variables）

## 一句話

環境變數是「從程式外面餵進來的設定值」，讓同一份程式碼在本機、測試、正式環境跑出不同行為，並把密碼、API key 這類機密跟程式碼分開，不寫死在原始碼裡。

## 我的理解

後端跟前端一樣都有環境變數，而且都不能推上 Git。

兩個重點小知識：

1. ENV 裡的值永遠是字串。所以 `"false"` 拿來做布林判斷時會被當成 `true`（非空字串都是 truthy），要自己手動轉型。
2. 在 shell 視窗裡用 `export` 設定的環境變數，關掉視窗就失效；如果寫進 shell 的啟動設定檔（例如 `~/.bashrc`），則每次開終端機都會自動存在。

## 前端類比

你在 Vite / Next.js 裡早就用過它了：

| 前端 | 後端 |
|---|---|
| `.env` 檔放 `VITE_API_URL=...` | `.env` 檔放 `DATABASE_URL=...` |
| `import.meta.env.VITE_API_URL` | `process.env.DATABASE_URL`（Node）/ `os.environ["DATABASE_URL"]`（Python） |
| `.env` 不進 git（在 `.gitignore`） | 一樣，`.env` 永遠不進 git |
| 不同環境用 `.env.development` / `.env.production` | 不同環境注入不同變數 |

關鍵差異（也是最大的坑）：前端的環境變數最後會被打包進 bundle、送到瀏覽器，所以 `VITE_` / `NEXT_PUBLIC_` 開頭的東西其實是公開的，使用者打開 DevTools 就看得到。後端的環境變數則留在伺服器上、永不外送，所以後端才是放真正機密（DB 密碼、私鑰）的地方。把密鑰放到 `VITE_SECRET` 等於貼在公布欄上。

## 實務用途

- **分環境設定**：本機連 `localhost:5432`，正式連雲端 DB，靠的就是 `DATABASE_URL` 這顆變數不同值。
- **存放機密**：DB 密碼、第三方 API key（Stripe、OpenAI）、JWT 簽章密鑰——全部走環境變數，絕不進原始碼。
- **開關功能（feature flag）**：`ENABLE_NEW_CHECKOUT=true`，不改程式就能在正式環境開關功能。
- **CI/CD 與容器**：Docker `docker run -e KEY=value`、Kubernetes Secret、GitHub Actions 的 Secrets，本質都是在「注入環境變數」。

## 常見誤解

- **「`.env` 檔是被加密的、很安全」**：完全沒有。`.env` 只是純文字檔，安全性 100% 來自「它沒被 commit、沒被外人讀到」。它的角色只是方便本機注入，不是保險箱。
- **「環境變數設了就馬上生效」**：環境變數是程序「啟動那一刻」讀進記憶體的快照。你改了 `.env` 或雲端面板的值，已經在跑的程式不會自動更新，必須重啟程序才會吃到新值。
- **「前端後端的環境變數是同一回事」**：前端的會進 bundle、是公開的；後端的留在伺服器、是私密的。把機密放前端變數是最常見的安全事故。
- **「環境變數有型別」**：它永遠是字串。`DEBUG=false` 拿到的是字串 `"false"`，在 JS 裡 `if (process.env.DEBUG)` 會判斷成 `true`——這是經典 bug，要自己手動轉型。

## 10 分鐘動手

打開 Git Bash，依序體會「注入 → 讀取 → 同一份程式不同值 → 不同行為」。

`export` 是 shell 指令，意思是「在這個終端機裡設一個變數，並讓接下來啟動的程式也讀得到」。

```bash
# 1) 在這個 shell 裡注入一個環境變數
export MY_GREETING="hello from env"

# 2) 讀它
echo $MY_GREETING

# 3) 讓一個「程式」去讀它（程式碼沒寫死值，是去外面拿）
node -e "console.log(process.env.MY_GREETING)"
# 沒裝 Node 的話，純 shell 也能體會：echo $MY_GREETING
```

接著體會「同一份程式、不同環境變數 → 不同行為」。下面兩行的 `node -e "..."` 完全一樣，只有前面餵的值不同：

```bash
APP_ENV=production  node -e "console.log('現在環境是:', process.env.APP_ENV)"
APP_ENV=development node -e "console.log('現在環境是:', process.env.APP_ENV)"
```

`KEY=value 指令` 這個寫法 = 只在這一次執行時臨時塞一個環境變數給它。你會看到輸出分別是 production / development——程式碼一個字都沒改，只因餵的值不同，行為就變了。這就是環境變數存在的全部意義。

最後體會「字串型別陷阱」：

```bash
DEBUG=false node -e "console.log('型別:', typeof process.env.DEBUG, '| if 判斷:', process.env.DEBUG ? '進入(被當true)' : '跳過')"
```

`DEBUG=false` 竟然走進了 true 分支——因為它是非空字串。

## 回想題

你的後端用 `DATABASE_PASSWORD` 環境變數連資料庫，某天密碼外洩，你緊急在雲端平台把 `DATABASE_PASSWORD` 改成新密碼。為什麼光改這個值還不夠、服務可能還在用舊密碼連線？要讓新密碼生效，你還得做什麼動作？這跟哪一個「常見誤解」是同一件事？
