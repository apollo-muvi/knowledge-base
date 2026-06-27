# Cloudflare Zero Trust Access Policies

> **參考來源：** https://developers.cloudflare.com/cloudflare-one/access-controls/policies/
>
> **日期：** 2026-06-27 | **應用場景：** `cramai.rain0425.com`（Cloudflare Tunnel 後的自架應用）

---

## 概觀

Cloudflare Access 決定誰可以存取你的應用程式。每個 Policy 由四個要素組成：

1. **Actions** — Allow / Block / Bypass / Service Auth
2. **Rule types** — Include（OR）/ Require（AND）/ Exclude（NOT）
3. **Selectors** — 檢查的屬性（Email、IP、Country、Everyone...）
4. **Values** — 要比對的具體值

---

## Actions（行為）

| Action | 用途 | 說明 |
|:-------|:-----|:------|
| **Allow** | 允許符合條件的人通過 | 需設 Include（誰能進）+ 可選 Require / Exclude |
| **Block** | 封鎖特定的人 | 用來排除例外（default 是 deny） |
| **Bypass** | 完全跳過 Access 驗證 | ⚠️ 不紀錄、不安全。只用在必須公開的路徑 |
| **Service Auth** | 機器對機器，不用人登入 | 用 Service Token 或 mTLS，建議取代 Bypass |

### 注意點

- **Bypass 不建議永久開放內部應用** — 要提供員工零信任存取，應使用 Cloudflare One Client
- **Bypass 時，安全設定回歸 Zone 預設** — 如果 Zone 有開 Always Use HTTPS，Bypass 仍走 HTTPS
- **Service Auth 有 logging 且不需要 IdP 登入** — 適合 API / webhook 使用

---

## Rule Types（邏輯運算）

| Rule Type | 邏輯 | 說明 |
|:----------|:-----|:------|
| Include | OR | 符合任一條件即可（至少需一條 Include） |
| Require | AND | 必須**全部**符合 |
| Exclude | NOT | 符合者踢出，不允許存取 |

### 陷阱：Require 內的多個值預設是 AND

❌ 錯誤寫法（會導致無人能通過）：
```
Allow → Require Country = United States, Portugal
         Require Email ending in = @cloudflare.com, @contractors.com
```
結果：使用者要同時在美國**和**葡萄牙，且 email 同時結尾 `@cloudflare.com` **和** `@contractors.com`

✅ 解法：用 **Rule Group** 實現 OR
1. 建立 Rule Group `Country requirements` → Include Country = United States OR Portugal
2. Policy 設 `Require = Country requirements` + `Include = Email ending in @cloudflare.com OR @contractors.com`

---

## Policy 執行順序

1. **Service Auth**（先，照 UI 由上到下）
2. **Bypass**（第二）
3. **Allow / Block**（最後，照排列，中了就停，後面不繼續評估）

---

## 常用 Selectors

| Selector | 檢查時機 | 連續檢查 | 身份型 |
|:---------|:---------|:---------|:-------|
| Email | 登入時 | ❌ | ✅ |
| Email ending in | 登入時 | ❌ | ✅ |
| IP ranges | 每次請求 | ✅ | ❌ |
| Country | 每次請求 | ✅ | ❌ |
| Everyone | 登入時 | ❌ | ❌ |
| Service Token | 每次請求 | ✅ | ❌ |
| Valid Certificate | 每次請求 | ✅ | ❌ |
| Device Posture | 每次請求 | ✅ | ❌ |

> 非身份型 Selector（IP、Country、Service Token）在每次 HTTP 請求都會重新檢查，**session 期間如果有變動會即時生效**。

---

## 實際應用：cramai.rain0425.com

### 現況

- `cramai.rain0425.com` 已有 Cloudflare Tunnel → Pi4 `localhost:8000`
- App 本身有 JWT 登入（`admin` / `cramai456`）
- LINE Webhook path: `/line/*`
- Health check path: `/health`

### 建議 Policies

| # | Action | Path | Rule | 原因 |
|:-:|:-------|:-----|:-----|:-----|
| 1 | **Bypass** | `/line/*` | Everyone | LINE 伺服器不能登入 Access |
| 2 | **Bypass** | `/health` | Everyone | Deploy 腳本驗證用 |
| 3 | **Bypass** | `/api/auth/login` | Everyone | 登入 API 不能被擋 |
| 4 | **Allow** | `/` (全部) | Email OTP | 使用者輸入 email → 收驗證碼後進入 |

### 使用者流程（有 Access）

```
瀏覽器 → cramai.rain0425.com
         ↓
  🔒 Cloudflare Access 閘門（Email OTP）
     輸入 email → 收驗證碼 → 通過
         ↓
  🔒 App JWT 登入頁
     輸入 admin / cramai456 → 通過
         ↓
     Dashboard
```

### 注意事項

1. **雙重登入問題** — Cloudflare Access + App JWT 等於要登兩次。可考慮拿掉 App JWT 改用 Cloudflare Access 做唯一認證
2. **無法 local 測試** — Access 是 Cloudflare edge 功能，須直接在 Zero Trust Dashboard 設定後上線驗證
3. **Cloudflare Zero Trust Free Plan** — 50 users 免費額度，小補習班夠用
4. **設定位置** — https://one.dash.cloudflare.com/ → Access → Applications → Add application → Self-hosted → `cramai.rain0425.com`

---

## API / Terraform

Cloudflare Access 也可透過 API 或 Terraform 管理：

- API: https://developers.cloudflare.com/api/operations/access-applications-list-access-applications
- Terraform: https://developers.cloudflare.com/cloudflare-one/api-terraform/

---

## 常見錯誤配置

| 規則 | 結果 |
|:-----|:------|
| Include Everyone | 任何人都能存取（等於沒設） |
| Include Login Methods = One-time PIN | 所有能收到 OTP 的人都能進，不是你想要的限制 |