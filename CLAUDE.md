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

## 測試

- 預覽：Preview MCP `preview_start`（名稱 `fruit-ninja-cam`，port 8767），開 `http://localhost:8767/index.html`。
- 自動化測試走**滑鼠模式**（headless 無鏡頭）：點「改用滑鼠／觸控玩」，對 canvas 派發 `pointermove` 事件模擬揮砍。
- `window.__game` 暴露完整遊戲狀態（`state`、`mode`、`score`、`lives`、`fruits[]`…）供斷言；也可直接呼叫內部不可及時用它讀取水果座標來瞄準。
- 真實體感效果需使用者以實體攝影機手動驗證。
