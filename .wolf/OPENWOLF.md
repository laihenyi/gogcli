# OpenWolf Operating Protocol（gogcli 精簡版）

本專案用 `.wolf/` 在不同 harness（Claude Code、Pi、Codex 等）之間交接工作狀態。以下規則每個 session 都適用。

## Session 開始

在做任何事之前：

1. 檢查 `.wolf/handoff.md` 是否存在。存在就**完整讀完**：它是前一個 session 壓縮後的狀態、已驗證的成果、未完成項目、陷阱與建議的下一步。
2. 讀完後用一兩句話向使用者確認你看到的收尾狀態與 `next_action`，再開始工作。
3. handoff 的 front matter 有 `local_head`、`remote_head`、`sync`；先跑 `git rev-parse --short HEAD`、`git rev-parse --short @{u}`、`git status --short` 比對，不一致時以 repo 實際狀態為準，並明說文件哪些部分已過時。
4. **不要刪除 `.wolf/handoff.md`**。它由 `/handoff` 覆寫；只有使用者確認 session 完全結束時才手動清除。

## Session 結束或切換

- 使用者要求交接、對話即將被壓縮、或交付完成但仍有未完成項目時，執行 `/handoff`。
- `/handoff` 會同時寫 `.wolf/handoff.md`、`~/.claude/handoffs/latest-gogcli.md` 與 `~/.claude/handoffs/archive/gogcli/<UTC>.md`，三份內容相同；`.wolf/handoff.md` 存在時以它為準。
- 前一版未完成的項目不得無聲消失：完成的才移除，並註明關閉它的 commit 或 PR。

## 其他

- `.wolf/handoff.md` 已列入 `.gitignore`，屬本機交接狀態，不進版本控制；本檔可提交。
- 本專案沒有 `.wolf/cerebrum.md`、`.wolf/anatomy.md`、`.wolf/memory.md`；不需要維護它們。
- 本 repo 是 gogcli 的 fork（remote laihenyi/gogcli），同步上游時勿把 `.wolf/` 帶進 PR。
- 專案本身的開發約定見 `AGENTS.md`；Google Workspace 操作（gog 帳號、授權、排程規則）依 Claude Code 的 `pe-google` skill。
