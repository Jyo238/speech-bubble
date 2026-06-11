# 嘴泡 Mouth Bubble

把「漫畫對話框」壓在影片頂部、直接在瀏覽器輸出新 MP4 的迷因小工具。
用途是做 Threads 上那種「嘲諷對話框」反應迷因。

**[線上試玩 Demo](https://github.com/Jyo238/speech-bubble)**（部署到 Cloudflare Pages 後把網址貼這裡）

> A tiny in-browser meme tool: drop a video, draw a comic speech bubble on a white
> band above it, export a new MP4 — 100% client-side, your video never leaves your device.

---

## 三條不能妥協的紅線

1. **100% 瀏覽器端處理** — 沒有後端、沒有 server、沒有資料庫、沒有任何會收到影片的 API。
2. **影片絕不上傳、絕不儲存** — 剪輯、合成、輸出全部在你的裝置裡完成。
3. **零成本靜態部署** — 一份 `index.html` 丟上 Cloudflare Pages 免費方案就能跑，無 build step。

這三條互相成全：沒有後端 → 沒地方存影片 → 不用付伺服器費。
任何「加個上傳 API」的想法都是走錯方向。

## 功能

- 拖曳 / 點選載入影片（mp4 / mov / webm）
- 五種氣泡樣式：**脆版**（巨型氣泡只露底邊 + 細尖角，Threads 經典款）、圓框、方框、雲朵（思考）、爆炸
- 對話框文字（中文逐字折行、自動縮放），常用吐槽句一鍵插入
- 尖角朝上 / 朝下，**直接在預覽畫面拖曳氣泡與尖角**左右移位，氣泡寬度可縮放
- 對話框高度可調，即時預覽與輸出用同一套計算 → 所見即所得
- 匯出前等待 Web 字型載入完成，中文字不會變豆腐
- 時長 / 檔案大小護欄提示、合成進度百分比
- 設定（樣式 / 朝向 / 高度 / 寬度）記在 localStorage

## 技術架構

- **單一 `index.html`，零 build step。** 介面用 [Vue 3](https://vuejs.org/)（CDN global build），不需要 npm / bundler。
- 影片合成用 [ffmpeg.wasm](https://ffmpegwasm.netlify.app/)（**單執行緒 core 0.12.x**），
  不需要 SharedArrayBuffer / cross-origin isolation，丟到任何靜態空間都能跑。
- 合成流程：讀影片寬高 → 在 canvas 以原始解析度畫出頂部 band（白底 + 氣泡 + 文字）→ 匯出 PNG →
  ```
  [0:v]crop=W:H:0:0[c];[c]pad=iw:ih+B:0:B:color=white[bg];[bg][1:v]overlay=0:0[v]
  ```
  `-map [v] -map 0:a?`，編碼 `libx264 / yuv420p / aac / +faststart`，全程強制偶數尺寸（h264 要求）。

## 本機跑

wasm worker 有 CORS 限制，**不要用 `file://` 直接開**，要起一個 http 伺服器：

```bash
npx serve .
# 或
python3 -m http.server
```

然後開 localhost 那個網址。

## 部署到 Cloudflare Pages

1. Cloudflare 後台 → Workers & Pages → Create → Pages → Upload assets，把資料夾拖上去（或接這個 Git repo）。
2. 拿到 `xxx.pages.dev` 網址，完事。

> `_headers.multithread-example` 是選用範例：只有想換「多執行緒 ffmpeg core」（更快，但需要
> cross-origin isolation）時，才把它改名成 `_headers` 部署，並自行處理 CDN script 的 CORP/CORS。
> 預設的單執行緒版**不需要**它。

## 已知限制

- 第一次匯出會從 CDN 下載約 25 MB 的 ffmpeg 引擎（之後快取）。10 秒 / 720p 大約數十秒，手機更慢。
- 影片太長太大可能撞到瀏覽器 wasm 記憶體上限（介面會給護欄提示）。
- 會擋 CDN fetch 的沙箱 iframe 裡「匯出」跑不動；正式部署後正常。

## 開發文件

- [HANDOFF.md](HANDOFF.md) — 專案 handoff：架構、關鍵決策、雷點、待辦。
- [PROMPT.md](PROMPT.md) — 給 AI 編輯器的開場 prompt（記錄這個專案的硬限制）。

## License

[MIT](LICENSE)
