# cfiler（内骨格）改良・機能追加 Feasibility Study

- 作成日: 2026-10-27
- 対象: jackCivet/cfiler（crftwr/cfiler v2.62 の Fork、`master` = `0d1017e`）
- 範囲: ソースの静的調査だけ。ビルドも実行もしていない（調査に使った PC はまもなく使えなくなるため）
- 構成: §1〜8 は土台（ビルドと Python のモダン化）。§9 以降は機能追加の調査（1 件ずつ追記していく）
  - §9 機能 1: VS Code の Terminal を下ペインに移植する（チャットでの議論を反映済み）
  - §10 ネイティブ（Rust）化による高速化の候補（候補を挙げただけ。優先順位は未定）

## 1. 結論（要約）

| 項目 | 判定 | 根拠 |
|---|---|---|
| 現状のまま（Python 3.8）ビルドを再現する | △ 可能。ただし手間あり | ckit / pyauto の最新版は既に Python 3.13 用。3.8 で作るには 2023-11 時点のコミットに戻す必要がある |
| Python 3.13 への移植 | ◎ 現実的 | ckit / pyauto は本家が 3.13 へ移植済み。cfiler 側で直す C API は 2 ファイル・6 か所だけ |
| Python だけ改良する（C++ ビルドなし） | ○ 有力（未検証） | 配布版は `extension/` を `library.zip` より先に読み込む。改変した `.py` を置けば差し替えられる見込み |
| 機能追加 | ○ | アプリの本体は Python（約 13,200 行）。C++ は約 1,800 行で、ほぼ起動処理とネイティブ補助だけ |
| 機能 1: VS Code の Terminal を下ペインに移植 | ○ 機能としては実現できる。コードをそのまま移植するのは不可 | ConPTY + Python 製の VT エミュレーター + ckit の描画で、ポータブルなまま実現できる。VS Code からはシェル統合スクリプトとロジックを流用する（§9） |
| Rust による高速化 | ○ 部分的に置き換えるなら有効 | ckit を全部書き換えても速くならない。端末の解析、Grep、差分、文字コード判定に絞ると効果が大きい（§10） |

**推奨**: 目標の Python は **3.13** にする（ckit / pyauto の本家と揃うため）。3.8 での基準ビルドは行わず、cfiler 本体を 3.13 に移植してから機能追加に入る。

## 2. 現状

- cfiler v2.62（最終リリース 2021-06-07）。最終コミットは 2023-11-05。ライセンスは MIT。本家の open issue は 43 件
- 構成
  - `cfiler.exe`（`main.cpp`）: Python を組み込んで起動する C++ の起動プログラム
  - `cfiler_native`: C++ の拡張モジュール（15 ファイル）
  - `cfiler_*.py`: アプリの本体
  - 依存: **ckit**（UI フレームワーク。C++ と Python）、**pyauto**（入力フック。C++ と Python）、Pillow、**comtypes**（readme には書かれていない）、同梱の DLL（7-zip64 / tar64 / migemo）
- ビルド設定（`.vcxproj`）
  - PlatformToolset は `v140`（VS2015）、Windows SDK は `10.0.18362.0`。readme には「VS2019」と書かれているが、実際はリターゲットが必要
  - Python のパス `c:\Python38-64` が直接書かれている場所: `main.cpp:16`、`cfiler.vcxproj`、`cfiler_native\cfiler_native.vcxproj`、`makefile.py:32`

## 3. 依存リポジトリの状況（2025-02 に本家で Python 3.13 へ移植済み）

| リポジトリ | 3.13 移植のコミット | 内容 |
|---|---|---|
| crftwr/ckit | `92a53d2`（2025-02-15）、`16de5c1` | `PyEval_CallObject` を `PyObject_Call` に置き換え、`PyUnicode_AS_UNICODE` を `PyUnicode_AsWideCharString` に置き換え、パスを `c:\Python313` に変更、無効なエスケープ文字列を修正 |
| crftwr/pyauto | `a848225`（2025-02-15） | 3.13 対応と 64bit への一本化（32bit 構成を削除） |

- 3.8 で基準ビルドをするには、ckit を `3c7a130`、pyauto を `875b030`（どちらも 2023-11-05）に固定する必要がある
- 同じ作者の keyhac は 2026-10 時点でも活発に開発されていて、pyproject と venv を使ったビルド検証を導入している。モダン化の参考になる
- **注意（ckit 本家のバグ候補）**: `ckitcore/pythonutil.cpp` で、`PyUnicode_AsWideCharString` が返したバッファを `PyMem_Free` で解放していない（メモリリーク）。Fork したら修正する

## 4. cfiler 本体を Python 3.13 へ移植する作業

### C/C++（必須）
| 場所 | 問題 | 対応 |
|---|---|---|
| `cfiler_native/pythonutil.cpp:8,23` | `PyUnicode_AS_UNICODE` / `PyUnicode_GET_SIZE` は 3.12 で削除された | `PyUnicode_AsWideCharString` に置き換え、`PyMem_Free` で解放する（ckit の実装を流用し、リークは直す） |
| `main.cpp:68-104` | `Py_SetPythonHome` / `Py_SetPath` / `PySys_SetArgv` は 3.11 から非推奨 | `PyConfig`（`home`、`module_search_paths`、`argv`）と `Py_InitializeFromConfig` に書き換える |
| `main.cpp:16`、各 `.vcxproj`、`makefile.py` | `Python38` が直接書かれている | `Python313` にするか、プロパティシートや環境変数で指定できるようにする |
| 各 `.vcxproj` | ツールセットが v140 / SDK が 18362 | 導入する VS / Build Tools のバージョンにリターゲットする |

`cfiler_native` では `PyEval_CallObject` は使われていない（ckit 側で既に対応済み）。

### Python 側
- `cfiler_diffviewer.py:135`: 無効なエスケープ文字列 `\+`（3.12 以降は SyntaxWarning）。raw 文字列にする
- `cfiler_networkplace.py:5`: **Python 2 の `import urlparse` が残っている** → 「ネットワークプレース」機能は Python 3 では読み込んだ時点で ImportError になる（`cfiler_mainwindow.py:6309` で必要になったときに import している）。`urllib.parse` に直せば、既存の不具合の修正にもなる
- ckit `ckit_resource.py:10`: `locale.getdefaultlocale()` は 3.11 から非推奨
- 3.12 以降で削除された標準モジュール（imp、distutils、asyncore など）は使われていない
- comtypes は 3.13 に対応した版を入れる必要がある

**作業量の見込み**: C++ の修正は 1〜2 日、ビルド環境の構築と動作確認が 2〜3 日（VS を用意できることが前提）。

## 5. ビルドなしで Python だけ改良する案（未検証）

`main.cpp` のリリースビルドは、モジュールの検索パスを `extension;library.zip;lib` の順にしている。
そのため、配布版 v2.62（`cfiler_262.zip`、Python 3.8 同梱）の `extension/` に改変した `cfiler_*.py` を置けば、`library.zip` 内のモジュールより優先して読み込まれる見込み。

- 利点: Visual Studio が要らない。今すぐ機能追加を試せる
- 欠点: Python 3.8 のまま。ネイティブ部分は変えられない。モジュールの読み込み順の副作用がないか、検証が必要
- 用途: 機能のプロトタイプ作りや、ネイティブ移植と並行して進める作業向き

## 6. 開発環境の方針

- Python は **uv** で複数のバージョンを管理する（`uv python install 3.13`）。既存のシステムの Python には影響しない
  - C++ のビルドに必要な `include\Python.h`、`libs\python313.lib`、`python313.dll` が uv の Python に含まれているか確認する。無ければ python.org のインストーラーに切り替える
  - `c:\Python313` へのジャンクションか、`.vcxproj` 側でパスを指定する方法で参照させる
- Visual Studio 2022 Build Tools（C++ のワークロード）が必要
- ckit / pyauto も Fork し、`C:\Project\{cfiler,ckit,pyauto}` のように同じ階層に並べる
- Git のコミット者は、リポジトリごとに個人のアカウントを設定する（`--global` は変更しない）

## 7. 推奨ロードマップ

1. ckit と pyauto を Fork する（本家の 3.13 版をそのまま使う。ckit のリークを修正する）
2. Python 3.13（uv）と VS Build Tools を用意し、ckit と pyauto をビルドする
3. cfiler を 3.13 へ移植する（§4）。ブランチは `port/python313`
4. 起動確認とひととおりの操作確認をして、リリースパッケージ（`makefile.py`）を作る
5. 既存の不具合（ネットワークプレースなど）を修正し、本家の open issue から改良候補を選ぶ
   - 例: #69 フォルダブックマークのジャンプ、#61 多重起動の防止、#53 ジャンクションやシンボリックリンクの表示
6. 必要に応じて、keyhac にならって pyproject と依存バージョンの固定を導入する

## 8. リスクと未確認事項

- 社内 PC などでは、VS Build Tools をポリシーで入れられない可能性がある（winget や Store が使えない環境では MSI やオフラインインストーラーで導入する）
- uv の Python で C 拡張を作れるか（`libs\*.lib` が入っているか）は未確認
- §5 の `extension/` で差し替える案は未検証
- comtypes、Pillow、migemo.dll などと Python 3.13 の組み合わせで実際に動くかは未確認
- 本家は 2023 年以降更新されていないため、今後 upstream から更新を取り込むことはあまり期待できない（Fork 側で独自に保守する前提）

---

## 9. 機能 1: VS Code の Terminal を cfiler の下ペインに移植する

### 9.1 要件
- VS Code の統合ターミナル（パネルの Terminal、プロファイルメニュー）相当の機能を、cfiler の下ペイン（現在のログペイン）で使えるようにする
- 移植したコードも cfiler の中に含め、**インストーラー無しでフォルダごと引っ越せる**ことを守る（VS Code の最新版とずれることは承知の上）
- 細かい仕様は、この Feasibility Study の後で決める

### 9.2 結論と、議論で決めた方針
| # | 論点 | 結論・方針 |
|---|---|---|
| 1 | 実現方式 | **方式 A（ConPTY + VT エミュレーター + ckit で描画）で確定**。VS Code のコードをそのまま移植するのは不可（TypeScript と Electron/DOM の前提）。流用するのは、シェル統合スクリプト（MIT、そのまま同梱）、プロファイル検出などのロジック（Python に書き直す）、メニューとキー操作の仕様 |
| 2 | Log の扱い | **Log を cmd / PowerShell / Ubuntu bash と同列に並べ、タブで切り替える**。Log は入力を受け付けないターミナルとして扱う |
| 3 | フォーカスとキー | **cfiler の作法どおり config.py に書く**。ただし書くのは**タブの移動とペインの移動だけ**で、それ以外のキーはすべて各ターミナル（シェル）に送る |
| 4 | Ghostty（WSL2 側） | タブの中には入れられない（Ghostty は「シェル」ではなく cfiler と同じ層の「ターミナルエミュレーター」。Windows 版は無く、WSLg の別ウィンドウになり埋め込めない）。**外部ウィンドウとして起動するコマンド**を config.py に追加する形なら可能 |
| 5 | Rust の使いどころ | ckit を全部書き換えても速くならない。**端末の解析部分（§9.6）だけを Rust などのネイティブコードにする**のが筋が良い。cfiler のほかの機能の候補は §10 |

### 9.3 VS Code のターミナルの構成（microsoft/vscode、MIT）
| 層 | VS Code の実装 | 場所 |
|---|---|---|
| 擬似コンソール（PTY） | **node-pty**（Node のネイティブモジュール。Windows では ConPTY を使う） | `src/vs/platform/terminal/node/terminalProcess.ts`、`ptyService.ts` |
| プロファイル（PowerShell / Git Bash / cmd / WSL）の検出 | TypeScript | `src/vs/platform/terminal/node/terminalProfiles.ts` |
| 環境変数・シェルの起動設定 | TypeScript | `src/vs/platform/terminal/node/terminalEnvironment.ts` |
| シェル統合（現在のディレクトリ、コマンドの境界、終了コード） | PowerShell / bash / zsh / fish のスクリプトと OSC 633 シーケンス | `src/vs/workbench/contrib/terminal/common/scripts/shellIntegration.ps1` など |
| VT の解析・画面バッファ・描画 | **xterm.js**（`@xterm/xterm` 6.x、MIT。DOM / WebGL で描画） | 別リポジトリ xtermjs/xterm.js |
| UI（タブ、分割、メニュー、操作） | TypeScript | `src/vs/workbench/contrib/terminal/browser/`（terminalInstance.ts、terminalGroup.ts、terminalMenus.ts など） |

→ 画面上のターミナルの中身は、ほぼ **xterm.js + node-pty** でできている。どちらもブラウザエンジンや Node.js の実行環境が前提なので、cfiler（Python を組み込んだ Win32 アプリ）にそのまま組み込むことはできない。

### 9.4 実現方式の比較（A に決定）
| | **A. Python ネイティブ実装（採用）** | B. WebView2 + xterm.js を埋め込む | C. 外部のコンソールウィンドウを埋め込む |
|---|---|---|---|
| 概要 | ConPTY（Win32 API）を ctypes で呼ぶ。VT エミュレーターで画面を作り、ckit で描画する | 下ペインに WebView2 の子ウィンドウを置き、xterm.js を動かす | conhost や Windows Terminal のウィンドウを SetParent で取り込む |
| VS Code との見た目の一致 | △ cfiler の見た目に合わせる | ◎ 同じ描画エンジン | × |
| ポータブル性 | ◎ 追加はすべて lib に同梱するだけ | ○ WebView2 ランタイムは Win11 に標準搭載。ホストに C++ か pythonnet が必要 | × |
| cfiler との統合 | ◎ 既存の描画とキー処理に乗る | △ 子 HWND とのフォーカスやキーの受け渡しが難しい | × |
| メモリ | 小 | +100MB 程度 | 小 |

### 9.5 下ペインの設計（方針 2 と 3 の具体化）
```
┌──────────── 左のファイル一覧 ───────────┬──────────── 右のファイル一覧 ───────────┐
│                                          │                                          │
├──────────────────────────────────────────┴──────────────────────────────────────────┤
│ [ Log ] [ PowerShell ] [ cmd ] [ Ubuntu ]                       ← 下ペインのタブ     │
│ PS C:\Project\cfiler> _                                                              │
└──────────────────────────────────────────────────────────────────────────────────────┘
```
- **タブは共通の仕組みで扱う**（描画、スクロール、選択、入力）。Log タブは今のログペインの描画・選択処理をそのまま移し、入力は受け付けない
- `sys.stdout` と `sys.stderr` は今まで通り Log タブへ流す。**別のタブを見ている間に Log に出力があったら、タブに印を付けて知らせる**
- **キーの振り分け**
  - config.py に、ターミナル用のキー割り当て（例: `window.terminal_keymap`）を新しく設ける。既存の `window.keymap[...] = command_xxx` と同じ書き方にする
  - ターミナルにフォーカスがあるとき: `terminal_keymap` にあるキー（タブの移動、ペインの移動）は cfiler が処理し、**それ以外はすべて PTY に送る**
  - 注意: 切り替え用に割り当てたキーは、シェルに届かなくなる（Ctrl+W や Alt+矢印はシェルや vim とぶつかりやすい。Ctrl+Tab などが無難）
- **cfiler・ckit 側の受け皿（コードで確認済み）**

| 必要なもの | 現状 | 評価 |
|---|---|---|
| 下ペインの領域 | `cfiler_mainwindow.py` の `log_pane`、`logPaneRect()`（:1385）、高さ `log_window_height`（ini に保存。サイズ変更のコマンドもある :2987〜） | ◎ |
| 文字グリッドの描画 | `_paintLog()` が `putString` と `ckit.Attribute(fg, bg)` で 1 行ずつ描画している。色は RGB 指定 → 256 色やトゥルーカラーも表せる | ◎（太字や斜体は Attribute に無い） |
| カーソル・IME | ckit の `setCaretPosition`、`enableIme`、`setImeRect` | ◎ |
| 再描画の間引き | ckit の `setTimer` | ◎ |
| 選択とコピー | ログペインの選択処理（:1020〜、:1287〜） | ○ 流用できる |
| フォーカス | 今は `FOCUS_LEFT` と `FOCUS_RIGHT` だけ | △ ターミナル用のフォーカスを追加する |
| 子プロセス | 既存の `ckit.SubProcess` はパイプで 1 行ずつ読むだけ | 新しく作る |

### 9.6 「端末の解析部分」とネイティブ化の候補
シェルの出力が画面に出るまでの流れ:
```
ConPTY の出力(バイト列)
  │ ① UTF-8 デコード
  │ ② VT パーサー（状態機械）  … 1 文字ずつ「文字か、制御か、エスケープシーケンスの途中か」を判定
  │      例: ESC[31m → 前景色を赤に / ESC[2J → 画面を消去 / OSC 7 → 現在のディレクトリの通知
  │ ③ 端末の状態モデル  … セルのグリッド（文字、幅、色）、カーソル、行末での折り返し、スクロール領域、
  │                        代替画面（vim など）、スクロールバック、全角と半角の幅、変化した行の記録
  │ ④ 描画（変化した行だけ ckit の putString で描く）  ← Python と ckit に残す
（逆方向）キー入力 → エスケープシーケンスに変換 → ConPTY  ← 軽い処理なので Python に残す
```
- ネイティブ化するのは **② と ③ をまとめて**。② だけだと、解析した結果を 1 件ずつ Python に返すため、受け渡しの負荷で速さを打ち消してしまう
- Python との接口は「変化した行を、色ごとの区切り付きでまとめて受け取る」形に決めておき、中身を差し替えられるようにする

| 候補 | 言語 | ライセンス | 範囲 | 特徴 / 組み込み方 |
|---|---|---|---|---|
| pyte（基準・PoC 用） | Python | LGPL-3.0 | ②③ | すぐ使える。大量の出力では遅い可能性がある |
| **alacritty_terminal** | Rust | Apache-2.0 | ②③（選択、検索、スクロールバックまで） | Alacritty の中核で実績が豊富。PyO3 と maturin で `.pyd` にする。外部向けに安定した API ではないので、バージョンを固定して使う |
| vte | Rust | Apache-2.0 / MIT | ② だけ | ③ は自分で書く |
| wezterm-term / termwiz | Rust | MIT | ②③ | 機能は最も豊富。crates.io での公開状況と API の安定性は要確認 |
| **libvterm** | C | MIT | ②③ | Neovim や Vim で使われていて枯れている。**既存の cfiler_native（C++）に組み込め、Rust の開発環境が要らない** |
| libghostty-vt | Zig（C から呼べる） | MIT | ②③ | Windows 対応と成熟度は未確認 |

- 進め方: **pyte で作る → 計測する → 遅ければ alacritty_terminal（Rust）か libvterm（C）に差し替える**
- alacritty_terminal と libvterm のどちらにするかは、Rust を採用する目的（高速化だけか、将来ほかの部分も Rust にしていくか）で決める（未決定）

### 9.7 VS Code の機能（スクリーンショットのメニュー）ごとの実現性
| VS Code の機能 | 判定 | メモ |
|---|---|---|
| New Terminal | ◎ | MVP |
| PowerShell / Command Prompt / Git Bash / WSL（Ubuntu）のプロファイル | ◎ | `terminalProfiles.ts` の検出ロジックを Python に移植する（既知のパス、レジストリ、`wsl.exe -l`） |
| Select Default Profile / Configure Terminal Settings | ○ | cfiler の ini や config.py に保存する |
| 複数のターミナル（タブ） | ○ | §9.5 のタブ |
| Split Terminal | ○ | 下ペインを左右に分割する |
| New Terminal Window | △ | ckit で別ウィンドウを作る。優先度は低い |
| Run Task / Configure Tasks | △ | VS Code の tasks.json に相当する仕組みが必要で、別の機能として扱う |
| JavaScript Debug Terminal / Agent Host / Copilot のヒント | × | VS Code の拡張機能やサービスに依存するので対象外 |
| Ghostty など外部のターミナル | ○（外部起動） | config.py にコマンドを追加し、`wsl.exe -d Ubuntu -- ghostty --working-directory=<WSL のパス>` を実行する（パスは `wslpath` で変換） |
| **cfiler ならではの連携** | ◎ | シェル統合で cwd を受け取り、**ファイル一覧とターミナルの現在のディレクトリを双方向に同期**する。選択したファイルのパスをターミナルに貼り付ける |

### 9.8 ポータブル性とライセンス
- 方式 A で追加するのは、Python のコード、シェルスクリプト、（ネイティブ化する場合は）1 つの `.pyd` だけ。ConPTY は OS に含まれている → **インストーラー不要の性質を保てる**
- VS Code と xterm.js（どちらも MIT）から移植・同梱するコードは、著作権表示とライセンス文を `license/` に追加する
- pyte（LGPL-3.0）は、差し替えられるよう `.py` のまま `lib/` に置き、ライセンス文を同梱する
- Rust の部品（Apache-2.0 / MIT）は、NOTICE とライセンス文を同梱する

### 9.9 未決事項（仕様を決めるときに詰めること）
- 起動時に開いておくタブ（例: Log と既定のプロファイル 1 つ）
- 同じプロファイルを複数開けるか（VS Code 方式）、プロファイルごとに 1 つか
- タブを閉じたときや cfiler を終了したときに、動いているシェルをどう扱うか
- タブとペインの移動に割り当てる具体的なキー
- 選択中の Ctrl+C を、コピーとシグナル送信のどちらにするか（VS Code は選択があればコピー）
- 色テーマ（cfiler のテーマに合わせるか、VS Code に寄せるか）

### 9.10 リスク
- **性能**: 純 Python の VT エミュレーターは大量の出力で遅い可能性がある → 描画をタイマーで間引く。足りなければ §9.6 のネイティブ化
- **キーの衝突**: cfiler の keymap、ckit の IME、シェルに送るキーの振り分け
- **文字幅のずれ**: 罫線や絵文字などの曖昧な幅の文字で、wcwidth と ckit の `getStringWidth` の判定が食い違い、カーソル位置がずれる可能性がある
- **ConPTY**: Windows 10 1809 以降が必要
- **Python 3.8 でのプロトタイプ**: ctypes と pyte は 3.8 でも動く見込み。§5 の方法で、ビルド前に配布版の上で PoC ができる可能性がある（未検証）
- 参考（別件）: 提供されたスクリーンショットで、`config.py` の `configure_ImageViewer`（`window.keymap[VK_F1]`）が `TypeError` を出している。ユーザーの設定ファイルと本体のバージョンが合っていない可能性がある

### 9.11 進め方（案）
1. **PoC-1（cfiler の外）**: ConPTY（ctypes）+ pyte で pwsh を動かす。日本語、色、リサイズ、Ctrl+C を確認する
2. **PoC-2（cfiler の中）**: 下ペインに描画し、フォーカスの切り替え、キーの振り分け、IME を確認する
3. **MVP**: Log とターミナルのタブ、プロファイル（PowerShell / cmd / Git Bash / Ubuntu）、スクロールバック、コピー＆ペースト、設定の保存
4. **拡張**: 複数のターミナルや分割、シェル統合による cwd とファイル一覧の連携、外部ターミナル（Ghostty など）の起動
5. **計測してから高速化**: 必要なら §9.6 のネイティブ化
- 作業量の見込み（§7 の土台が終わっている前提）: PoC-1 は 1〜2 日、PoC-2 は 2〜3 日、MVP は 1〜2 週間、拡張は 1〜2 週間

---

## 10. ネイティブ（Rust）化による高速化の候補（ターミナル以外）

> 今の段階では候補を挙げるだけにとどめる。優先順位と着手するかどうかは、実際に計測してから決める。

### 10.1 前提: ckit を全部 Rust で書き換えても速くならない
- ckit の C++ 部分（ckitcore.cpp、約 8,000 行）がやっているのは、ウィンドウの管理、GDI での描画、IME、メッセージループ。Rust にしても同じ Win32 API を呼ぶので、処理時間はほぼ変わらない
- 描画が遅い場合に効くのは、言語の変更ではなく描画方式の見直し（GDI から DirectWrite / Direct2D へ）
- 全部書き換える利点はメモリ安全性と保守のしやすさ。費用は約 8,000 行の書き直しと、Python との接続部分（PyO3）の作り直し
- 遅くなりそうなのは **Python のループで 1 件ずつ処理している部分**。そこだけを狙う

### 10.2 候補（cfiler と ckit のコードで確認）
| 優先度（仮） | 機能 | 今の実装 | 遅い理由 | 置き換え先 | 期待できる効果 |
|---|---|---|---|---|---|
| ★★★ | **Grep**（`cfiler_grep.py`） | ファイルを丸ごと読み、文字コード判定、デコード、`splitlines`、1 行ずつ `re` で照合。すべて Python | 1 ファイルごと・1 行ごとの処理が Python。1 ファイルずつ順番に処理 | ripgrep の部品（`grep-searcher`、`grep-regex`、`ignore`。MIT / Unlicense）＋ `encoding_rs`（cp932、EUC-JP、ISO-2022-JP） | 大きなフォルダで 10〜100 倍。複数の CPU コアで並列に検索 |
| ★★★ | **差分表示**（`cfiler_diffviewer.py:131`） | `difflib.unified_diff`（純 Python） | 処理量が最悪で行数の 2 乗に比例し、大きなファイルで固まる | `similar`（Apache-2.0。Myers 法や Patience 法） | 大きなファイルでも実用的になる。行の中の差分も表示しやすい |
| ★★ | **文字コード判定**（ckit の `ckit_misc.detectTextEncoding`） | 最大 1000 行について、5 種類の文字コードで実際にデコードしてみて採点する | テキストビューア、差分、ファイル比較、**Grep では 1 ファイルごと**に呼ばれる | `encoding_rs` ＋ `chardetng`（Firefox の文字コード判定） | 速くなり、判定の精度も上がる。Grep と部品を共有できる |
| ★ | ファイル内容の比較・ハッシュ（`cfiler_filecmp.py`、`cfiler_mainwindow.py:6950` の MD5） | Python のループ + hashlib（C で実装） | ほとんどがディスクの読み書き待ち | 並列に読む、xxh3 / BLAKE3 | SSD で複数のファイルを比べるときに効く程度 |

### 10.3 Rust にしなくてよいもの
| 機能 | 理由 / 代わりの方法 |
|---|---|
| ファイルコピー（`cfiler_mainwindow.py:3530`。1MB ずつ読んで書く Python のループ） | 効果を出すなら Rust より **Win32 の `CopyFile2` / `CopyFileEx`**（OS の高速な経路、SMB でのサーバー側コピー、属性の保持、進捗の通知）。cfiler_native や ctypes から呼べる。Rust とは別の課題として扱う |
| フォルダの一覧（`cfiler_native.findFile`） | 既に C++ |
| 並べ替え、ワイルドカードでの絞り込み、インクリメンタルサーチ | 扱う件数が少ない。フォルダに 10 万件以上のファイルがあるような極端な場合だけ問題になる |
| Migemo、画像ビューア、書庫 | 既に DLL（migemo、Pillow、7-zip）に任せている |

### 10.4 方針（案）
- Rust のモジュールは、機能ごとに分けず **1 つの `.pyd`（仮の名前は `cfiler_rs`）にまとめる**（端末、Grep、差分、文字コード判定）。ポータブル性を保て、ビルドも 1 回で済む
- **Rust 側で GIL を手放して処理する** → 速くなるだけでなく、Grep などのバックグラウンド処理中も画面の操作がなめらかになる（今は Python のスレッドが GIL を握るため、処理中は画面が重くなる）
- 順番の候補: 端末 → Grep と文字コード判定 → 差分。ただし**計測してから着手する**
- 開発環境には Rust のツールチェーンと maturin が増える（新しい PC で uv と一緒に用意する）
