# CLAUDE.md

> 這份檔案是一份**建置規格 / 提示詞**。把它放進一個空資料夾,對 Claude Code(或 Claude)說「**依照 CLAUDE.md 產生這個 app**」,就能重建出完整的程式。檔案本身不含任何第三方貼圖素材。

---

## 0. 目標(一句話)

做一支**純前端、單一 HTML 檔**的工具,可以**瀏覽** LINE 商店的貼圖組、**播放**某一組的動畫貼圖、並把整組**下載**成 ZIP 或產生**可離線使用的 HTML**。沒有後端、沒有建置步驟,雙擊或用本機伺服器打開即可運作。

## 1. 硬性限制

- 產出**一個** `.html` 檔(所有 HTML / CSS / JS 內嵌其中)。
- 不使用框架;**原生 JavaScript**。
- 唯一允許的外部相依:`JSZip`(從 cdnjs 載入)與 Google Fonts(可省略,省略則用系統字型)。
- **不得使用** `localStorage` / `sessionStorage` 以外仍可,但本程式不需要任何瀏覽器儲存。
- 不內嵌、不附帶任何貼圖圖檔;所有素材都是執行時即時抓取。
- 全程在瀏覽器執行,可離線打開(離線版產物尤其要零外部相依)。

## 2. 背後原理(務必理解,否則會做錯)

1. **資源在公開 CDN**:LINE 貼圖放在 `stickershop.line-scdn.net`,網址規則固定(見 §3)。
2. **動畫是 APNG**:動畫貼圖是 Animated PNG,瀏覽器原生支援。**放進 `<img>` 就會自動循環播放**,不需要任何解碼器或 canvas。播放器的核心就是「組出正確網址、塞進 img」。
3. **CORS 待遇不同(關鍵)**:
   - 用 `<img src>` **顯示**圖片**不受** CORS 限制 → 只要有 sticker id,動畫**一定播得出來**。
   - 用 `fetch` **讀取位元組 / HTML / JSON**(抓清單、抓頁面、抓圖打包)**會被 CORS 擋**,因為 LINE 不給授權標頭。
   - 設計原則:**把「一定可行的事(img 顯示)」與「可能被擋的事(fetch 讀取)」分開**,後者一律走「多代理輪詢 + 手動貼上保底」。

## 3. LINE CDN 端點(非官方慣例)

設 `CDN = https://stickershop.line-scdn.net/stickershop/v1`,`STORE = https://store.line.me/stickershop/showcase/top/zh-Hant`。

| 用途 | URL |
| --- | --- |
| 貼圖組清單(JSON) | `{CDN}/product/{pid}/iphone/productInfo.meta` → 含 `stickers[].id` |
| 靜態圖(首格) | `{CDN}/sticker/{id}/iphone/sticker@2x.png` |
| 動畫(APNG) | `{CDN}/sticker/{id}/iphone/sticker_animation@2x.png` |
| 商店縮圖 | `{CDN}/product/{id}/LINEStorePC/main.png` |
| 官方整包 ZIP | `{CDN}/product/{pid}/iphone/stickerpack@2x.zip` |
| 商店分類頁 | `{STORE}?category={cat}&page={n}` |

分類代碼:動態`1000014`、全螢幕`1000073`、特效`1000082`、大貼圖`1000080`、訊息`1000081`、隨你填`1000077`、動畫`1000015`、LINE FRIENDS`1`、迪士尼`2`、幽默`8`、吉祥物`9`。

## 4. UI(兩個分頁,一次只顯示一頁)

頂部 sticky 標題列:左邊品牌字 `LINE Sticker Player`,右邊一個 segmented control 切換「**瀏覽**」與「**播放器**」。`.view` 區塊用 `.active` 控制顯示;切換時捲到頂端。

### 瀏覽分頁
- 分類 `<select>` + 「載入清單」按鈕 + 上/下一頁(`‹ 第 N 頁 ›`)。
- 狀態列(可顯示錯誤,錯誤用警示色)。
- 結果:`.sets` 格狀清單,每張卡 = 縮圖(`main.png`,失敗退回 `sticker@2x.png`)+ 名稱 + `id · 類型` + 「開啟」鈕。
- 「開啟」→ 把該 id 填入播放器、**切到播放器分頁**、載入。
- 摺疊「方法 C」:貼上分類頁原始碼,本機解析成清單(零連線保底)。

### 播放器分頁
- 「‹ 清單」(切回瀏覽)+ `ID` 輸入框(預設 `1399310` 當範例)+「載入」+「全部暫停」+「下載全部 (ZIP)」+「產生離線版 HTML」。
- 第二列:大小 slider(調 `--cell`)+「官方整包 ZIP」連結(載入後顯示)。
- 狀態列 + `.grid` 貼圖格 + 空狀態提示。
- 每張卡:棋盤透明背景的舞台 + APNG `<img>` + 角落 id 標籤 + 一列「暫停 / 重播 / 原圖」。
- 摺疊「方法 A / B」:貼 sticker id(空白/逗號/換行分隔)、或貼 `productInfo.meta` 的 JSON,直接載入播放器。

## 5. 網路層:多代理輪詢

定義代理建構器(同一目標網址包成不同代理 URL):

```
SRC = {
  direct: u => ({u}),
  jina:   u => ({u:'https://r.jina.ai/'+u}),          // 回 Markdown
  ag:     u => ({u:'https://api.allorigins.win/get?url='+encodeURIComponent(u), json:1}),  // 回 {contents}
  cors:   u => ({u:'https://corsproxy.io/?url='+encodeURIComponent(u)}),
  thing:  u => ({u:'https://thingproxy.freeboard.io/fetch/'+u}),
  agraw:  u => ({u:'https://api.allorigins.win/raw?url='+encodeURIComponent(u)})
}
```

- `getText(url, order)`:依序試 order 內的代理,第一個成功(`json` 者取 `.contents`,否則取 `.text()`)就回傳;全失敗則 throw。
- `getBlob(url, order)`:同上但回傳 `blob`。
- 用法:
  - 商店分類頁 → `getText(url, ['jina','ag','cors','thing'])`
  - `productInfo.meta` → `getText(url, ['direct','ag','cors','thing'])` 後 `JSON.parse`
  - 圖片位元組 → `getBlob(url, ['direct','cors','thing','agraw'])`

## 6. 解析(同時吃 HTML 與 Markdown)

`parseSets(text)` 要能處理兩種回傳格式,回傳 `[{id,name,type}]`:
1. 先用 Markdown 連結正則:`/\[([^\]]*)\]\([^)]*\/stickershop\/product\/(\d+)[^)]*\)/g`(jina 輸出)。
2. 若無結果,用 `DOMParser` 找 `a[href*="/stickershop/product/"]`(HTML 代理輸出)。
3. 再無,退回純抓 `/stickershop/product/(\d+)`,名稱用 `#id`。
4. 以 id 去重。

`cleanName(raw)`:移除尾巴的類型字樣(`Animation only icon`、`Animation & Sound icon`、`Popup`、`Effect`、`Sound`…)並推得 `type`(動態 / 動+聲 / 全螢幕 / 特效 / 有聲);**商店會把名稱重複兩次**,需偵測「前半 === 後半」時取一半。

## 7. 播放 / 暫停(APNG 無原生暫停)

- **線上播放器**:暫停 = `img.src = sticker@2x.png`(靜態首格);播放 = `img.src = animation@2x.png + '?cb='+Date.now()`(cache-bust 讓動畫從頭)。
- **離線版**:圖是 data URI(同源、不污染 canvas)。暫停 = 把當前畫格 `drawImage` 到一個覆蓋的 `<canvas>` 定格、隱藏 `<img>`;播放 = 顯示 `<img>`、隱藏 canvas。

## 8. 下載 / 離線版

- **下載全部 ZIP**:對每個 id `getBlob(animation)`(失敗退 static),用 JSZip 以 `{id}.png` 命名打包後觸發下載。
- **產生離線版 HTML**:把每張抓成 blob → `FileReader` 轉 base64 data URI → **內嵌**進一份新 HTML 字串(連同 §7 的離線播放/暫停 JS 與 CSS),用 Blob 下載。產物**單一檔、零外部相依、永不需連網**。
- **⚠️ 實作陷阱**:離線 HTML 字串裡那段內嵌 `<script>` 的**結尾標籤必須寫成 `<\/script>`**(反斜線跳脫),否則會提前關閉外層 script。
- 兩者都讀位元組(需 CORS/代理);全失敗時引導使用者改用「官方整包 ZIP」連結。

## 9. 行為細節

- 重新載入清單(換分類 / 翻頁 / 方法 C)時呼叫 `resetPlayer()`:清空播放器格、停用下載鈕、隱藏 zip 連結、清狀態。
- 動畫 `<img>` `onerror`:先退回靜態圖,再失敗才顯示「無法載入」。
- 縮圖 `onerror`:退回 static,再失敗就隱藏。
- 尊重 `prefers-reduced-motion`:預設暫停。
- 加一個內嵌 SVG favicon(`data:image/svg+xml,...`)避免 favicon 404。

## 10. 視覺風格(可重現用)

深色號誌箱質感。色票:背景 `#0d100e`、面板 `#161b18`、訊號綠 `#1ee36a`(主色 / 啟用態)、停止紅 `#ff453a`、警示黃 `#ffcb47`、文字 `#eef2ee`、次要 `#8b948c`、線 `#2b332d`。字型:標題 `Space Grotesk`、數據/ID `IBM Plex Mono`。透明貼圖用淺灰棋盤背景襯出。卡片圓角 14px。

## 11. 支援範圍(在 UI 或註解中誠實標示)

- 動態 / 有聲動態:動畫可播(聲音未播)。
- 靜態 / 大貼圖:退回顯示靜態。
- 全螢幕 / 特效:資源為 `popup` 類,目前只試一般動畫 → 會退回靜態(可擴充)。
- 訊息 / 隨你填:重點是動態合成的使用者文字,抓圖無法重現。

## 12. 完成標準

1. 「載入清單」能列出貼圖組(代理可用時);代理全掛時方法 C 可手動列出。
2. 點「開啟」或輸入 ID「載入」,能看到整組 APNG 動畫播放。
3. 暫停 / 重播 / 全部暫停皆正常。
4. 「下載全部 ZIP」與「產生離線版 HTML」可運作;離線檔打開後零連線、可播放/暫停。
5. 換清單時播放器會清空。
6. 全程無 console 語法錯誤;`</script>` 跳脫正確。

## 13. 聲明(請保留)

端點為非官方、未公開,LINE 隨時可能變動。公開代理時靈時不靈、有流量限制。貼圖版權屬原作者所有,本工具僅供個人預覽研究,**不附帶任何貼圖素材**,亦不應用於散布或商業用途。
