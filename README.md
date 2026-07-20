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
├── v2/                  schemaVersion=2 の配信データ（index.json / matches/ / highlights/ / SCHEMA.md）
├── pdf/                 元ネタの公式ランニングスコア PDF（**.gitignore 済み**、ローカル専用）
└── pdf-matches/         PDF から自動抽出した中間 JSON（**.gitignore 済み**、ローカル専用の staging）
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

出力先は **`pdf-matches/` (gitignore 済み)** で、ここは配信前の staging。ここには**実名のロースターが入る**ので、そのまま `v2/` へコピーしてはいけない。内容を確認 → OK と判断したら、親リポの [`tools/promote-sample-matches/`](../../tools/promote-sample-matches/) で昇格する（選手名の置き換え・`v2/index.json` の更新・同一試合の二重配信検出をまとめて行う）:

```sh
(cd tools/promote-sample-matches && uv sync)
tools/promote-sample-matches/.venv/bin/python tools/promote-sample-matches/promote.py \
  --staging apps/handball-sample-matches/pdf-matches/jha \
  --dist apps/handball-sample-matches/v2 \
  --description "..." --dry-run
```

かつては `shotMissed` の時刻が合成（フッター総数からの逆算）で品質要レビューだったが、importer が現行 SAMPLE_DTO_V2 を直接出力するようになった時点（handball-project#54）で解消済み。現在の `shotMissed` は PDF に明示記録された 7m スロー失敗のみで、時刻も実記録。

昇格済みの `.timer` サンプル:

| slug | 件数 | 出所 | 選手名 |
|---|---|---|---|
| `2025-12-20-zeekstar-tokyo-vs-bravekings-kariya` / `2025-12-21-bluefalcon-vs-zeekstar-tokyo` | 2 | JHA 公式ランニングスコア（handball-project#53） | 実名 |
| `2025-12-1{7,8,9}-{m,w}NN` / `2025-12-2{0,1}-{m,w}NN` | 41 | 第77回日本選手権 公式ランニングスコア（handball-project#63） | **背番号のみ** |

後者の slug は元 PDF の試合番号（`m01`-`m22` = 男子 / `w01`-`w23` = 女子）をそのまま使っており、出所を追跡できる。第77回日本選手権の全 45 試合のうち、次の 2 件は昇格していない:

- `w17`（ブルーサクヤ鹿児島 vs HC名古屋）— **延長戦**。出力 DTO が 1800 秒ハーフ × 2 固定のため importer が明示的に fail する
- `m09`（大阪体育大学 vs 福井永平寺ブルーサンダー）— **元 PDF 側の欠落**。ランニングスコアの列が満杯で最後の 1 点が行として印字されておらず、集計側にのみ含まれるため検算が通らない

前者 2 件は公開済み `.video` サンプル（同一試合を動画から手記録したもの）と突き合わせ済みで、背番号別の得点内訳が全選手一致することを確認している。昇格前の検証は `handball-toolkit` の検証 CLI で行う:

```sh
cd ../handball-toolkit
cargo run -p handball-toolkit-cli -- validate ../handball-sample-matches/v2
```

親リポ (`handball-project/`) の root から実行する（`cd` して相対パスで叩かない）:

```sh
(cd tools/jhl-pdf-importer && uv sync)
tools/jhl-pdf-importer/.venv/bin/python tools/jhl-pdf-importer/parse_jhl_pdf.py \
  apps/handball-sample-matches/pdf/jhl/running_501M156.pdf \
  --out apps/handball-sample-matches/pdf-matches/jhl/2026-04-25-jeekstar-vs-corazon.json
```

JHA 主催試合の PDF は `tools/jha-pdf-importer/parse_jha_pdf.py` と `pdf/jha/` → `pdf-matches/jha/` を使う（CLI・出力スキーマは同一）。

詳細・オプション・既知の制約は [tools/jhl-pdf-importer/README.md](../../tools/jhl-pdf-importer/README.md) / [tools/jha-pdf-importer/README.md](../../tools/jha-pdf-importer/README.md) を参照。

Claude Code の `/import-pdf` skill 経由でも実行できる（リーグ判定・PDF からの抽出・検算まで対話で案内する）。

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

**関係者の本名を含むデータは push しない。**

プロ選手の実名は職業上の公開情報として扱い、JHL 所属チーム同士の試合では残している（`2025-12-20-zeekstar-tokyo-vs-bravekings-kariya` など 4 件）。それ以外は**選手名を背番号ラベル（`7番`）へ置き換えて配信する**。

置き換えても選手別スタッツは失われない。`playerKey` が `{home,away}_{背番号}` なので、得点・シュートの集計は背番号だけで完全に成立する。失われるのは表示名のみ。チーム名（`国分中央高校` 等）は組織名なのでそのまま残す。

> ⚠️ **これは仮名化であって匿名化ではない。** 背番号・チーム名・日付が分かれば、大会主催者が公開している公式記録 PDF を引くことで実名に戻せる。「配信データが実名を含まない」ことは満たすが「個人を特定できない」ことは満たさない、と理解したうえで運用する。

置き換えは親リポの [`tools/promote-sample-matches/`](../../tools/promote-sample-matches/) が昇格時に行い、実行時に「元の実名が公開 JSON に残っていないか」を全文照合で検査する。**手でコピーせずこのツールを通すこと。**

エクスポータはアプリ内のデータをそのまま JSON 化するため、アプリ由来の JSON を push する場合は内容を必ず目視確認すること。

## スキーマバージョン

現行: `schemaVersion: 1`。詳細は [SCHEMA.md](SCHEMA.md) 参照。

後方互換破壊変更時は `schemaVersion` を上げる。アプリ側は不一致を検出して当該試合をスキップする実装。
