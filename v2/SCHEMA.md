# JSON スキーマ仕様 (schemaVersion=2)

iOS アプリ「ハンド記録」が `/v2/` path 経由で読み取る V2 サンプル試合データの仕様。

V1 schema との主な差分:
- `MatchConfiguration` は tagged union (`kind` discriminator + sub-payload)
- 旧 `phaseRules` / `MatchPhase.timeline` は廃止 (phase 情報は `phaseStart` fact から再現)
- `ControlFact` は 2 case sum type (`phaseStart` / `stoppage`)
- `MatchClock` は試合通算累積秒 (phase 内秒数ではない)

すべての日付フィールドは ISO 8601 文字列。

## ステータス

V2 schema は 2026-05 現在 cutover 中。 `/v2/index.json` には matches が空配列で配置されており、 既存 V1 サンプルからの conversion は別 task で実施予定。

V1 schema (`/matches/` / `/highlights/`) は引き続き運用中で、 アプリの V1 経路はこちらを参照する。

## ファイル配置

```
/v2/
  index.json
  matches/
    {slug}.json
  highlights/
    index.json
    {slug}.json
```

## `/v2/index.json`

```jsonc
{
  "schemaVersion": 2,
  "matches": [
    {
      "slug": "2025-04-15-tigers-vs-falcons",
      "displayName": "Tigers vs Falcons",
      "description": "前後半で得点が大きく動いた一戦",
      "date": "2025-04-15T13:00:00Z",
      "homeScore": 24,
      "awayScore": 22,
      "hasVideo": true
    }
  ]
}
```

| フィールド | 型 | 必須 | 説明 |
|---|---|---|---|
| `schemaVersion` | Int | ✓ | `2` |
| `matches[].slug` | String | ✓ | `matches/{slug}.json` のファイル名 (拡張子除く) と完全一致 |
| `matches[].displayName` | String | ✓ | 表示名 |
| `matches[].description` | String? | | 1 文の見どころ |
| `matches[].date` | Date | ✓ | ISO 8601 |
| `matches[].homeScore` | Int | ✓ | goal 集計結果を手動転記 |
| `matches[].awayScore` | Int | ✓ | 同上 |
| `matches[].hasVideo` | Bool | ✓ | `.video` / `.videoHighlight` のとき true |

## `/v2/matches/{slug}.json`

```jsonc
{
  "schemaVersion": 2,
  "match": {
    "displayName": "Tigers vs Falcons",
    "date": "2025-04-15T13:00:00Z",
    "configuration": {
      "kind": "video",
      "video": {
        "source": {
          "provider": "youtube",
          "externalID": "abc123"
        }
      }
    }
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
  "facts": [
    {
      "factID": "...",
      "recordedAt": "2025-04-15T13:00:00Z",
      "payload": {
        "kind": "control",
        "control": {
          "kind": "phaseStart",
          "phaseStart": { "kind": "regular" },
          "anchor": {
            "kind": "both",
            "matchClock": { "elapsedSeconds": 0 },
            "videoClock": { "elapsedSeconds": 0 },
            "endMatchElapsedSeconds": 1800,
            "endVideoElapsedSeconds": 1800
          }
        }
      }
    },
    {
      "factID": "...",
      "recordedAt": "2025-04-15T13:01:35Z",
      "payload": {
        "kind": "play",
        "play": {
          "kind": "goal",
          "teamKey": "home",
          "playerKey": "h_07",
          "relatedPlayerKey": null,
          "anchor": {
            "kind": "videoClock",
            "videoClock": { "elapsedSeconds": 95.5 }
          },
          "title": null,
          "note": null
        }
      }
    }
  ]
}
```

### `match.configuration` (tagged union)

`kind` discriminator で case を分岐:

| `kind` | サブペイロード | 説明 |
|---|---|---|
| `"timer"` | `timer.phaseDurationSeconds: Double` | タイマーモード (動画なし) |
| `"video"` | `video.source: { provider, externalID }` | 動画モード (フル試合) |
| `"videoHighlight"` | `videoHighlight.source: { provider, externalID }` | ハイライト経路 (内部経路フラグ、 通常 `/v2/highlights/` で使う) |

`provider` は `"youtube"` のみ。

### `facts[].payload` (tagged union)

| `kind` | サブペイロード |
|---|---|
| `"play"` | `play: SamplePlayFact` |
| `"control"` | `control: SampleControlFact` |

#### `SamplePlayFact`

| フィールド | 型 | 説明 |
|---|---|---|
| `kind` | String | `"goal"` / `"shotMissed"` / `"freeNote"` / `"yellowCard"` / `"twoMinuteSuspension"` / `"redCard"` |
| `teamKey` | String? | `"home"` / `"away"` / null |
| `playerKey` | String? | `teams.{home,away}.players[].key` |
| `relatedPlayerKey` | String? | 同上 |
| `anchor` | `SampleFactAnchor` | 後述 |
| `title` | String? | `freeNote` 用の見出し |
| `note` | String? | 自由記述 |

#### `SampleControlFact`

| フィールド | 型 | 説明 |
|---|---|---|
| `kind` | String | `"phaseStart"` / `"stoppage"` |
| `phaseStart` | `{ kind: String }` | `kind = "regular"` / `"shootout"` |
| `stoppage` | `{ stoppageKind: String, note: String? }` | `stoppageKind = "timeout"` / `"pause"` |
| `anchor` | `SampleFactAnchor` | 後述。 `phaseStart` は end 必須、 `stoppage` は end optional |

### `SampleFactAnchor`

| フィールド | 型 | 説明 |
|---|---|---|
| `kind` | String | `"matchClock"` / `"videoClock"` / `"both"` |
| `matchClock` | `{ elapsedSeconds: Double }?` | 試合通算累積秒 |
| `videoClock` | `{ elapsedSeconds: Double }?` | 動画再生位置 |
| `endMatchElapsedSeconds` | Double? | 範囲の end (matchClock 側) |
| `endVideoElapsedSeconds` | Double? | 範囲の end (videoClock 側) |

end 系は `phaseStart` (必須) / `stoppage` (任意) で使用。 `play` では両方 null。

end の `kind` は start の `kind` を継承する (両方 nil なら range なし)。

### `MatchClock` の累積秒

V1 は phase 内秒数 (`phaseTimeSeconds`) だったが、 V2 は試合通算 (`elapsedSeconds`) に統一されている。 例:
- firstHalf (1800 秒) → matchClock 0..1800
- secondHalf (1800 秒) → matchClock 1800..3600

## `/v2/highlights/index.json`

```jsonc
{
  "schemaVersion": 2,
  "highlights": [
    {
      "slug": "2026-05-05-bera-bera-vs-aula",
      "displayName": "石川空選手の得点。",
      "description": null,
      "date": "2026-05-05T09:29:49Z",
      "homeTeamName": "BERA BERA",
      "awayTeamName": "AULA",
      "factCount": 1,
      "hasVideo": true
    }
  ]
}
```

| フィールド | 型 | 説明 |
|---|---|---|
| `factCount` | Int | 本体の facts 配列の長さ |
| 他 | | matches/index と同等。 chart は `homeTeamName` / `awayTeamName` 必須 |

## `/v2/highlights/{slug}.json`

本体スキーマは `/v2/matches/{slug}.json` と同一。 違いは:
- `configuration.kind` は通常 `"videoHighlight"` (フル試合と区別)
- `away.players` は空配列のことがある (home 側の選手だけ取り上げる場合)
- アプリの `HighlightStoreV2` は `configurationOverride: .videoHighlight(source)` で読み込み時に固定する
