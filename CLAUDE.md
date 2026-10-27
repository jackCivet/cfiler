# CLAUDE.md

## セッション開始時
- `.claude/handover/` に handover ファイル（`YYYY-MM-DD_handover.md`）があれば、日付が最新のものを最初に読み、その「次にやること」から作業を再開すること。
- handover が無い場合（別の PC でクローンした直後など）は、`docs/feasibility_study.md` の「推奨ロードマップ」を現在地として扱うこと。
- `.claude/handover/` は端末ごとの情報を含むため git の管理対象外（.gitignore 済み）。
- セッションを中断するときは、同じフォルダにその日の日付で handover を作成（同じ日なら追記）し、途中経過・決めた方針・次にやることを書くこと。
