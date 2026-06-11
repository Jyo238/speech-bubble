# 嘴泡 / Mouth Bubble — 專案 Handoff

一個把「漫畫對話框」壓在影片頂部、輸出成新 MP4 的小工具。用途是做 Threads 上那種「嘲諷對話框」反應迷因。

---

## 三個不能妥協的限制（它們決定了所有設計）

1. **部署在 Cloudflare（Pages）。**
2. **絕不上傳、絕不儲存使用者的影片（隱私）** → 所有處理都在瀏覽器端做。
3. **零成本** → 純靜態網站，沒有後端、沒有資料庫、沒有儲存。

這三條互相成全：沒有後端 → 沒地方存影片 → 不用付伺服器/儲存費。只要有人想加後端、加上傳 API，就是走錯方向。

---

## 架構

- 一份靜態 `index.html`。沒有 build step、沒有框架、沒有後端。
- 影片合成 100% 在瀏覽器跑，用 `ffmpeg.wasm`（**單執行緒** core）。
- 使用者的影片不離開裝置。唯一從網路抓的東西是 ffmpeg 引擎本身（wasm/js，來自 unpkg CDN），跟影片無關。
- 部署 = 把資料夾上傳到 Cloudflare Pages。

---

## 對話框是怎麼壓上去的

1. 用 `<video>` 讀出影片的寬高。
2. 算一個頂部「band」高度（影片高度的百分比，取偶數以符合 h264）。
3. 在離屏 `<canvas>` 上以原始解析度畫出整條 band（白底 + 漫畫氣泡 + 尖角 + 文字）→ 匯出成 PNG。
4. ffmpeg 先把影片頂部補一條白 band，再把 PNG 疊上去：
   ```
   [0:v]pad=iw:ih+B:0:B:color=white[bg];[bg][1:v]overlay=0:0[v]
   ```
   `-map [v] -map 0:a?`（音訊有就帶上）、`libx264 / yuv420p / aac / +faststart`。
5. 讀回 `out.mp4` → Blob → object URL → 預覽 + 下載。

即時預覽用一個 2D canvas，每個 rAF 畫「band + 當前影片畫面」，而且用的是跟匯出**同一套 band 計算**，所以所見即所得。

---

## 關鍵決策與理由

- **用單執行緒 ffmpeg core，不用多執行緒**：避開 `SharedArrayBuffer`，那東西需要 cross-origin isolation（COOP/COEP header）。單執行緒 = 零 header 設定、丟哪都能跑。代價是慢一點，做短迷因夠用。
- 之後若嫌慢 → 換 `@ffmpeg/core-mt`，加上 `_headers`（COOP/COEP），但要接受某些嵌入情境在 isolation 下會壞掉。
- **強制偶數尺寸**（h264 要求）：寬高向下取偶、band 取偶。

---

## 檔案

- `index.html` — 整個 app。
- `_headers` — 選用，只有走「多執行緒升級路線」時才需要。

---

## 本機跑

wasm worker 有 CORS 限制，**不要直接用 `file://` 開**，要用 http 伺服器：

```bash
npx serve .
# 或
python3 -m http.server
```

然後開 localhost 那個網址。

---

## 部署到 Cloudflare Pages

1. 一個資料夾，放 `index.html`（要的話加 `_headers`）。
2. Cloudflare 後台 → Workers & Pages → Create → Pages → Upload assets（把資料夾拖上去）。或接 Git repo。
3. 拿到 `xxx.pages.dev` 網址。

---

## 已知限制 / 雷點

- 第一次匯出會下載約 25MB 引擎（之後快取）。10 秒 / 720p 大約數十秒，手機更慢。建議提醒使用者剪短。
- 會擋 CDN fetch 或 SAB 的沙箱 iframe（例如某些線上預覽框）裡，「匯出」那一步跑不動；正式部署到 CF 上就正常。
- 影片太長太大可能撞到瀏覽器 wasm 記憶體上限。可考慮加「時長上限」提示。
- **字型時序 bug（先修這個）**：匯出用的 PNG 可能在 Web 字型載入完成「之前」就畫好了，導致輸出影片裡的中文變成 fallback 字（豆腐/錯字）。在畫匯出 band 之前要先 `await document.fonts.ready`。

---

## 後續待辦（建議優先序）

1. 匯出前 `await document.fonts.ready`，確保中文字型正確（見上）。
2. 對話框可拖曳 / 縮放 / 左右移位。
3. 氣泡樣式變體：雲朵框（思考）、爆炸框、方框 vs 圓框。
4. 常用吐槽句一鍵插入。
5. 時長與檔案大小的護欄 + 更清楚的進度顯示。
6. 記住上次設定（localStorage；正式部署上沒問題）。
7. 可選：在支援的瀏覽器走 WebCodecs 加速，ffmpeg.wasm 當 fallback。
