# Azure Blob + HTML5 Video 播放器示範 (player.html)

此 README 逐段說明 `wwwroot/player.html` 的內容與設計意圖，方便你快速理解與延伸此示範程式。

## 檔案位置
- `wwwroot/player.html` — HTML5 播放器示範頁面（本說明以此檔為準）。

---

## 檔案總覽
`player.html` 的主要功能：
- 使用 HTML5 `<video>` 播放來自 Azure Blob 的影片（使用 SAS URL）。
- 提供載入、快轉/倒退、章節跳轉、指定秒數跳轉、播放速率、自訂鍵盤快捷鍵等便利控制。
- 範例中考慮跨域（CORS）與 SAS 的使用提醒。

下方依檔案區塊分段說明每一段程式碼的目的、重要屬性與可調整項目。

---

## 1) 文件頭與基本樣式（`<head>`）
```html
<!doctype html>
<html lang="zh-Hant">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>Azure Blob + HTML5 Video 播放器示範</title>
  <style>
    :root { font-family: system-ui, -apple-system, "Segoe UI", Roboto, "Noto Sans TC", sans-serif; }
    body { margin: 24px; }
    .row { display: flex; gap: 12px; align-items: center; flex-wrap: wrap; }
    .controls { margin-top: 12px; display: grid; gap: 8px; max-width: 960px; }
    button { padding: 8px 12px; border-radius: 10px; border: 1px solid #ddd; cursor: pointer; }
    input[type="number"] { width: 8rem; padding: 6px 8px; }
    input[type="text"] { width: min(960px, 100%); padding: 8px; }
    .hint { color: #666; font-size: 14px; }
    video { width: min(960px, 100%); max-height: 70vh; background: #000; border-radius: 12px; }
    .kbd { padding: 2px 6px; border: 1px solid #bbb; border-bottom-width: 2px; border-radius: 6px; background: #f9f9f9; font-family: ui-monospace, SFMono-Regular, Menlo, monospace; }
  </style>
</head>
```
說明：
- 設定基本 metadata 與響應式 viewport。`
- 內嵌 CSS 提供簡潔版面樣式：按鈕、輸入欄、提示字、與影片尺寸。可改成外部 CSS 檔以便維護。
- `video` 的最大寬度用 `min(960px, 100%)`，適合桌面與窄螢幕同時顯示。

---

## 2) 頁面主體（基本 UI）
```html
<body>
  <h1>Azure Blob + HTML5 Video 快轉/跳點示範</h1>

  <label for="src">影片來源（Azure Blob SAS URL）</label><br/>
  <input id="src" type="text" value="...SAS URL..."/>
  <div class="row">
    <button id="load">載入</button>
    <span class="hint">若按下後無法播放，請確認 SAS 是否未過期，或在 Azure Storage 啟用 Blob 服務 CORS（GET/HEAD/OPTIONS）。</span>
  </div>

  <div style="margin-top:12px">
  <video id="player" controls preload="metadata" playsinline crossorigin="anonymous" controlsList="nodownload"></video>
  </div>
```
重點說明：
- `#src`：輸入欄，預設放一個 Azure Blob 的 SAS URL 作為示範。使用者可以貼入自己的 URL。
- `#load` 按鈕：觸發載入 `#src` 的 URL 到 `<video>`。
- `<video>` 屬性說明：
  - `controls`：顯示瀏覽器內建控制列（播放、暫停、進度等）。
  - `preload="metadata"`：只預載 metadata（例如 duration），避免自動下載整個影片。可改為 `auto` 或 `none`。
  - `playsinline`：行動裝置上保持內嵌播放（避免進入全螢幕）。
  - `crossorigin="anonymous"`：若要讀取跨域影片 metadata 或使用 canvas 等功能時需要；也需伺服器正確回應 CORS header。
  - `controlsList="nodownload"`：部分瀏覽器支援，用來移除內建的「下載」選項（但不是所有瀏覽器都遵守）。

---

## 3) 控制列（HTML 範例按鈕）
重要互動按鈕區塊：
- 快轉/倒退按鈕（使用 `data-seek` 屬性）
- 指定秒數跳轉輸入與 `#jump` 按鈕
- 變速按鈕（使用 `data-rate` 屬性）
- 範例章節按鈕（使用 `data-at` 屬性）

示意（節錄）：
```html
<div class="row">
  <button data-seek="-10">⏪ 倒退 10s</button>
  <button data-seek="-5">-5s</button>
  <button id="toggle">⏯️ 播放/暫停</button>
  <button data-seek="5">+5s</button>
  <button data-seek="10">⏩ 快轉 10s</button>
</div>
```
說明：按鈕用 `data-*` 屬性攜帶必要參數，方便在 JS 中統一綁定事件處理。

---

## 4) JavaScript — 基本輔助與快捷操作
整段 JS 放於頁尾，使用純 vanila JS，沒有外部依賴。程式分為幾個子功能：

### 4.1 工具與全域元素
```js
const $ = sel => document.querySelector(sel);
const player = $('#player');
const rateNow = $('#rateNow');
```
- `$` 簡化查詢。
- 快取 `player` 與 `rateNow` DOM 物件以便重複使用。

### 4.2 載入來源（`#load`）
```js
$('#load').addEventListener('click', () => {
  const url = $('#src').value.trim();
  player.src = url;
  player.load();
  const m = location.hash.match(/t=(\d+)/);
  if (m) {
    const s = parseInt(m[1], 10);
    player.addEventListener('loadedmetadata', () => { player.currentTime = s; }, { once: true });
  }
});
```
- 將 `#src` 的值指定給 `player.src`，呼叫 `player.load()` 觸發瀏覽器載入影片資源（或 metadata，視 `preload` 而定）。
- 支援 URL hash `#t=秒數` 的自動跳轉功能（取得 `location.hash`，在 `loadedmetadata` 後設定 `currentTime`）。
- 注意：如果影片跨域且伺服器未回應正確 CORS header，`duration` 可能為 `NaN` 或無法讀取。

### 4.3 頁面載入時自動載入一次
```js
window.addEventListener('DOMContentLoaded', () => $('#load').click());
```
- 方便示範：頁面載入時會自動按一次「載入」，把 input 中預設的 SAS URL 載入。

### 4.4 快轉/倒退按鈕（`data-seek`）
```js
document.querySelectorAll('button[data-seek]').forEach(btn => {
  btn.addEventListener('click', () => { player.currentTime += parseFloat(btn.dataset.seek); });
});
```
- 取出 `data-seek`，加到 `currentTime`，可正可負。
- 若影片尚未加載完成或 `duration` 未就緒，瀏覽器會依情況處理（有些瀏覽器會等待 metadata）。

### 4.5 章節跳轉（`data-at`）
```js
document.querySelectorAll('.chapter').forEach(btn => {
  btn.addEventListener('click', () => { player.currentTime = parseFloat(btn.dataset.at); player.play(); });
});
```
- 直接將 `currentTime` 設為章節秒數，並呼叫 `play()`。

### 4.6 播放/暫停切換
```js
$('#toggle').addEventListener('click', () => { player.paused ? player.play() : player.pause(); });
```
- 使用 `player.paused` 判斷目前狀態，切換播放或暫停。

### 4.7 指定秒數跳轉
```js
$('#jump').addEventListener('click', () => {
  const s = parseFloat($('#sec').value || '0');
  if (!Number.isNaN(s)) player.currentTime = s;
});
```
- 從 `#sec` 取得數值並設為 `currentTime`。

### 4.8 變速播放（`data-rate`）
```js
document.querySelectorAll('button[data-rate]').forEach(btn => {
  btn.addEventListener('click', () => {
    player.playbackRate = parseFloat(btn.dataset.rate);
    rateNow.textContent = `目前速率：${player.playbackRate}×`;
  });
});
```
- 透過 `playbackRate` 調整播放速率，並在 `#rateNow` 顯示當前速率。

### 4.9 鍵盤快捷鍵
```js
document.addEventListener('keydown', e => {
  const tag = (e.target && e.target.tagName) || '';
  if (tag === 'INPUT' || tag === 'TEXTAREA') return; // 避免輸入框衝突
  if (e.key.toLowerCase() === 'j') player.currentTime -= 5;
  if (e.key.toLowerCase() === 'l') player.currentTime += 5;
  if (e.key.toLowerCase() === 'k') player.paused ? player.play() : player.pause();
  if (/^[0-9]$/.test(e.key)) {
    const pct = parseInt(e.key, 10) / 10; // 0–0.9
    if (Number.isFinite(player.duration)) player.currentTime = player.duration * pct;
  }
  if (e.key === 'ArrowLeft') player.currentTime -= 5;
  if (e.key === 'ArrowRight') player.currentTime += 5;
});
```
- 支援 J/L/K（YouTube 風格）、左右箭頭與 0–9 跳百分比（例如按 3 跳到 30%）。
- 先判斷目標元素是否為輸入欄以避免干擾使用者輸入。

### 4.10 顯示當前速率事件
```js
player.addEventListener('ratechange', () => { rateNow.textContent = `目前速率：${player.playbackRate}×`; });
```
- 當 `playbackRate` 變更時更新顯示。

---

## 5) 可客製化／延伸建議
- UI
  - 將內嵌 CSS 轉為 `css/site.css` 或其他檔案，便於維護與主題化。
  - 新增進度條自製 UI，可精確控制 hover 時顯示時間（若需在進度列上顯示 tooltip，可在 JS 中換算像素→時間）。

- 功能
  - 加入播放清單（playlist）並支援下一首/上一首。
  - 儲存上次播放位置（localStorage）以便恢復。
  - 加入快照截圖（使用 canvas drawImage，需要影片與原始來源允許 CORS）。

- 相容性
  - `controlsList="nodownload"` 並非所有瀏覽器都強制遵守，若要完全阻擋下載，需在伺服器端限制或採用授權/加密串流方案（如 Azure Media Services）。

---

## 6) 疑難排解
- 無法讀取 `duration` 或播放失敗：
  - 檢查 Blob SAS 是否過期。
  - 確認伺服器有回傳正確的 CORS header：`Access-Control-Allow-Origin: *`（或你的網域）、允許方法 `GET, HEAD, OPTIONS`。
  - 如果使用 `crossorigin`，確保伺服器同時回傳 `Access-Control-Allow-Credentials`（若設定 credentials）。

- 「下載」按鈕仍然出現：
  - 有些瀏覽器（或使用者代理）可能不支援 `controlsList`。
  - 若需更嚴格保護，考慮後端串流或 DRM 方案。

---

## 7) 如何在本機測試
- 若此專案為 ASP.NET 專案，啟動後在瀏覽器開啟 `https://localhost:PORT/player.html`（或依專案設定的路徑）。
- 若只想簡單開啟靜態頁面，可使用簡易 HTTP server，例如 Python：

```powershell
# 在 repo 根目錄執行（Windows PowerShell）
python -m http.server 8000
# 然後在瀏覽器開啟 http://localhost:8000/wwwroot/player.html
```

---

## 結語
此 README 提供逐段說明與延伸建議，方便你修改或整合進更完整的播放器專案。若要我把 README 翻成英文、或把說明內嵌成 `player.html` 注釋、或自動產生簡短 API 文件，我可以繼續幫忙。
