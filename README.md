# handball-sample-matches

iOS アプリ「ハンド記録」で配信するサンプル試合データの公開リポジトリ。

アプリは起動時にこの repo の `v2/index.json` を取得 → 各試合本体 (`v2/matches/{slug}.json`) を取得して、ユーザーの自分の試合と並べて「サンプル試合」として表示する。サンプル試合は端末側に永続化されない（毎回 fetch）。

## ディレクトリ構成

```
.
├── README.md            この repo の説明
├── v2/                  配信データ（schemaVersion=2。現行かつ唯一の配信経路）
│   ├── SCHEMA.md        JSON スキーマ仕様（schemaVersion=2）
│   ├── index.json       試合一覧の軽量メタデータ（必須）
│   ├── matches/         試合本体 JSON（必須、アプリに配信される）
│   │   └── {slug}.json  1 試合 = 1 ファイル
│   └── highlights/      ハイライト集（試合とは別配信、専用 index 持ち）
│       ├── index.json   ハイライト一覧の軽量メタデータ
│       └── {slug}.json  1 ハイライト = 1 ファイル
├── migration-snapshots/ V1 → V2 移行時の突合用スナップショット（配信対象外）
├── pdf/                 元ネタの公式ランニングスコア PDF（**.gitignore 済み**、ローカル専用）
└── pdf-matches/         PDF から自動抽出した中間 JSON（**.gitignore 済み**、ローカル専用の staging）
```

PDF → JSON 変換スクリプトは親リポの [`tools/jhl-pdf-importer/`](../../tools/jhl-pdf-importer/) に置いている。

`v2/index.json` と `v2/matches/{slug}.json` のパスはアプリ側 (`SampleMatchLoaderV2`) で固定。**変えると読めなくなる**。

## URL

- 試合一覧: `https://raw.githubusercontent.com/kinjo-ryura/handball-sample-matches/main/v2/index.json`
- 試合本体: `https://raw.githubusercontent.com/kinjo-ryura/handball-sample-matches/main/v2/matches/{slug}.json`
- ハイライト一覧: `https://raw.githubusercontent.com/kinjo-ryura/handball-sample-matches/main/v2/highlights/index.json`
- ハイライト本体: `https://raw.githubusercontent.com/kinjo-ryura/handball-sample-matches/main/v2/highlights/{slug}.json`

`raw.githubusercontent.com` は HTTPS（ATS 対応）+ Fastly CDN（デフォルト約 5 分キャッシュ）。

## V1 配信（root の `index.json` / `matches/` / `highlights/`）は廃止済み

**2026-07-26 に repo から削除した。** schemaVersion=1 の配信経路はもう存在しない。

- **経緯**: アプリ側は 2026-05-28 の V2 cutover（`HandballRecorder` の `af6eae2`）で V1 loader を削除済みで、以降 V1 は誰も更新していなかった。最終更新 2026-05-10 の時点で V1 は 2 試合 / 6 ハイライト、V2 は 45 試合。差分を埋め続けるコストに見合う利用者がおらず、**メンテナンス負荷を理由に凍結ではなく削除を選んだ**（[handball-project#116](https://github.com/kinjo-ryura/handball-project/issues/116)）。
- **影響範囲**: V1 を読むのは App Store 版 **1.0.1 (build 9, 2026-05-07 公開) 以前**のみ。現行 1.1.0（2026-07-11 公開）は V2 のみを読む。1.0.x のままのユーザーはサンプル試合の取得が 404 になり、一覧がエラー表示になる（V1 の `SampleMatchStore` は fetch 失敗を `.failed` 状態として扱うのでクラッシュはしない）。復旧手段はアプリの更新。
- **放置していた既知の不正確さ**: V1 の `2025-12-20-f352ea46`（ZEEKSTAR TOKYO vs ブレイブキングス刈谷）は前半のみの記録なのに、完全な試合として 19-16（実際の最終スコアは 34-33）で配信され続けていた。V2 では [handball-project#89](https://github.com/kinjo-ryura/handball-project/issues/89) で「（前半のみ）」の付記と注記を入れて修正済み。削除によりこの誤配信も解消した。
- **schemaVersion=1 のスキーマ仕様**（旧 root `SCHEMA.md`）は git 履歴から参照できる。

## 試合の追加手順

### 方法 A: アプリエクスポートから — **現在は通らない（要再整備）**

> ⚠️ **この経路は V1 配信の削除（2026-07-26）で塞がっている。** 新規サンプルの追加は当面 方法 B のみ。
>
> ハンド記録アプリの DEBUG エクスポータ (`MatchExporter`) は現在も **schemaVersion=1** を出力する（`SampleMatchSchemaVersion.current = 1`）。従来はその出力を root の `matches/` + `index.json` に置き、親リポの `tools/sample-converter-v2-py/` で repo 全体を V1 → V2 一括変換して `v2/` を再生成していた。V1 の置き場を削除したことで、この「V1 に置く → 変換する」段が成立しなくなった。
>
> 再整備には次のいずれかが要る:
>
> - エクスポータを V2 直接出力へ変更する（`SampleMatchConversionV2` 相当を DEBUG 経路にも通す）
> - 単一ファイルを V1 → V2 変換する CLI を用意し、`v2/` へ直接昇格させる
>
> 詳細と現況は [handball-project#116](https://github.com/kinjo-ryura/handball-project/issues/116) を参照。

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

配信先は `v2/highlights/`。schema 詳細は [v2/SCHEMA.md](v2/SCHEMA.md) の `/v2/highlights/{slug}.json` 節を参照。

1. **本体 JSON を作成して `v2/highlights/{slug}.json` に置く**
   - schema は `v2/matches/{slug}.json` と共通
   - HandballRecorder のハイライトモード + DEBUG エクスポートは **schemaVersion=1 を出力する**ため、そのままでは置けない（方法 A と同じ制約。上の ⚠️ を参照）
2. **`v2/highlights/index.json` の `highlights` 配列に summary を 1 件追加**
   - `slug` はファイル名（拡張子除く）と完全一致させる
   - `homeTeamName` / `awayTeamName` / `factCount` / `hasVideo` は本体から手動転記
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

現行: `schemaVersion: 2`。詳細は [v2/SCHEMA.md](v2/SCHEMA.md) 参照。

`schemaVersion: 1` の配信は 2026-07-26 に廃止・削除した（上の「V1 配信は廃止済み」節）。旧仕様は git 履歴の root `SCHEMA.md` を参照。

後方互換破壊変更時は `schemaVersion` を上げる。アプリ側は不一致を検出して当該試合をスキップする実装。
