# handball-sample-matches

iOS アプリ「ハンド記録」で配信するサンプル試合データの公開リポジトリ。

アプリは起動時にこの repo の `v2/index.json` を取得 → 各試合本体 (`v2/matches/{slug}.json`) を取得して、ユーザーの自分の試合と並べて「サンプル試合」として表示する。サンプル試合は端末側に永続化されない（毎回 fetch）。

## ディレクトリ構成

```
.
├── README.md            この repo の説明
├── .github/workflows/   CI（push / PR で v2/ を検証する。下の「配信データの検証」節）
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

PDF → JSON 変換スクリプトは親リポ `handball-project` の `tools/jhl-pdf-importer/` に置いている。

> ℹ️ **この README 内の `tools/...` はすべて親リポ `handball-project`（非公開）のパス。** 本 repo は submodule として parent の `apps/handball-sample-matches/` に置かれる前提で、コマンド例も親リポ root からの実行を想定している。公開 repo 単体を clone しただけでは実行できない。

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

### 方法 A: アプリエクスポートから（動画モード）

**動画モードのサンプルはこの経路が唯一の出所**（PDF には動画タイムスタンプが無いため方法 B では作れない）。

1. HandballRecorder の **DEBUG ビルド**で「V2データ」タブ → 対象の試合 → ツールバー「JSON エクスポート」→ 共有。出力は SAMPLE_DTO_V2（`schemaVersion: 2`）で、Rust コアの `exportSampleMatch` / `encodeSampleMatch` が生成する
2. 受け取った JSON を staging（gitignore 済みの場所）へ `{slug}.json` として置く。slug の規約は「[命名ルール](#命名ルール)」節
3. 親リポの `tools/promote-sample-matches/` で `v2/` へ昇格する（`v2/index.json` の更新・同一試合の二重配信検出をまとめて行う）:

   ```sh
   (cd tools/promote-sample-matches && uv sync)
   tools/promote-sample-matches/.venv/bin/python tools/promote-sample-matches/promote.py \
     --staging <JSON を置いたディレクトリ> \
     --dist apps/handball-sample-matches/v2 \
     --description "..." --dry-run
   ```

   実名を残すトップリーグ戦なら `--keep-player-names` を付ける（→「[個人情報の扱い](#個人情報の扱い)」）。付けない場合は選手名を背番号ラベルへ置き換えるため、**全選手に背番号が必要**（アプリは背番号なしの選手を登録できるが、無いと昇格が FAIL する）
4. `--dry-run` を外して本実行 → 検証 CLI（→「[配信データの検証](#配信データの検証)」）→ commit & push

> **経緯**: 2026-07-26〜07-30 はこの節で「V1 配信の削除で経路が塞がっている」と案内していたが**誤りだった**（[handball-project#137](https://github.com/kinjo-ryura/handball-project/issues/137)）。V2 直接出力はアプリ内に `ExportMatchSheetV2` として存在しており、`schemaVersion=1` を出力するのは V1 legacy の `MatchExporter`（`DevDataView` 専用で配信経路ではない）だった。V1 → V2 一括変換 CLI（`tools/sample-converter-v2-py/`）は入力ごと廃止済みで、この経路には関係しない。
>
> 手順 3 の昇格は exporter の golden fixture（バイト正のオラクル出力）で通ることを確認済み。手順 1 の実機 export からの通し確認は #137 で追跡中。

### 方法 B: JHL 公式 PDF からの自動生成（タイマーモード、ローカル専用）

JHL公式ランニングスコア PDF（`pdf/{試合コード}.pdf`）から、両チームのロースター・全得点・選手別シュート総数を抽出して JSON 化する。動画タイムスタンプは含まれない（タイマーモード前提）。

出力先は **`pdf-matches/` (gitignore 済み)** で、ここは配信前の staging。ここには**実名のロースターが入る**ので、そのまま `v2/` へコピーしてはいけない。内容を確認 → OK と判断したら、親リポの `tools/promote-sample-matches/` で昇格する（選手名の置き換え・`v2/index.json` の更新・同一試合の二重配信検出をまとめて行う）:

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

前者 2 件は公開済み `.video` サンプル（同一試合を動画から手記録したもの）と突き合わせ済みで、背番号別の得点内訳が全選手一致することを確認している。昇格前の検証は `handball-toolkit` の検証 CLI で行う（→「[配信データの検証](#配信データの検証)」）。push 後は CI が同じ検証を回す。

親リポ (`handball-project/`) の root から実行する（`cd` して相対パスで叩かない）:

```sh
(cd tools/jhl-pdf-importer && uv sync)
tools/jhl-pdf-importer/.venv/bin/python tools/jhl-pdf-importer/parse_jhl_pdf.py \
  apps/handball-sample-matches/pdf/jhl/running_501M156.pdf \
  --out apps/handball-sample-matches/pdf-matches/jhl/2026-04-25-jeekstar-vs-corazon.json
```

JHA 主催試合の PDF は `tools/jha-pdf-importer/parse_jha_pdf.py` と `pdf/jha/` → `pdf-matches/jha/` を使う（CLI・出力スキーマは同一）。

詳細・オプション・既知の制約は親リポの `tools/jhl-pdf-importer/README.md` / `tools/jha-pdf-importer/README.md` を参照。

Claude Code の `/import-pdf` skill 経由でも実行できる（リーグ判定・PDF からの抽出・検算まで対話で案内する）。

## ハイライトの追加手順

配信先は `v2/highlights/`。schema 詳細は [v2/SCHEMA.md](v2/SCHEMA.md) の `/v2/highlights/{slug}.json` 節を参照。

1. **本体 JSON を作成して `v2/highlights/{slug}.json` に置く**
   - schema は `v2/matches/{slug}.json` と共通
   - ハイライトモード（`.videoHighlight`）の DEBUG エクスポートも SAMPLE_DTO_V2（`schemaVersion: 2`）を出力するので、方法 A の手順 1 と同じ手順で取り出せる
   - ただし `tools/promote-sample-matches/` は `v2/matches/` + `v2/index.json` 専用で、**highlights の index は扱わない**（エントリの形が違う）。ハイライトは手順 2 の index 追記を手で行う
2. **`v2/highlights/index.json` の `highlights` 配列に summary を 1 件追加**
   - `slug` はファイル名（拡張子除く）と完全一致させる
   - `homeTeamName` / `awayTeamName` / `factCount` / `hasVideo` は本体から手動転記
   - 配列は date 降順で維持
3. commit & push

## 配信データの検証

`.github/workflows/validate.yml` が push（main）と PR のたびに [handball-toolkit](https://github.com/kinjo-ryura/handball-toolkit) の検証 CLI を走らせ、`v2/` 全体を検証する。index ↔ ファイルの突合と、スコア / `factCount` / `hasVideo` / `date` の転記整合まで見るので、手で index を書き換えたときの取りこぼしはここで落ちる。

- **失敗条件は CLI の exit code に従う**: error が 1 件でもあれば失敗、warning のみなら通過。現状は 53 ファイル / warning 1 件（`2025-12-20-f352ea46.json` の「前半のみ」= `corpus/matchCoverageIncomplete`）で green
- CI は toolkit の **main** をその場で checkout してビルドする。コア側で validators が強化されればこの検証にも自動で効く（＝ toolkit の変更でこちらが赤くなることもある）
- ビルドはクリーンで約 10 秒（依存が serde / serde_json / chrono / uuid だけ）なので、キャッシュは置いていない
- **同じ workflow が possession fact の混入も検査する**（`jq` で `facts[].payload.kind` を見る）。理由と解除条件は下の「possession fact は配信しない」節

手元で先に確認するときは同じコマンドを直接叩く（親リポ checkout 前提）:

```sh
cd ../handball-toolkit
cargo run -p handball-toolkit-cli -- validate ../handball-sample-matches/v2
```

`--json` を付けると機械可読出力になる。詳細は toolkit README の「CLI」節。

## 命名ルール

- slug は **ASCII 小文字 + 数字 + ハイフン** に限定（URL/ファイルシステム両対応）
- `{yyyy-MM-dd}-{home}-vs-{away}` を基本形式に
- 同日同対戦の重複は `-game2` / `-final` のような接尾辞で回避

## 個人情報の扱い

**関係者の本名を含むデータは push しない。**

判断基準は「**その試合の公式記録・公開配信で選手の実名が既に公表されているか**」。公表済みのトップリーグ戦は実名を残し、それ以外は**選手名を背番号ラベル（`7番`）へ置き換えて配信する**。実名で配信しているのは現在 10 件:

| 件数 | 内容 | 出所 |
|---|---|---|
| 4 | ZEEKSTAR TOKYO / ブレイブキングス刈谷 / 豊田合成ブルーファルコン（`.video` 2 + `.timer` 2） | JHL 公式ランニングスコア・公式配信 |
| 6 | Ohrid / Vardar / Eurofarm / Alkaloid / Tikvesh（北マケドニア）、BERA BERA / AULA（スペイン）の `.videoHighlight` | 各リーグの公開配信 |

アマチュア（高校・大学）・未成年を含む試合は必ず仮名化する。日本選手権のようなオープン大会はこれに該当する。

置き換えても選手別スタッツは失われない。fact は `playerKey` で選手を参照し、置き換えは `name` だけを差し替えてキーには触らないため。失われるのは表示名のみ。チーム名（`国分中央高校` 等）は組織名なのでそのまま残す。

> ⚠️ **これは仮名化であって匿名化ではない。** 背番号・チーム名・日付が分かれば、大会主催者が公開している公式記録 PDF を引くことで実名に戻せる。「配信データが実名を含まない」ことは満たすが「個人を特定できない」ことは満たさない、と理解したうえで運用する。

置き換えは親リポの `tools/promote-sample-matches/` が昇格時に行い、実行時に「元の実名が公開 JSON に残っていないか」を全文照合で検査する。**手でコピーせずこのツールを通すこと。** PDF 由来・アプリ export 由来のどちらも同じツールを通す。

実名を残す場合も同じツールを通し、`--keep-player-names` で明示 opt-in する。このモードでは仮名化と全文照合を行わないため、**目視確認が唯一の歯止めになる**（実行時に対象人数とチーム名を表示する）。エクスポータはアプリ内のデータをそのまま JSON 化するので、アプリ由来の JSON は特に内容を必ず確認すること。

## スキーマバージョン

現行: `schemaVersion: 2`。詳細は [v2/SCHEMA.md](v2/SCHEMA.md) 参照。

`schemaVersion: 1` の配信は 2026-07-26 に廃止・削除した（上の「V1 配信は廃止済み」節）。旧仕様は git 履歴の root `SCHEMA.md` を参照。

後方互換破壊変更時は `schemaVersion` を上げる。ただし **版チェックは index 単位の全体ゲート**で、アプリは `v2/index.json` の `schemaVersion` が自分の対応版と違えば「未対応のスキーマです (vN)」を出して**一覧ごと表示を止める**（`SampleMatchStoreV2.reload()`）。個別の match JSON は `schemaVersion` を持たないため、「1 件だけ版が新しい」は表現できない。

対して**個別ファイルの変換失敗は無言で脱落する** — 変換できなかった試合は `catch → continue` で一覧から消え、エラー表示も出ない（全滅したときだけ「取得に失敗しました」になる）。この非対称が次節の制約を生む。

## possession fact は配信しない（2026-08-21〜）

`schemaVersion` を上げずに possession fact（3 つめの fact 種別）を v2 契約へ足したため、**出荷済み 1.2.0 は possession 入りの試合を変換できず、その試合だけが一覧から無言で消える**。possession を parse できるのは **1.3.0 以降**。

`schemaVersion` を 3 に上げれば無言脱落は明示エラーに変わるが、上記のとおり版チェックは全体ゲートなので**配信中の全試合が 1.2.0 端末から消える**。1 件のために全体を落とす取引は割に合わない。1.2.0 に届く粒度は「全部か、無言脱落か」の二択しかない。

そこで **possession fact を含む試合は `/v2/` に配信しない**。

- **方針だけでは守れないので CI で守る**。エクスポータは fact を選別せず、試合が possession を持てば無条件で JSON に載る（toolkit の `encode_payload`）。配信データに possession が 0 件なのは方針の結果ではなく、possession を持つ試合をまだエクスポートしていないだけ。`validate.yml` の検査が唯一の歯止め
- **1.2.0 の利用者を能動的に押し上げる手段は無い**。アプリ内アップデート導線（[#150](https://github.com/kinjo-ryura/handball-project/issues/150)）は 1.3.0 で入ったため 1.2.0 端末には届かない。自然な更新を待つしかない
- **供給側は増えている**。macOS には possession 記録の導線があり、video-analysis も possession を出力経路に持つ。「表示する予定がない」ことは混入を防がない
- **解除の条件**: 1.2.0 の実利用が十分に落ちたと判断できた時点で、`validate.yml` の該当ステップとこの節を消す
- 経緯と判断: [handball-project#194](https://github.com/kinjo-ryura/handball-project/issues/194)
