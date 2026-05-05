# JSON スキーマ仕様 (schemaVersion=1)

iOS アプリ「ハンド記録」が読み取るサンプル試合データの仕様。

すべての日付フィールドは **ISO 8601** 文字列（例: `"2025-04-15T13:00:00Z"`）。

## `index.json`

試合一覧の軽量メタデータ。アプリが最初に取得する。

```jsonc
{
  "schemaVersion": 1,
  "matches": [
    {
      "slug": "2025-04-15-tigers-vs-falcons",
      "displayName": "Tigers vs Falcons",
      "description": "前後半で得点が大きく動いた一戦",
      "date": "2025-04-15T13:00:00Z",
      "homeScore": 24,
      "awayScore": 22,
      "hasYouTubeURL": true
    }
  ]
}
```

| フィールド | 型 | 必須 | 説明 |
|---|---|---|---|
| `schemaVersion` | Int | ✓ | 現行 `1` |
| `matches` | Array | ✓ | 試合サマリの配列。date 降順で並べる |
| `matches[].slug` | String | ✓ | `matches/{slug}.json` のファイル名（拡張子除く）と完全一致 |
| `matches[].displayName` | String | ✓ | 「Home vs Away」形式の表示名 |
| `matches[].description` | String? | | 試合の見どころ 1 文。`null` 可 |
| `matches[].date` | Date | ✓ | ISO 8601 |
| `matches[].homeScore` | Int | ✓ | 試合本体のイベントから計算した結果を手動転記 |
| `matches[].awayScore` | Int | ✓ | 同上 |
| `matches[].hasYouTubeURL` | Bool | ✓ | YouTube URL があれば true |

## `matches/{slug}.json`

試合 1 件分の本体。

```jsonc
{
  "schemaVersion": 1,
  "match": {
    "displayName": "Tigers vs Falcons",
    "date": "2025-04-15T13:00:00Z",
    "youtubeURL": "https://youtu.be/...",
    "phaseDurationSeconds": 1800
  },
  "teams": {
    "home": {
      "key": "home",
      "name": "Tigers",
      "players": [
        { "key": "h_07", "name": "Player A", "jerseyNumber": 7 }
      ]
    },
    "away": {
      "key": "away",
      "name": "Falcons",
      "players": [
        { "key": "a_05", "name": "Player B", "jerseyNumber": 5 }
      ]
    }
  },
  "events": [
    {
      "eventType": 0, "phase": 0,
      "phaseTimeSeconds": 0, "videoTimeSeconds": 0,
      "createdAt": "2025-04-15T13:00:00Z",
      "teamKey": null, "playerKey": null,
      "relatedPlayerKey": null, "note": null
    },
    {
      "eventType": 1, "phase": 0,
      "phaseTimeSeconds": null, "videoTimeSeconds": 95.5,
      "createdAt": "2025-04-15T13:01:35Z",
      "teamKey": "home", "playerKey": "h_07",
      "relatedPlayerKey": null, "note": null
    }
  ]
}
```

### `match` ヘッダ

| フィールド | 型 | 必須 | 説明 |
|---|---|---|---|
| `displayName` | String | ✓ | 表示用試合名 |
| `date` | Date | ✓ | ISO 8601 |
| `youtubeURL` | String? | | YouTube URL。動画なし試合は `null` |
| `phaseDurationSeconds` | Double? | | 1 ピリオドの秒数（通常 1800 = 30 分）|

### `teams`

`home` / `away` の 2 オブジェクト。`key` は `"home"` / `"away"` 固定。

| フィールド | 型 | 必須 | 説明 |
|---|---|---|---|
| `key` | String | ✓ | `"home"` または `"away"` |
| `name` | String | ✓ | チーム名 |
| `players[].key` | String | ✓ | 試合内で一意な選手識別子。`events[].playerKey` から参照される |
| `players[].name` | String | ✓ | 選手名 |
| `players[].jerseyNumber` | Int? | | 背番号。省略 / `null` 可 |

選手 key は試合内で一意であれば任意の文字列。アプリ側はデコード時に新規 UUID にマップするので、複数試合で同じ key を使っても衝突しない。

### `events`

時系列のイベント配列。

| フィールド | 型 | 必須 | 説明 |
|---|---|---|---|
| `eventType` | Int | ✓ | 後述の enum 値 |
| `phase` | Int | ✓ | 後述の enum 値 |
| `phaseTimeSeconds` | Double? | | その phase 内の経過秒数。動画モード由来の play event は `null` のことがある |
| `videoTimeSeconds` | Double? | | 動画再生位置（秒）。タイマーモード由来の event は `null` |
| `createdAt` | Date | ✓ | ISO 8601。記録時刻。アプリ側のソート安定化に使う |
| `teamKey` | String? | | `"home"` / `"away"` / `null` |
| `playerKey` | String? | | `teams.{home,away}.players[].key` を参照 |
| `relatedPlayerKey` | String? | | アシスト選手等。同じく players[].key 参照 |
| `note` | String? | | 自由記述メモ |

### `eventType` enum

| 値 | 名前 | 意味 |
|---|---|---|
| 0 | `phaseStarted` | phase 開始マーカー（control event）|
| 1 | `goal` | 得点 |
| 2 | `shotMissed` | シュート失敗 |
| 3 | `timeout` | (旧) タイムアウト |
| 4 | `yellowCard` | イエローカード |
| 5 | `twoMinuteSuspension` | 2 分間退場 |
| 6 | `redCard` | レッドカード |
| 7 | `phaseEnded` | phase 終了マーカー（control event）|
| 8 | `timeoutStarted` | タイムアウト開始（control event）|
| 9 | `timeoutEnded` | タイムアウト終了（control event）|
| 10 | `timerPaused` | タイマー停止（control event）|
| 11 | `timerResumed` | タイマー再開（control event）|

### `phase` enum

| 値 | 名前 | 意味 |
|---|---|---|
| 0 | `firstHalf` | 前半 |
| 1 | `secondHalf` | 後半 |
| 2 | `overtime1` | 延長前半 |
| 3 | `overtime2` | 延長後半 |
| 4 | `shootout` | 7m スローコンテスト |

### イベント記録のルール

- **動画モード**（`youtubeURL` あり）の play event は `videoTimeSeconds` のみ保存し `phaseTimeSeconds` は `null` のことがある（アプリ側がセグメントから推定する）
- **タイマーモード**（`youtubeURL` なし）の play event は `phaseTimeSeconds` のみ。`videoTimeSeconds` は `null`
- control event ペア（`timeoutStarted` ↔ `timeoutEnded`、`timerPaused` ↔ `timerResumed`）は対称になっていることをアプリ側が前提にしている
- 各 phase は `phaseStarted`（任意）と `phaseEnded`（必須）で囲む。`phaseEnded` が無いと phase の終端時刻が不定になる

## `highlights/index.json`

ハイライト一覧の軽量メタデータ。試合 (`matches/`) とは別経路で配信される（試合とは性質が異なり一覧 UI も分かれるため）。

```jsonc
{
  "schemaVersion": 1,
  "highlights": [
    {
      "slug": "2026-05-05-bera-bera-vs-aula",
      "displayName": "石川空選手の得点。",
      "description": null,
      "date": "2026-05-05T09:29:49Z",
      "homeTeamName": "BERA BERA",
      "awayTeamName": "AULA",
      "eventCount": 1,
      "hasYouTubeURL": true
    }
  ]
}
```

| フィールド | 型 | 必須 | 説明 |
|---|---|---|---|
| `schemaVersion` | Int | ✓ | 現行 `1` |
| `highlights` | Array | ✓ | ハイライトサマリの配列。date 降順で並べる |
| `highlights[].slug` | String | ✓ | `highlights/{slug}.json` のファイル名（拡張子除く）と完全一致 |
| `highlights[].displayName` | String | ✓ | クリップタイトル（試合名ではなく「石川空選手の得点。」など） |
| `highlights[].description` | String? | | 補足 1 文。`null` 可 |
| `highlights[].date` | Date | ✓ | ISO 8601 |
| `highlights[].homeTeamName` | String | ✓ | 元試合の home チーム名 |
| `highlights[].awayTeamName` | String | ✓ | 元試合の away チーム名 |
| `highlights[].eventCount` | Int | ✓ | 本体に含まれる play event 数（`goal` / `shotMissed` / `freeNote`）|
| `highlights[].hasYouTubeURL` | Bool | ✓ | YouTube URL があれば true |

## `highlights/{slug}.json`

ハイライト 1 件分の本体。**schema は `matches/{slug}.json` と同一**（`schemaVersion: 1`、`match` / `teams` / `events` 構造）。違いは:

- `match.displayName` がクリップタイトル（試合名ではなく「石川空選手の得点。」など）
- 記録される event は `goal` / `shotMissed` / `freeNote` の 3 type を主とする。phase は `firstHalf` のみで、`phaseStarted` / `phaseEnded` で囲む（`phaseEnded.videoTimeSeconds` は動画長）
- `away.players` が空配列のことがある（home 側の選手だけ取り上げる場合）

アプリ側はハイライト経路でロードした `Match` に `recordingMode = .highlight` を確実にセットして扱う（JSON 内に `recordingMode` フィールドは現状含めない、schemaVersion=1 互換維持）。

## バリデーション

アプリ側のローダは以下を検出すると **試合をスキップ** + エラーログ:

- `schemaVersion` 不一致
- `eventType` / `phase` が enum 範囲外
- `teamKey` / `playerKey` / `relatedPlayerKey` が `teams` 内に存在しない

push 前にこれらの整合性を目視確認するか、エクスポータ経由で生成するのが安全。
