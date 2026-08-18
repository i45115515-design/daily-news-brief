# daily-news-brief

這個 repo 只有一個用途：存放「每日資訊彙整」排程任務（台灣時間 07:00 與 22:00 各一次）產出的公開 HTML 報告，放在 `daily-news-reports/`，並透過 GitHub Pages 對外發布。每次執行都是全新、沒有記憶的容器。

## ⚠️ 已知重大事故與必要防護（每次涉及 git commit/push 都要遵守）

**2026-08-09 發現**：連續 3 天、6 次排程執行（2026-08-06 ~ 2026-08-08）的報告雖然都在本地 commit 成功，卻**從未真正推送到 GitHub**——GitHub Pages 與依賴 push 觸發的 Discord 通知因此整整靜默 3 天。

根本原因：這個環境每次啟動時，git working copy 有時會是 **detached HEAD** 狀態（而不是 checkout 在 `master` 分支上）。如果在 detached HEAD 底下執行 `git commit`，commit 只會前進「detached HEAD」這個匿名指標，不會移動本地的 `master` 分支引用。接著若執行 `git push -u origin master`，git push 的是本地那個「原地沒動」的 `master` 分支，而不是剛剛 commit 的內容——指令通常不會報錯（甚至顯示 `Everything up-to-date`），看起來像成功了，實際上什麼都沒送到 GitHub。

**因此，任何要在這個 repo commit/push 的任務，一律要做以下兩件事：**

### 1. commit 之前：確認不是 detached HEAD

```bash
git branch --show-current
```

- 有印出分支名（例如 `master`）→ 正常，繼續。
- 沒印出任何東西（detached HEAD）→ 執行以下指令把 HEAD 接回 `master`，**不要用 `git checkout master`**（若有未 commit 的變更會被擋下來）：
  ```bash
  git symbolic-ref HEAD refs/heads/master
  ```
  執行後再跑一次 `git branch --show-current` 確認已經是 `master`，才能繼續 `git add` / `git commit`。

  如果這時 `git status` 顯示大量「意外」的 staged 變更（因為 detached HEAD 上可能已經有更早的、未 push 的歷史 commit），**不要直接 commit 全部混在一起**——用 `git reset --soft <上一個已知乾淨的 commit>`（例如目前 detached HEAD 上最新的一個歷史 commit）只保留這次真正要新增的檔案變更在 index 裡，其餘歷史 commit 維持各自獨立，再正常 commit 這次的變更，避免把好幾天的 commit 訊息擠壓成一個。

### 2. push 之後：驗證真的送達，不能只看指令沒報錯

```bash
git push -u origin master
git fetch origin master
git rev-parse HEAD origin/master
```

比對兩個 SHA：
- **一致** → 才算成功。
- **不一致** → push 沒有真正生效。重新檢查 `git branch --show-current` 是否為 `master`、`git log --oneline -1` 是否為剛剛的 commit，修正後重試 push，最多 3 次。3 次後仍不一致，**不要假裝完成**——在最終輸出中明確寫出「⚠️ push失敗，GitHub上的內容未更新，需要人工介入」，並附上本地 HEAD 與 origin/master 的 SHA 差異。

## Discord 通知架構

- 不要自己用 curl 連線 discord.com 或任何 webhook——這個環境的網路白名單會擋下來，一定失敗。
- 通知完全交給 `.github/workflows/notify-discord.yml`：只要 `daily-news-reports/latest-summary.json` 的內容在 push 到 GitHub 後有變化，GitHub Actions 會自動轉發到 Discord。
- 因此本 repo 唯一要做的事，就是把 `latest-summary.json` 寫對格式（Discord embed JSON）、成功 commit 且**驗證過確實 push 到 GitHub**。

### `DISCORD_WEBHOOK_URL` repo secret 狀態（2026-08-09 曾經全部失敗，2026-08-18 確認已修復）

**2026-08-09 發現**：檢查 `Notify Discord` workflow 的完整執行歷史（截至 2026-08-09，共 13 次 run），**全部 13 次都是 failure**，卡在 "Send to Discord" 這個 step，錯誤訊息固定是：

```
::error::DISCORD_WEBHOOK_URL repo secret is empty or not set. Add it in Settings > Secrets and variables > Actions.
```

**2026-08-18 22:13 這次排程執行時複查**：透過 GitHub Actions API 確認 `Notify Discord` workflow 從 2026-08-13 起（含這次執行）連續每一次 run 的 conclusion 都是 `success`，不再卡在上述錯誤。代表使用者已經在某個時間點（08-09 之後、08-13 之前）補上了 `DISCORD_WEBHOOK_URL` repo secret，這個問題目前視為已解決。

**注意**：這不代表以後永遠不會再壞（secret 可能被移除、webhook URL 可能失效等）。如果之後的排程執行又發現 Discord 完全沒收到通知，**先查 `Notify Discord` workflow 最近一次 run 的 log**（用 GitHub Actions API／MCP 工具：`mcp__github__actions_list` method `list_workflow_runs`、`resource_id` 填 `notify-discord.yml`），確認 conclusion 是否又變回 failure、以及是否是同樣的 secret 錯誤訊息，不要憑印象或只看這份文件的舊紀錄就下結論，也不要一開始就假設是 push 或 detached HEAD 的問題。

## 內容規範

- 這個 repo 完全公開，只放公開新聞摘要報告，絕對不要寫入任何使用者個人/敏感資料。
- 抓不到資料就照實寫「無法取得資料」，絕對不要編造標題或連結。
