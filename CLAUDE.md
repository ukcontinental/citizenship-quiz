# 入籍陷阱題特訓 · 專案紀錄

給之後接手的 Claude（或人）看的：這個專案是什麼、資料放在哪、改東西要注意什麼。

## 這是什麼

加拿大公民入籍考試練習網站，給 Willie 一家人用（本人、太太、兒子）。
- 網址：https://ukcontinental.github.io/citizenship-quiz/
- 整個網站只有一個檔案 `index.html`，GitHub Pages 從 `main` 分支根目錄發布，push 後 1～2 分鐘上線。
- 內容：三套陷阱題（每套涵蓋 Discover Canada 全部 10 章）、模擬考（20 題／45 分／15 題及格）、逐題練習、錯題本（重練、匯出 PDF）、速讀時間軸、陷阱字典。
- 題庫在 `index.html` 裡的 `Q1`、`Q2` 陣列。題目 id（A01、B02…）是**依章節輪流分配到三套後自動編號**的，新增或刪除題目會讓後面的 id 位移，已存的錯題會對不上。要改題目時，盡量只改字、不改順序；要加題就加在各章最後並告知使用者。

## 登入與錯題同步（不需要任何帳號）

- 每個人輸入「名字＋4 位數密碼」。`bookId = SHA-256("citz-v1|" + 名字小寫去空白 + "|" + 密碼)`（64 位 hex）。
- 登入資訊存在 localStorage `citz-me` = `{name, id}`；本機快取 `citz-wrong:<id>`，未上傳標記 `citz-pending:<id>`（本機未上傳的修改優先於雲端）。
- 雲端：Firebase 專案 **citizenship-quiz-27223**（Willie 的 Google 帳號、免費 Spark 方案），Firestore `(default)`，位置 northamerica-northeast1（蒙特婁）。
- 文件路徑 `books/<bookId>`，欄位只有 `items`（JSON 字串，`{題目id: {n 錯幾次, t 時間, pick 選了什麼}}`）和 `updated`。
- 用 Firestore REST API 直接讀寫（`?key=` 帶 web apiKey，金鑰本來就公開在網頁裡，不是秘密）。重新整理、切回分頁、恢復網路、每 60 秒會再拉一次。
- Firestore 安全規則（在 Firebase 主控台 → Firestore → 規則）：
  ```
  match /books/{bookId} {
    allow get: if bookId.size() == 64;
    allow create, update: if bookId.size() == 64
      && request.resource.data.keys().hasOnly(['items', 'updated']);
    allow list, delete: if false;
  }
  match /{document=**} { allow read, write: if false; }
  ```
  → 不能列出別人的錯題本、不能刪除、不能塞其他欄位。也因此**用 API 刪不掉測試資料**，要到 Firebase 主控台手動刪。主控台裡名字像「測試T…」「爸爸T…」的空錯題本是 2026-10-06 測試留下的，無害。
- 匯入連結：`#import=<base64url(JSON items)>`，登入後合併（同一題取錯得多的那筆），用完會清掉網址上的 hash。2026-10-06 用這個把舊 Claude 頁面的 9 題錯題交給 Willie。

## 相關網站與資料

- **canada-citizenship-prep**（讀課文、朗讀、每章重點、另一套題庫）：https://ukcontinental.github.io/canada-citizenship-prep/ ，repo `ukcontinental/canada-citizenship-prep`。和這個網站**同一個網域**，所以共用 localStorage 的 `citz-me`，登入一次兩邊都生效。它的錯題本存在同一個 Firestore 的 `books/<SHA-256(personId + "|prep")>`，和這裡是分開的兩本（題目 id 不同）。兩站互相有連結。
- 舊版：Claude artifact「入籍陷阱題特訓」https://claude.ai/artifact/FusLZKJ4fGheNJ7T1kGhXy （需要 Claude 帳號才能存錯題，已由這個網站取代，保留未刪）。

## 測試方式

- 語法：把 `<script>` 內容抽出來 `node --check`。
- 實際測：用 Playwright（`executablePath: '/opt/pw-browsers/chromium'`，雲端環境要加 `proxy` 與 `--ignore-certificate-errors`；本機 http server 要 `bypass: '<-loopback>'`）開兩個 browser context 當兩台裝置，同名同密碼看錯題是否同步，不同人是否分開。測試會在 Firestore 留下資料，名字加上 `T<時間>` 方便辨認。

## 進度紀錄

- 2026-10-05 確認舊 Claude 頁面的錯題本是全家共用一本；先改成依 Claude 帳號分開（需要每人有 Claude 帳號並被設為 Editor）。
- 2026-10-06 家人沒有 Claude 帳號 → 改成這個獨立網站：建立 Firebase 專案與規則、建立此 repo、開 GitHub Pages、匯入 Willie 舊的 9 題錯題、加上到 canada-citizenship-prep 的連結。
- 同日在 canada-citizenship-prep 加上同一套登入與錯題雲端同步，並修好人物時間軸等頁面上方 CSS 被當文字顯示的問題。
