# 資料庫結構

以下是資料庫版本變更的代表性子集，呈現主要資料表、權限政策與防護機制的演進過程——**不是完整的部署腳本**。正式環境的完整結構包含本資料夾未收錄的設定表、通知紀錄與查詢函式；若要從零重建完整環境，需另外整理完整的 schema 匯出。

| 檔案 | 內容 |
|---|---|
| `001_orders.sql` | 建立 `orders` 訂單主表，以及最基本的 RLS 政策：匿名使用者只能新增訂單，登入的店主才能查看與更新 |
| `002_order_counter.sql` | 取餐編號改由資料庫統一發號（而非前端各自計數），避免不同裝置產生相同編號 |
| `003_order_confirm_flow.sql` | 訂單狀態機升級：新增確認流程所需欄位與狀態轉換規則 |
| `004_line_push.sql` | LINE 推播所需的資料庫端設定：記錄推播對象、通知開關 |
| `005_menu_v2.sql` | 菜單資料結構重構，改為雲端維護，取代前端寫死的備用菜單 |
| `006_security.sql` | 資安強化：複查並補齊各資料表的 RLS 政策，鎖死敏感設定表（如 LINE token 保險櫃）僅伺服器端可讀 |
| `007_turnstile.sql` | Cloudflare Turnstile 人機驗證所需的資料庫端設定 |
| `008_antiabuse.sql` | 防洗版限流、高額下單警示、棄單黑名單機制 |

## 權限模型的演進

訂單寫入權限分兩階段收斂，`001_orders.sql` 與 `007_turnstile.sql` 記錄了這個過程：

**第一階段（`001_orders.sql`）**：先讓客人瀏覽器用 `anon` 金鑰直接寫入，快速讓第一版上線：

```sql
alter table public.orders enable row level security;

create policy "客人可以送出訂單"
  on public.orders
  for insert
  to anon
  with check (true);

create policy "老闆登入後可以看訂單"
  on public.orders
  for select
  to authenticated
  using (true);
```

**第二階段（`007_turnstile.sql`）**：待前端改為呼叫 `place-order`（身分驗證、人機驗證、限流都在這一關完成）並驗證過後，收回匿名直寫權限，之後所有訂單一律經 `place-order` 以 `service_role` 寫入：

```sql
drop policy if exists "客人可以送出訂單" on public.orders;
```

這段刻意保留成一份「操作手冊」而不是直接套用，是因為前後端的切換需要先後順序——客人端改完才能關門，順序顛倒會讓客人暫時送不出單。

## 已知的權限設計缺口

- `orders` 的 `select`／`update` 政策目前對所有 `authenticated` 角色一視同仁，沒有另外限定「這個帳號是不是真正的店主」，目前靠關閉自助註冊降低風險，但不是資料庫層的強制保證
- 部分統計用的資料庫函式（`security definer`）尚未像 `next_order_no()` 一樣明確用 `revoke`／`grant` 限定可呼叫的角色，執行權限的收斂還不徹底
