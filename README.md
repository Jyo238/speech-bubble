# 嘴泡 Mouth Bubble

做 Threads 那種「嘲諷對話框」迷因用的小工具。
丟一支影片或一張圖進來，頂上自動壓一條漫畫對話框，輸出新檔，拿去發文。
所有處理都在你的瀏覽器裡完成，**檔案不會上傳**。

**線上版：https://speech-bubble-7ct.pages.dev**

<img src="demo/mouthbubble.png" alt="輸出範例" width="340" />

> Drop a video or an image, stamp the classic Threads-style speech bubble on top,
> export a new MP4 / PNG. Everything runs in your browser — your file is never uploaded.

## 功能

- 支援影片（mp4 / mov / webm）和圖片（jpg / png）
- 自動壓上 Threads 經典款對話框：巨型氣泡只露出底邊那條線 + 細針尖角
- 影片輸出 MP4（ffmpeg.wasm）；圖片輸出 PNG（canvas 直接合成，秒出）
- 存檔：桌面跳「另存新檔」對話框，手機跳系統分享面板可直接存進相簿，檔名為 `原檔名_bubble`
- 即時預覽與輸出用同一套計算，所見即所得
- 影片時長 / 檔案大小提示、合成進度百分比

## 技術

- 單一 `index.html`，無 build step。介面用 [Vue 3](https://vuejs.org/)（CDN），影片合成用
  [ffmpeg.wasm](https://ffmpegwasm.netlify.app/) 單執行緒 core——不需要
  SharedArrayBuffer / cross-origin isolation，丟到任何靜態空間都能跑
- 合成流程：canvas 以原始解析度畫出頂部 band → ffmpeg `pad` + `overlay` →
  `libx264 / yuv420p / aac / +faststart`，全程強制偶數尺寸
- 圖片不走 ffmpeg，canvas 合成後直接 `toBlob` 輸出
- 小提醒：ffmpeg.wasm 的 UMD 版會直接從 CDN `new Worker()` 而被同源政策擋下，
  這裡用 `toBlobURL` 轉成同源 blob 再傳給 `classWorkerURL`，core 要用 `dist/esm` 版

## 本機跑

wasm worker 有 CORS 限制，不能用 `file://` 直接開，要起一個 http 伺服器：

```bash
npx serve .
# 或
python3 -m http.server
```

## 部署到 Cloudflare Pages

只需要部署 `index.html` 一個檔：

```bash
npx wrangler pages project create speech-bubble --production-branch main
npx wrangler pages deploy <只放 index.html 的資料夾> --project-name speech-bubble --branch main
```

或走後台：Workers & Pages → Create → Pages → Upload assets。

## 已知限制

- 第一次匯出會從 CDN 下載約 25 MB 的 ffmpeg 引擎（之後快取）。10 秒 / 720p 大約數十秒，手機更慢
- 影片太長太大可能撞到瀏覽器 wasm 記憶體上限（介面會提示）
- iPhone 的 HEIC 圖片在部分瀏覽器讀不出來（會明確提示），請改用 JPG / PNG
- App 內建瀏覽器（LINE / FB / IG…）可能擋選檔或存檔，頁面會偵測並提示改用 Safari / Chrome 開啟

## License

[MIT](LICENSE)
