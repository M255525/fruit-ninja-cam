# fruit-ninja-cam — 體感切水果

單檔靜態網頁遊戲：`index.html` 即完整專案，無建置步驟、無 package.json。

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

## 手機版面修正（2026-08-05）

實測手機寬度（390px）發現 HUD 頂部的 `#level-info`（第 X 關＋倒數，`left:50%` 水平置中）跟 `#lives`（愛心，`right:18px` 靠右對齊）會疊在一起——兩者都是各自 `position:absolute` 假設螢幕夠寬才不會碰撞，實測窄於約 624px 就會重疊。**修法**：新增 `@media (max-width:700px)` 規則，把 `#level-info` 往下移到第二行（`top:54px`，字體縮到 17px），跟分數／愛心那一列分開，不論愛心格數多少都不會再碰在一起（已用 `getBoundingClientRect()` 實測前後座標確認重疊消失）。這是純 CSS 版面修正，跟手部追蹤／切割判定邏輯無關。

## 測試

- 預覽：Preview MCP `preview_start`（名稱 `fruit-ninja-cam`，port 8767），開 `http://localhost:8767/index.html`。
- 自動化測試走**滑鼠模式**（headless 無鏡頭）：點「改用滑鼠／觸控玩」，對 canvas 派發 `pointermove` 事件模擬揮砍。
- `window.__game` 暴露完整遊戲狀態（`state`、`mode`、`score`、`lives`、`fruits[]`…）供斷言；也可直接呼叫內部不可及時用它讀取水果座標來瞄準。
- 真實體感效果需使用者以實體攝影機手動驗證。
- **⚠️ 用真實 Chrome（非 headless sandbox）測試這個專案要格外小心**：2026-08-05 測試手機版面時，點擊「改用滑鼠／觸控玩」後畫面**仍然啟動了真實攝影機**並拍到使用者本人畫面（不是預期的滑鼠模式）——原因未完全查明，懷疑是瀏覽器已有先前授予的攝影機權限、且該次點擊時機/座標與頁面初始化有 race condition。**之後測試這個專案時，觸發任何「開始遊戲」相關按鈕前，务必先用 `document.getElementById('mouse-btn').disabled` 或監看 `mode-badge` 文字確認真的進入滑鼠模式再截圖**，避免意外擷取真人鏡頭畫面；若不慎拍到，立即在頁面內呼叫 `video.srcObject.getTracks().forEach(t=>t.stop())` 停止攝影機，並刪除該截圖檔案，不得保留或分享。
