# FURUKE 維護頁（GitHub Pages 靜態版）

停機維護期間對外顯示的靜態頁面。純 HTML／CSS／JS，無 build step、無外部依賴。

```
maintenance-page/
├── index.html    # 維護頁本體（樣式、文案、倒數計時都在裡面）
├── 404.html      # index.html 的複本，讓任何路徑都顯示維護頁
├── favicon.ico   # 取自 public/favicon.ico
├── img/logo/     # apple-touch-icon 與頁面 logo，取自 public/img/logo/
├── robots.txt    # Disallow: /，避免被收錄
└── .nojekyll     # 跳過 GitHub 的 Jekyll 處理
```

頁面 logo 以 base64 內嵌在 HTML 裡，所以 `404.html` 在 `/foo/bar/baz` 這種深度路徑
也能正常顯示（相對路徑在那裡會失效）。favicon 走絕對路徑 `/favicon.ico`，
因此**必須掛在網域根目錄**（自訂網域），不能用 `username.github.io/repo/` 這種子路徑形式。

## 修改維護時間

只需要改 `index.html` 裡 `<script>` 開頭那段：

```js
var MAINTENANCE = {
  start: '2026-08-06T08:00:00+08:00',   // ISO 8601，務必帶時區
  end: '2026-08-06T12:00:00+08:00',
  siteUrl: 'https://www.furuke.com/'    // 維護結束後把使用者送回哪裡
};
```

改完務必同步 404：

```bash
cd maintenance-page && cp index.html 404.html
```

語言依瀏覽器 `navigator.language` 自動判斷（含 `zh` → 繁中，其餘 → 英文），
使用者可用頁面下方的連結手動切換（記在 localStorage），也支援 `?lang=en`。
倒數結束後會自動跳回 `siteUrl`。頁面只有淺色樣式，不跟隨系統深色模式。

## 部署到 GitHub Pages

GitHub Pages 在**私有 repo** 需要付費方案，所以建議另開一個**公開** repo 專放這頁：

```bash
cd maintenance-page && git init && git add -A && git commit -m "FURUKE maintenance page"
```

```bash
cd maintenance-page && git remote add origin git@github.com:furuke/maintenance.git && git push -u origin main
```

1. repo → Settings → Pages → Source 選 **Deploy from a branch** → `main` / `(root)`
2. Custom domain 填 `maintenance.furuke.com`（會自動 commit 一個 `CNAME` 檔）
3. Cloudflare DNS 加一筆 `CNAME`：`maintenance` → `furuke.github.io`，
   **Proxy status 要設成 DNS only（灰雲）**，否則 GitHub 無法簽發憑證
4. 等 Pages 顯示憑證就緒後，勾選 **Enforce HTTPS**

先自己開 `https://maintenance.furuke.com/` 和 `https://maintenance.furuke.com/whatever`
確認兩個路徑都顯示維護頁再往下做。

## 維護期間把流量導過來

GitHub Pages 攔不到 `www.furuke.com` 的流量，要在 Cloudflare 端導。
用 **Rules → Redirects（Single Redirect）**，免費方案就有，不需要 Worker：

- 規則名稱：`maintenance`
- When incoming requests match：`hostname equals www.furuke.com`
  （要放行的路徑在這裡排除，例如 `and not starts_with(http.request.uri.path, "/webhooks/")`）
- Then：Static redirect → `https://maintenance.furuke.com/`
- Status code：**302 (Found)** — 不要用 301，301 會被瀏覽器長期快取，維護結束後很難收回
- Preserve query string：關閉

維護結束就把規則 **停用**（不用刪除，下次維護直接開回來）。

要擋的 hostname 不只一個（`furuke.com`、其他子網域）就把條件改成
`hostname in {"furuke.com" "www.furuke.com"}`。

## 這個做法的限制

改用靜態頁面後，有幾件事是 Worker 版做得到、這個版本做不到的，先確認可以接受：

| 項目 | 說明 |
|------|------|
| **HTTP 狀態碼** | GitHub Pages 只能回 `200`／`404`，導流過去是 `302`，**沒有 `503`**。搜尋引擎不會認得這是暫時性停機，有被當成正式內容收錄的風險。已用 `noindex` meta + `robots.txt` 降低影響，但保護程度不如 `503`。 |
| **`Retry-After`** | 無法設定，爬蟲與 client 不知道何時該回來。 |
| **API 回應** | 無法針對 `/api/*` 回 JSON，App／前端 AJAX 會拿到一段 HTML 或跟著 302 跳轉，錯誤處理可能出現奇怪畫面。 |
| **團隊 bypass** | 沒有 token 機制。要在維護中驗證線上站，得改用 Cloudflare 規則排除自己的 IP，或暫時停用 redirect 規則。 |
| **維護視窗自動生效** | 靜態頁不會自己開關流量，redirect 規則要人工啟用／停用（頁面上的倒數只是顯示用）。 |

如果 `503` 與 API JSON 這兩點之後變成問題，Worker 版仍留在
[`cloudflare/maintenance-worker/`](../cloudflare/maintenance-worker/)，兩者頁面設計相同。

## 更新品牌圖檔

本站換 logo／favicon 時：

```bash
cp public/favicon.ico maintenance-page/ && cp public/img/logo/icon-192x192.png public/img/logo/with-text.png maintenance-page/img/logo/
```

`with-text.png` 還要重新產生內嵌的 base64，換掉 `index.html` 裡 `class="logo"` 那個
`src="data:image/png;base64,..."`：

```bash
base64 -i public/img/logo/with-text.png | tr -d '\n' | pbcopy
```
