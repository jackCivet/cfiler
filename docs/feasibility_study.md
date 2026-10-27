# cfiler（内骨格）改良・機能追加 Feasibility Study

- 作成日: 2026-10-27
- 対象: jackCivet/cfiler（crftwr/cfiler v2.62 の Fork、`master` = `0d1017e`）
- 範囲: ソースの静的調査だけ。ビルドも実行もしていない（調査に使った PC はまもなく使えなくなるため）

## 1. 結論（要約）

| 項目 | 判定 | 根拠 |
|---|---|---|
| 現状のまま（Python 3.8）ビルドを再現する | △ 可能。ただし手間あり | ckit / pyauto の最新版は既に Python 3.13 用。3.8 で作るには 2023-11 時点のコミットに戻す必要がある |
| Python 3.13 への移植 | ◎ 現実的 | ckit / pyauto は本家が 3.13 へ移植済み。cfiler 側で直す C API は 2 ファイル・6 か所だけ |
| Python だけ改良する（C++ ビルドなし） | ○ 有力（未検証） | 配布版は `extension/` を `library.zip` より先に読み込む。改変した `.py` を置けば差し替えられる見込み |
| 機能追加 | ○ | アプリの本体は Python（約 13,200 行）。C++ は約 1,800 行で、ほぼ起動処理とネイティブ補助だけ |

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
