# handball-sample-matches

iOS アプリ「ハンド記録」で配信するサンプル試合データの公開リポジトリ。

アプリは起動時にこの repo の `index.json` を取得 → 各試合本体 (`matches/{slug}.json`) を取得して、ユーザーの自分の試合と並べて「サンプル試合」として表示する。サンプル試合は端末側に永続化されない（毎回 fetch）。

## ディレクトリ構成

```
.
├── README.md            この repo の説明
├── SCHEMA.md            JSON スキーマ仕様（schemaVersion=1）
├── index.json           試合一覧の軽量メタデータ（必須）
├── matches/             試合本体 JSON（必須、アプリに配信される）
│   └── {slug}.json      1 試合 = 1 ファイル
├── highlights/          ハイライト集（試合とは別配信、専用 index 持ち）
│   ├── index.json       ハイライト一覧の軽量メタデータ
│   └── {slug}.json      1 ハイライト = 1 ファイル
├── pdf/                 元ネタの JHL 公式ランニングスコア PDF（**.gitignore 済み**、ローカル専用）
└── pdf-matches/         PDF から自動抽出した中間 JSON（**.gitignore 済み**、ローカル専用）
```

PDF → JSON 変換スクリプトは親リポの [`tools/jhl-pdf-importer/`](../../tools/jhl-pdf-importer/) に置いている。

`index.json` と `matches/{slug}.json` のパスはアプリ側 (`SampleMatchLoader`) で固定。**変えると読めなくなる**。

## URL

- 試合一覧: `https://raw.githubusercontent.com/kinjo-ryura/handball-sample-matches/main/index.json`
- 試合本体: `https://raw.githubusercontent.com/kinjo-ryura/handball-sample-matches/main/matches/{slug}.json`
- ハイライト一覧: `https://raw.githubusercontent.com/kinjo-ryura/handball-sample-matches/main/highlights/index.json`
- ハイライト本体: `https://raw.githubusercontent.com/kinjo-ryura/handball-sample-matches/main/highlights/{slug}.json`

`raw.githubusercontent.com` は HTTPS（ATS 対応）+ Fastly CDN（デフォルト約 5 分キャッシュ）。

## 試合の追加手順

### 方法 A: アプリエクスポートから

1. **試合本体 JSON を作成して `matches/{slug}.json` に置く**
   - ハンド記録アプリの DEBUG ビルドで「データ」タブ → 試合詳細 → 共有ボタン → JSON を書き出すのが最短
   - エクスポータが出力する slug は `{yyyy-MM-dd}-{home}-vs-{away}` 形式（日本語チーム名は `{date}-{8桁hex}` フォールバック）。意味のあるファイル名にリネーム推奨
2. **`index.json` の `matches` 配列に summary を 1 件追加**
   - `slug` はファイル名（拡張子除く）と完全一致させる
   - `homeScore` / `awayScore` / `hasYouTubeURL` は試合本体から手動で転記（軽量メタとして index 単独で表示するため重複を許容）
   - 配列は date 降順（新しい試合が上）で維持する
3. commit & push

### 方法 B: JHL 公式 PDF からの自動生成（タイマーモード、ローカル専用）

JHL公式ランニングスコア PDF（`pdf/{試合コード}.pdf`）から、両チームのロースター・全得点・選手別シュート総数を抽出して JSON 化する。動画タイムスタンプは含まれない（タイマーモード前提）。

出力先は **`pdf-matches/` (gitignore 済み)** で、当面は配信対象外。`shotMissed` イベントの時刻が合成（フッター総数からの逆算）になっており、品質要レビューのため。手動で内容を確認 → OK と判断したら `matches/` に手動コピーして `index.json` に追加することで配信対象にできる。

```sh
cd ../../tools/jhl-pdf-importer    # 親リポのツール置き場へ
python3 -m venv .venv && .venv/bin/pip install -r requirements.txt
.venv/bin/python parse_jhl_pdf.py \
  ../../apps/handball-sample-matches/pdf/running_501M156.pdf \
  --out ../../apps/handball-sample-matches/pdf-matches/2026-04-25-jeekstar-vs-corazon.json
```

詳細・オプション・既知の制約は [tools/jhl-pdf-importer/README.md](../../tools/jhl-pdf-importer/README.md) を参照。

Claude Code の `/import-jhl-pdf` skill 経由でも実行できる（PDF からの抽出・検算まで対話で案内する）。

## ハイライトの追加手順

1. **本体 JSON を作成して `highlights/{slug}.json` に置く**
   - HandballRecorder のハイライトモードで作成 → DEBUG エクスポートが最短（schema は `matches/{slug}.json` と共通）
   - schema 詳細は [SCHEMA.md](SCHEMA.md) の `highlights/{slug}.json` 節を参照
2. **`highlights/index.json` の `highlights` 配列に summary を 1 件追加**
   - `slug` はファイル名（拡張子除く）と完全一致させる
   - `homeTeamName` / `awayTeamName` / `eventCount` / `hasYouTubeURL` は本体から手動転記
   - 配列は date 降順で維持
3. commit & push

## 命名ルール

- slug は **ASCII 小文字 + 数字 + ハイフン** に限定（URL/ファイルシステム両対応）
- `{yyyy-MM-dd}-{home}-vs-{away}` を基本形式に
- 同日同対戦の重複は `-game2` / `-final` のような接尾辞で回避

## 個人情報の扱い

公開対象は **プロ試合（公開情報のみ）** に限定する方針。アマチュア試合・関係者の本名等を含むデータは push しない。エクスポータはアプリ内のデータをそのまま JSON 化するため、push 前に内容を必ず目視確認すること。

## スキーマバージョン

現行: `schemaVersion: 1`。詳細は [SCHEMA.md](SCHEMA.md) 参照。

後方互換破壊変更時は `schemaVersion` を上げる。アプリ側は不一致を検出して当該試合をスキップする実装。
