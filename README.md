# handball-sample-matches

iOS アプリ「ハンド記録」で配信するサンプル試合データの公開リポジトリ。

アプリは起動時にこの repo の `index.json` を取得 → 各試合本体 (`matches/{slug}.json`) を取得して、ユーザーの自分の試合と並べて「サンプル試合」として表示する。サンプル試合は端末側に永続化されない（毎回 fetch）。

## ディレクトリ構成

```
.
├── README.md            この repo の説明
├── SCHEMA.md            JSON スキーマ仕様（schemaVersion=1）
├── index.json           試合一覧の軽量メタデータ（必須）
└── matches/             試合本体 JSON（必須）
    └── {slug}.json      1 試合 = 1 ファイル
```

`index.json` と `matches/{slug}.json` のパスはアプリ側 (`SampleMatchLoader`) で固定。**変えると読めなくなる**。

## URL

- 一覧: `https://raw.githubusercontent.com/kinjo-ryura/handball-sample-matches/main/index.json`
- 試合本体: `https://raw.githubusercontent.com/kinjo-ryura/handball-sample-matches/main/matches/{slug}.json`

`raw.githubusercontent.com` は HTTPS（ATS 対応）+ Fastly CDN（デフォルト約 5 分キャッシュ）。

## 試合の追加手順

1. **試合本体 JSON を作成して `matches/{slug}.json` に置く**
   - ハンド記録アプリの DEBUG ビルドで「データ」タブ → 試合詳細 → 共有ボタン → JSON を書き出すのが最短
   - エクスポータが出力する slug は `{yyyy-MM-dd}-{home}-vs-{away}` 形式（日本語チーム名は `{date}-{8桁hex}` フォールバック）。意味のあるファイル名にリネーム推奨
2. **`index.json` の `matches` 配列に summary を 1 件追加**
   - `slug` はファイル名（拡張子除く）と完全一致させる
   - `homeScore` / `awayScore` / `hasYouTubeURL` は試合本体から手動で転記（軽量メタとして index 単独で表示するため重複を許容）
   - 配列は date 降順（新しい試合が上）で維持する
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
