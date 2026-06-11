# 貼給 Claude Code（或你用的 AI 編輯器）的開場 prompt

> 直接複製下面整段貼進去。重點是前面那三條硬限制，把它們講死，AI 才不會自作主張加後端。

---

你要幫我繼續做一個很小的網頁工具。先把這些限制看清楚，它們不能違反、而且決定了整個架構。

**功能**：一個靜態網頁，使用者把影片拖進來，工具會在影片頂部加一條白 band、在裡面畫一個漫畫風格的對話框（可打字、可留空、尖角可朝上/朝下、高度可調），然後輸出一支新的 MP4 給使用者下載。用途是做 Threads 那種「嘲諷對話框」反應迷因。

**硬限制（絕不違反）**：
1. 100% 在瀏覽器端做。**沒有後端、沒有 server、沒有資料庫、沒有任何會收到影片的 API。**
2. 使用者的影片**絕不上傳、絕不儲存**到任何地方，全部在瀏覽器裡處理。
3. 必須能當靜態網站部署在 **Cloudflare Pages 免費方案**。優先「無 build step」（單一 index.html）；最多接受極簡 Vite。

**技術方向**：
- 影片合成用 `ffmpeg.wasm`，**單執行緒 core**（`@ffmpeg/core` 0.12.x），這樣**不需要** SharedArrayBuffer / cross-origin isolation。除非我明講，否則不要換成多執行緒 core。
- 疊圖做法：讀影片寬高 → 在 canvas 上以原始解析度畫出頂部 band（白底 + 漫畫氣泡 + 尖角 + 選填文字）→ 匯出 PNG → ffmpeg pad + overlay：
  `[0:v]pad=iw:ih+B:0:B:color=white[bg];[bg][1:v]overlay=0:0[v]`，`-map [v] -map 0:a?`，編碼 `libx264 / yuv420p / aac / +faststart`。
- 強制偶數尺寸（h264 要求）。
- 即時預覽用 2D canvas，且要跟匯出用**同一套 band 計算**，做到所見即所得。

**如果這個資料夾裡已經有 `index.html`，就在它上面改，不要從零重寫。** 第一件事先修這個已知 bug：匯出用的 PNG 可能在 Web 字型載入完成「之前」就畫好，導致輸出影片裡的中文用到 fallback 字。請在畫匯出 band 之前先 `await document.fonts.ready`。

**接著依序做**：
1. （上面那個字型 bug 修好）
2. 對話框可拖曳 / 縮放 / 左右移位。
3. 氣泡樣式變體：雲朵框、爆炸框、方框/圓框。
4. 常用吐槽句一鍵插入。
5. 時長 / 檔案大小護欄 + 更清楚的進度。

保持單一 `index.html`，除了 ffmpeg.wasm 的 CDN script 之外不要加依賴，除非我同意。**在引入任何 build 系統、框架或後端之前先問我。**

---

## 本機怎麼跑（提醒 AI 也提醒你自己）

不要用 `file://` 直接開（wasm worker 有 CORS）。用：

```bash
npx serve .
# 或 python3 -m http.server
```
