# JSON スキーマ仕様 (schemaVersion=2)

iOS アプリ「ハンド記録」が `/v2/` path 経由で読み取る V2 サンプル試合データの仕様。

V1 schema との主な差分:
- `MatchConfiguration` は tagged union (`kind` discriminator + sub-payload)
- 旧 `phaseRules` / `MatchPhase.timeline` は廃止 (phase 情報は `phaseStart` fact から再現)
- `ControlFact` は 2 case sum type (`phaseStart` / `stoppage`)
- `MatchClock` は試合通算累積秒 (phase 内秒数ではない)

すべての日付フィールドは ISO 8601 文字列。

### `date` と `recordedAt` の使い分け

- **`date`（`index.json` / `match.date`）は試合開始日時**。「いつ記録したか」ではなく「いつ試合が行われたか」を表す。動画から後日記録したハイライトでも、`date` は試合当日を指す（例: 2026-04-10 の試合を 2026-05-09 に記録した場合も `date` は `2026-04-10T05:32:00Z`）。slug の先頭 `{yyyy-MM-dd}` と日付部分が一致することが不変条件（検証 CLI の `corpus/slugDateMismatch`）。
- **試合開始時刻が不明な場合は `T00:00:00Z`** を使う（PDF 由来の試合など、日付しか分からないもの）。分かっている場合のみ実際の開始時刻を入れる。
- **`facts[].recordedAt` が記録日時**。アプリが fact を記録した実時刻で、`date` とは独立に動く。

> ⚠️ V1 → V2 backfill 時に一部ハイライトの `date` へ記録日時が転記される退行があり、2026-07-28 に修正した（[handball-project#115](https://github.com/kinjo-ryura/handball-project/issues/115)）。`date` に `recordedAt` を流し込まないこと。

## ステータス

**現行かつ唯一の配信経路。** cutover は完了済みで、`/v2/index.json` は 45 試合、`/v2/highlights/index.json` は 6 ハイライトを配信中。

アプリは 1.1.0（2026-07-11 公開）以降 `SampleMatchLoaderV2` でこの `/v2/` 経路のみを読む。

V1 schema (`/index.json` / `/matches/` / `/highlights/` および root の `SCHEMA.md`) は **2026-07-26 に削除した**。旧仕様は git 履歴を参照。経緯は README「V1 配信は廃止済み」節。

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
| `matches[].date` | Date | ✓ | 試合開始日時。本体の `match.date` と一致必須（検証 CLI の `corpus/dateMismatch`） |
| `matches[].homeScore` | Int | ✓ | goal 集計結果を手動転記 |
| `matches[].awayScore` | Int | ✓ | 同上 |
| `matches[].hasVideo` | Bool | ✓ | `.video` / `.videoHighlight` のとき true |

`matches` 配列は **`date` 降順**（新しい試合が上）で維持する（検証 CLI の `corpus/indexNotDateDescending`）。アプリは配列順をそのまま表示に使う。

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
    },
    {
      "factID": "...",
      "recordedAt": "2025-04-15T13:01:37Z",
      "payload": {
        "kind": "possession",
        "possession": {
          "teamKey": "away",
          "anchor": {
            "kind": "videoClock",
            "videoClock": { "elapsedSeconds": 97.0 },
            // 終わりは任意。省けば消費側が導出する
            "endVideoElapsedSeconds": 118.5
          }
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

### `facts[]`

| フィールド | 型 | 必須 | 説明 |
|---|---|---|---|
| `factID` | String? | | fact の同一性。**同一試合内で一意**（検証 CLI の `corpus/duplicateFactID`）。省略時はアプリ側が採番する |
| `recordedAt` | Date | ✓ | アプリが fact を記録した実時刻（`date` とは独立） |
| `payload` | tagged union | ✓ | 後述 |

### `facts[].payload` (tagged union)

| `kind` | サブペイロード |
|---|---|
| `"play"` | `play: SamplePlayFact` |
| `"control"` | `control: SampleControlFact` |
| `"possession"` | `possession: SamplePossessionFact` |

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

#### `SamplePossessionFact`

**ポゼッション開始**（あるチームのポゼッションがそこから始まった、という点の事実）。

| フィールド | 型 | 説明 |
|---|---|---|
| `teamKey` | String | `"home"` / `"away"`。**必須** (`SamplePlayFact.teamKey` と違い null 不可) |
| `anchor` | `SampleFactAnchor` | 始まり。end 系は**任意** (後述) |

判別子 (`kind`) を持たない。ポゼッションの fact は「開始」1 種類だけで、2 つ目が出た時点でどのみちスキーマ変更になるため、空の判別子を先置きしない。

**`anchor` が指すのは「そのチームのプレーが動き出した瞬間」**。ターンオーバーやルーズボールの確保では保持した瞬間と一致するが、**得点・ボールアウトの後はスローオフ / スローインが実行された瞬間**であって、被得点の瞬間ではない。生成側と消費側でここがズレると**エラーにならず統計だけが静かに間違う**。

> ⚠️ 2026-08-22 に改めた（[handball-project#220](https://github.com/kinjo-ryura/handball-project/issues/220)）。旧仕様は「ボールを保持した瞬間 (GK のキャッチ / ルーズボールの確保 / ターンオーバーの成立 / **被得点の瞬間**)」で、攻撃効率が世界共通でその区切りで計算されることを根拠にしていた。**その根拠は数え方の境界 (手が替わったら 1 ポゼッション) を守るものであって、境界の秒を縛らない** — 点を保持からプレー再開へ動かしてもポゼッション数は 1 件も変わらないので、攻撃効率の比較可能性は失われない。加えて旧仕様は生成側が原理的に満たせなかった (カメラの向きの変化から検出する経路では、カメラが動かない被得点の瞬間に手がかりが無い)。**旧仕様で生成した既存データはそのまま有効**（この差は数秒で、`unexpectedAnchorEnd` のような検証エラーにはならない）。

**終わり (`anchor` の end 系) は任意**。書けるなら書き、出せないなら省く — どちらも正常な入力で、省いた場合は消費側が導出する。

> ⚠️ 2026-08-22 に改めた（[handball-project#220](https://github.com/kinjo-ryura/handball-project/issues/220)）。旧仕様は「end 系は両方 null」で、書いても検証 CLI が `unexpectedAnchorEnd` で弾いていた。**必須にはしない** — 必須化すると同じことを言う 2 つの fact (この end と次のポゼッション開始) が食い違え、終わりを出せない生成側の出力が一切取り込めなくなる。**end を持たない既存データはそのまま有効**。

消費側の導出は **`明示 end` → `区間内の同チーム goal` → `次のポゼッション開始 / その phase の end`** の順で、いずれも最後の「次の開始 / phase end」で**クランプする**。したがって:

- **end が次のポゼッション開始より後ろでも検証エラーにしない**。生成側は 1 試合あたり 100 件超を書き、goal の時刻と次の開始は独立に推定されるので順序の逆転は常態。消費側がクランプするので区間の重なりは起こりえない
- **区間が隙間なく phase を覆う必要は無い**。隙間はボールデッド (得点後、次の攻撃が始まるまで) であって取りこぼしとは限らない
- 検証されるのは **`start < end`** だけ (逆順 / 0 長は `possessionEndBeforeStart`)

同じチームの `possession` が 2 件続くのは**矛盾ではなく 2 件目が冗長なだけ**で、ポゼッション数は「チームが切り替わった回数」で数える。

### `SampleFactAnchor`

| フィールド | 型 | 説明 |
|---|---|---|
| `kind` | String | `"matchClock"` / `"videoClock"` / `"both"` |
| `matchClock` | `{ elapsedSeconds: Double }?` | 試合通算累積秒 |
| `videoClock` | `{ elapsedSeconds: Double }?` | 動画再生位置 |
| `endMatchElapsedSeconds` | Double? | 範囲の end (matchClock 側) |
| `endVideoElapsedSeconds` | Double? | 範囲の end (videoClock 側) |

end 系は `phaseStart` (必須) / `stoppage` (任意) / `possession` (任意) で使用。 `play` では両方 null（非 null は検証 CLI の `corpus/unexpectedAnchorEnd`。 デコードは値を黙って捨てるため、 書いても区間にはならない）。

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
      "slug": "2026-04-19-bera-bera-vs-aula",
      "displayName": "石川空選手の得点。",
      "description": null,
      "date": "2026-04-19T00:00:00Z",
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
| `date` | Date | 試合開始日時。本体の `match.date` と一致必須（`corpus/dateMismatch`） |
| 他 | | matches/index と同等。 chart は `homeTeamName` / `awayTeamName` 必須 |

`highlights` 配列は **`date` 降順**（新しい試合が上）で維持する（`corpus/indexNotDateDescending`）。アプリの `HighlightStoreV2` は index の順序をそのまま保持し、`MatchListViewV2` はソートせず配列順で描画するため、ここの順序が画面の並びになる。

## `/v2/highlights/{slug}.json`

本体スキーマは `/v2/matches/{slug}.json` と同一。 違いは:
- `configuration.kind` は通常 `"videoHighlight"` (フル試合と区別)
- `away.players` は空配列のことがある (home 側の選手だけ取り上げる場合)
- アプリの `HighlightStoreV2` は `configurationOverride: .videoHighlight(source)` で読み込み時に固定する
