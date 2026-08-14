# fruit-ninja-cam — 體感切水果

單檔靜態網頁遊戲：`index.html` 即完整專案，無建置步驟、無 package.json。

**GitHub Pages 上線網址**：<https://m255525.github.io/fruit-ninja-cam/>（2026-08-05 啟用，`master` 分支根目錄）。公開 repo：<https://github.com/M255525/fruit-ninja-cam>，含 `README.md`。因為主檔本來就叫 `index.html`，不像 `Rummikub` 需要額外的重新導向 stub，`git push` 後 GitHub Pages 會自動重新部署（通常一兩分鐘內生效）。

## 加入主畫面（PWA，2026-08-14 新增）

比照工作區 `expense-tracker-pwa` 的做法：`manifest.json`＋`icons/`（深色 `#0b1020` 背景、橘紅 `#FF6B4A`「切」字圖示）＋`service-worker.js`（network-first＋同源快取備援，跨網域的 MediaPipe CDN／模型檔請求一律略過不進快取，不需要每次改動升版 `CACHE_NAME`）。安裝按鈕（`#installBtn`）放在 `#start-screen`「改用滑鼠／觸控玩」下方，跟既有 `.btn secondary` 同款樣式；本專案沒有 `showToast`，安裝失敗走「暫時置換按鈕文字」的簡易 fallback。**測試時務必只被動檢查 DOM／SW 狀態，不要點擊任何按鈕**（見下方「⚠️ 用真實 Chrome 測試」一節，即使只是測 PWA 功能也要避免誤觸開始遊戲/攝影機）。已用 Playwright 實測 Chromium 觸發 `beforeinstallprompt`、SW 註冊成功，過程全程未點擊任何按鈕。


**iOS／iPadOS／macOS 相容性補強（2026-08-14 同日追加）**：Safari（含 iOS 上的 Chrome/Firefox，底層都是 WebKit）**永遠不會觸發 `beforeinstallprompt`**，原本的按鈕邏輯在這些瀏覽器上一律落入「瀏覽器不支援」這句話，其實是誤導——蘋果裝置本來就能加入主畫面，只是要透過分享選單手動操作，不像 Chrome/Edge 有自動彈窗。修法：安裝腳本新增 `isIOSDevice`（`/iPad|iPhone|iPod/` 或 `navigator.platform==='MacIntel' && maxTouchPoints>1`——後者是因為 iPadOS 13+ 預設偽裝成 Mac 桌面版 UA，要用觸控點數才分得出來是 iPad 還是真的 Mac）與 `isMacDesktop && isSafariEngine`（macOS 桌面版 Safari 17+ 是「檔案→加入 Dock」，跟手機的分享選單操作不同）兩種判斷，各自顯示對應的操作指引文字，不再顯示「不支援」；`isStandalone`（`matchMedia('(display-mode: standalone)')` 或 iOS 專有的 `navigator.standalone`）為真時直接隱藏按鈕（已經是安裝後開啟，不需要再顯示安裝按鈕）。`<head>` 同步補上 `apple-touch-icon`（180×180 專用尺寸，`icons/apple-touch-icon.png`，純色不透明背景）＋ `apple-mobile-web-app-capable`／`mobile-web-app-capable`（兩個都要，前者給 Safari、後者是 Chrome 主張的新標準，只寫一個 Chrome 會在主控台噴 deprecation warning）＋ `apple-mobile-web-app-status-bar-style`／`apple-mobile-web-app-title`。這五個判斷/訊息字串在全部 9 個已加裝 PWA 的專案裡是逐字複製的同一段邏輯，日後若要調整任一處的措辭或判斷式，建議九個一起改，避免各專案的安裝體驗不一致。

**回饋機制與快取踩坑修正（2026-08-14，使用者實測回報「加入主畫面沒有功能」才發現兩層問題）**：(1) 原本無 `showToast` 時用「暫時置換按鈕文字」當提示，在工具列裡太不明顯，使用者完全沒注意到訊息出現過——改成 `window.alert(fallbackMessage())`，`deferredPrompt.prompt()` 也包 try/catch。(2) 改完使用者仍回報沒反應，追查發現 `service-worker.js` 的 `fetch(event.request)` 沒有繞過瀏覽器 HTTP 快取——GitHub Pages 對回應下 `Cache-Control: max-age=600`，10 分鐘內「network-first」名不符實，可能吃到舊版內容重新存進 Cache Storage。改成 `fetch(event.request, {cache:'reload'})` 強制略過 HTTP 快取，`CACHE_NAME` 同步升版 v1→v2 清掉已污染的快取。這是跟 `expense-tracker-pwa` 那次「install 階段 `cache.addAll()` 忘記加 `{cache:'reload'}`」同一個 bug class 的 runtime 版本，細節見 [[pwa-install-rollout]]。

## 架構

- **手部追蹤**：`@mediapipe/tasks-vision` HandLandmarker，經 jsdelivr CDN 以動態 `import()` 載入（版本 0.10.14），模型檔從 Google 官方 storage 載入。GPU delegate 失敗自動退回 CPU。**首次啟動需要網路**下載 wasm 與模型（約 10MB），之後靠瀏覽器快取。
- **刀鋒**：食指指尖（landmark 8），最多雙手。攝影機畫面以 `object-fit: cover` + `scaleX(-1)` 鏡像滿版，`lmToScreen()` 負責 landmark → 畫面座標換算（含 cover 裁切與鏡像）。
- **切割判定**：指尖軌跡最新線段 vs 水果圓形（`segCircle`），且移動速度需超過 `SLICE_SPEED`（700 px/s）——手停在水果上不算切。
- **降級**：攝影機被拒／無鏡頭／CDN 載入失敗 → 自動退回滑鼠/觸控模式；開始畫面也有「改用滑鼠玩」按鈕。
- **音效**：全部 WebAudio 合成，零外部音檔。
- **鏡頭選擇**：取得權限後以 `enumerateDevices()` 列出所有 videoinput，開始畫面有下拉選單（≥2 顆才顯示）。首次無記憶時自動優先挑 label 含 USB/UVC/webcam 且非 built-in 的外接鏡頭；選擇記在 localStorage `fnc-cam`，指定鏡頭開啟失敗（已拔除）自動退回預設並清除記憶。結算畫面有「回主選單」可換鏡頭。
- **三關制**：`LEVELS` 陣列定義每關的難度參數（第 1 關 60 秒、第 2/3 關 45 秒）（生成間隔、每波顆數 2-3／3-5／4-6、炸彈機率 5%／12%／18%）。過關顯示橫幅並回復 1 ❤️（共 5 顆），撐完三關出現勝利畫面。
- **每關保底 50 顆水果**（`MIN_FRUITS_PER_LEVEL`，不含炸彈）：生成時依剩餘額度動態縮短間隔配速；時間到但額度未滿則延長該關直到拋滿才過關。實測三關各拋 50／75／124 顆。
- **最高分**：localStorage key `fnc-best`。
- **排行榜（2026-08-05 新增）**：localStorage key `fnc-leaderboard`，本機前 5 名（`{name, score, mode, date}`）。結算畫面（`gameOver()`）判斷本局分數是否擠得進前 5（`lbQualifies()`：不到 5 筆或分數高於目前第 5 名），擠得進才顯示暱稱輸入框（`#score-entry`，留白預設「訪客」，`maxlength=10`），按「🏆 記錄成績」才寫入並依分數重新排序、截斷到 5 筆。列表每筆額外標示當局用的是 📷 攝影機模式還是 🖱️ 滑鼠／觸控模式（`e.mode`）。輸入框故意不用 `prompt()` 原生對話框（會擋住整個頁面且無法客製樣式），改用畫面內的 inline input。

## 手機版面修正（2026-08-05）

實測手機寬度（390px）發現 HUD 頂部的 `#level-info`（第 X 關＋倒數，`left:50%` 水平置中）跟 `#lives`（愛心，`right:18px` 靠右對齊）會疊在一起——兩者都是各自 `position:absolute` 假設螢幕夠寬才不會碰撞，實測窄於約 624px 就會重疊。**修法**：新增 `@media (max-width:700px)` 規則，把 `#level-info` 往下移到第二行（`top:54px`，字體縮到 17px），跟分數／愛心那一列分開，不論愛心格數多少都不會再碰在一起（已用 `getBoundingClientRect()` 實測前後座標確認重疊消失）。這是純 CSS 版面修正，跟手部追蹤／切割判定邏輯無關。

## 頂部跑馬燈與「關於」資訊（2026-08-05 新增）

`#marqueeBar` 顯示跟工作區其他工具共用同一份 Google Sheet 維護的公告內容，同一個授權伺服器 Apps Script 網址。這個遊戲**沒有序號登入機制**，做法是頁面載入時直接 POST 空序號給該網址，只取回傳的 `marquee` 欄位。`localStorage` key 為 `fncMarquee`。

**版面整合方式跟其他工具不同**：這個遊戲的 `#app` 是 `position:fixed;inset:0` 撐滿整個視窗，底下所有 HUD／攝影機／canvas 元素都是相對 `#app` 絕對定位（不是 flex column）。所以顯示跑馬燈時不是逐一調整每個 HUD 元素的 `top`，而是用 `body.has-marquee #app{top:26px}` 把整個 `#app` 的頂邊往下推 26px（跑馬燈高度）——底下所有絕對定位的子元素會跟著一起平移，不用動任何既有 HUD 元素的座標。跑馬燈本身則是獨立的 `position:fixed` 疊層，蓋在最上面（`z-index:50`）。

使用警語（僅供個人娛樂與教學示範使用）＋「創作者：蔡豐全（Mark Tsai）」放在開始畫面（`#start-screen`）最下面，做成 `.footnote` 小字——這個遊戲沒有「設定頁」，開始畫面是唯一常駐、不需要進入遊戲/開攝影機就看得到的畫面，所以放這裡。**不要放進 `#hud`**（遊戲進行中畫面全被攝影機/canvas 佔滿，加常駐文字只會擋到水果）。

## 手機水果尺寸縮小（2026-08-05）

`spawnWave()` 原本水果半徑是固定 58~84px（絕對像素），在桌機（畫面較寬）比例正常，但手機螢幕窄很多時同樣的絕對像素值顯得過大（實測 390px 寬螢幕上，水果直徑 116~168px，佔螢幕寬度 30~43%）。**修法**：以螢幕較短邊（`Math.min(W,H)`）700px 為基準等比縮小，`sizeScale = clamp(min(W,H)/700, 0.45, 1)`，半徑乘上這個係數——**桌機／平板（較短邊 ≥700px）維持原尺寸完全不變**，手機（較短邊 <700px）等比縮小，下限 0.45 避免螢幕極窄時水果小到難點中。實測 390px 寬時水果直徑降到 70~90px（佔螢幕寬度 18~23%），1280×800 桌機尺寸下維持原本 58~84px 範圍不變（驗證用 `window.__game._test.spawnWave()` 這個測試 hook 直接生波比對半徑，沒有依賴 `requestAnimationFrame` 計時，也沒有進入攝影機模式）。

**⚠️ 驗證這次改動時刻意只測開始畫面**（截圖、量測 `#app`／`#marqueeBar` 座標），完全沒有點擊「開始遊戲」或「改用滑鼠／觸控玩」進入遊戲流程——見上方「⚠️ 用真實 Chrome 測試」的教訓，避免重蹈意外觸發攝影機的覆轍。

## 測試

- 預覽：Preview MCP `preview_start`（名稱 `fruit-ninja-cam`，port 8767），開 `http://localhost:8767/index.html`。
- 自動化測試走**滑鼠模式**（headless 無鏡頭）：點「改用滑鼠／觸控玩」，對 canvas 派發 `pointermove` 事件模擬揮砍。
- `window.__game` 暴露完整遊戲狀態（`state`、`mode`、`score`、`lives`、`fruits[]`…）供斷言；也可直接呼叫內部不可及時用它讀取水果座標來瞄準。
- 真實體感效果需使用者以實體攝影機手動驗證。
- **⚠️ 用真實 Chrome（非 headless sandbox）測試這個專案要格外小心**：2026-08-05 測試手機版面時，用座標／描述定位的點擊工具點「改用滑鼠／觸控玩」後畫面**仍然啟動了真實攝影機**並拍到使用者本人畫面（不是預期的滑鼠模式）——懷疑是那種點擊方式的座標／時機跟頁面初始化有 race condition，實際點到別的元素。**後來改用 `document.getElementById('mouse-btn').click()`（直接呼叫 DOM 元素的 `click()`，不靠座標）就再也沒有重現這個問題**，之後測試一律用這個方式進入滑鼠模式，點擊後先用 `evaluate` 讀 `window.__game.mode === 'mouse'` 確認過再繼續，不要邊點邊截圖。
- **`window.__game._test.spawnWave()`**：直接呼叫一次生成邏輯、把新水果塞進 `G.fruits`，不用等 `requestAnimationFrame` 計時（背景分頁時 rAF 會被瀏覽器節流，`elapsed`/`spawnTimer` 幾乎不會走），驗證水果尺寸、生成邏輯這類不需要真的跑動畫迴圈的改動時優先用這個，比等待真實時間流逝可靠得多。
