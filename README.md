# 嘴泡 Mouth Bubble

把「漫畫對話框」壓在影片或圖片頂部、直接在瀏覽器輸出新檔的迷因小工具。
用途是做 Threads 上那種「嘲諷對話框」反應迷因。

**線上版：https://speech-bubble-9d5.pages.dev**

<img src="demo/mouthbubble.png" alt="輸出範例" width="340" />

> A tiny in-browser meme tool: drop a video or an image, stamp the classic Threads-style
> speech bubble band on top, export a new MP4 / PNG — 100% client-side, your file never
> leaves your device.

---

## 三條不能妥協的紅線

1. **100% 瀏覽器端處理** — 沒有後端、沒有 server、沒有資料庫、沒有任何會收到影片的 API。
2. **影片絕不上傳、絕不儲存** — 剪輯、合成、輸出全部在你的裝置裡完成。
3. **零成本靜態部署** — 一份 `index.html` 丟上 Cloudflare Pages 免費方案就能跑，無 build step。

這三條互相成全：沒有後端 → 沒地方存影片 → 不用付伺服器費。
任何「加個上傳 API」的想法都是走錯方向。

## 功能

刻意做到最少——丟檔案、拖尖角、輸出，三步完事：

- 拖曳 / 點選載入**影片**（mp4 / mov / webm）或**圖片**（jpg / png），效果一樣
- 自動壓上 Threads 經典款對話框：巨型氣泡只露出底邊那條線 + 細針尖角（固定置中）
- 點一下預覽就能重新選檔
- 影片輸出 MP4（ffmpeg.wasm）；圖片輸出 PNG（純 canvas，秒出）
- 存檔三層策略，檔名固定 `原檔名_bubble.mp4/png`：桌面 Chromium 跳「另存新檔」對話框；
  手機（iOS 15+ / Android）跳系統分享面板可直接存進相簿；其他瀏覽器退回一般下載
- 即時預覽與輸出用同一套計算 → 所見即所得
- 時長 / 檔案大小護欄提示、合成進度百分比

## 技術架構

- **單一 `index.html`，零 build step。** 介面用 [Vue 3](https://vuejs.org/)（CDN global build），不需要 npm / bundler。
- 影片合成用 [ffmpeg.wasm](https://ffmpegwasm.netlify.app/)（**單執行緒 core 0.12.x**），
  不需要 SharedArrayBuffer / cross-origin isolation，丟到任何靜態空間都能跑。
  UMD 版會直接從 CDN `new Worker()` 而被同源政策擋下，已用 `toBlobURL` 轉成
  同源 blob 再傳 `classWorkerURL` 解掉。
- 圖片不走 ffmpeg：直接 canvas 合成 → `toBlob` 輸出 PNG。
- 影片合成流程：讀影片寬高 → 在 canvas 以原始解析度畫出頂部 band（白底 + 氣泡）→ 匯出 PNG →
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

用 wrangler（只需要部署 `index.html` 一個檔）：

```bash
npx wrangler pages project create speech-bubble --production-branch main
npx wrangler pages deploy <只放 index.html 的資料夾> --project-name speech-bubble --branch main
```

或走後台：Workers & Pages → Create → Pages → Upload assets，把資料夾拖上去（或接這個 Git repo）。

> `_headers.multithread-example` 是選用範例：只有想換「多執行緒 ffmpeg core」（更快，但需要
> cross-origin isolation）時，才把它改名成 `_headers` 部署，並自行處理 CDN script 的 CORP/CORS。
> 預設的單執行緒版**不需要**它。

## 已知限制

- 第一次匯出會從 CDN 下載約 25 MB 的 ffmpeg 引擎（之後快取）。10 秒 / 720p 大約數十秒，手機更慢。
- 影片太長太大可能撞到瀏覽器 wasm 記憶體上限（介面會給護欄提示）。
- 會擋 CDN fetch 的沙箱 iframe 裡「匯出」跑不動；正式部署後正常。
- iPhone 的 HEIC 圖片在部分瀏覽器讀不出來（會明確提示），請改用 JPG / PNG。
- App 內建瀏覽器（LINE / FB / IG…）可能擋選檔或存檔，頁面會偵測並提示改用 Safari / Chrome 開啟。

## 廣告位

頁面預留了三個廣告位（虛線佔位框），要接聯播網時把代碼貼進對應的 `div`、刪掉裡面兩個 `span` 即可：

| id | 位置 | 建議尺寸 |
|---|---|---|
| `#ad-top` | 標題下方、工具上方 | 728×90 / 響應式橫幅 |
| `#ad-side` | 右欄輸出按鈕下方 | 300×250 |
| `#ad-bottom` | 隱私聲明下方、頁尾上方 | 728×90 / 響應式橫幅 |

## 開發文件

- [HANDOFF.md](HANDOFF.md) — 專案 handoff：架構、關鍵決策、雷點、待辦。
- [PROMPT.md](PROMPT.md) — 給 AI 編輯器的開場 prompt（記錄這個專案的硬限制）。

## License

[MIT](LICENSE)
