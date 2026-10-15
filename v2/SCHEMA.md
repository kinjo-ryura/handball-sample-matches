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

**現行かつ唯一の配信経路。** cutover は完了済みで、`/v2/index.json` は 45 試合、`/v2/highlights/index.json` は 6 ハイライトを配信中。大会（`/v2/competitions/`）は 2026-10 に足した（下の「大会」節）。

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
  competitions/
    index.json
    {id}.json
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
| `matches[].homeTeamName` | String? | | 本体 `teams.home.name` の転記。**大会の一覧では必須**（下の「大会」節）。書いてあれば本体と一致必須（`corpus/teamNameMismatch`） |
| `matches[].awayTeamName` | String? | | 同上（`teams.away.name`） |

`matches` 配列は **`date` 降順**（新しい試合が上）で維持する（検証 CLI の `corpus/indexNotDateDescending`）。アプリは配列順をそのまま表示に使う。

## `/v2/competitions/`（大会）

**大会**は、「注目の試合」（`/v2/index.json`）とは別に配信する試合のまとまり（[handball-project#566](https://github.com/kinjo-ryura/handball-project/issues/566)。2026-10-14 に「index には載せず、別のものとして 1 つの大会として扱う」と決めた）。
1 シーズン分のリーグ戦のように数百試合になるものを、注目の試合の一覧に混ぜずに出すためのもの。
アプリは「注目の試合」の上に大会ごとのバナーを出し、タップでその大会の試合だけを一覧にする。
季が終わったら `competitions/index.json` から外す（本体と一覧ファイルは残してよい。試合 URL は開き続ける）。

```jsonc
// /v2/competitions/index.json
{
  "schemaVersion": 2,
  "competitions": [
    {
      "id": "league-h-2026-27-men",
      "title": "リーグH 2026-27 男子",
      "description": "2026-27 シーズンの男子の全試合（タイマーモード・動画なし）"
    },
    {
      "id": "league-h-2026-27-women",
      "title": "リーグH 2026-27 女子",
      "description": "2026-27 シーズンの女子の全試合（タイマーモード・動画なし）"
    }
  ]
}
```

| フィールド | 型 | 必須 | 説明 |
|---|---|---|---|
| `schemaVersion` | Int | ✓ | `2` |
| `competitions[].id` | String | ✓ | `competitions/{id}.json` のファイル名（拡張子除く）と完全一致。字種は slug と同じ（英数字と `-`、1〜64 文字）。`index` は予約語（`corpus/unsafeCompetitionID`） |
| `competitions[].title` | String | ✓ | 画面に出す大会名 |
| `competitions[].description` | String? | | 1 文の説明（バナーの副題） |

`competitions` 配列は**アプリがバナーを出す順**。並びの規則は持たない（配列順をそのまま使う）。
`competitions/index.json` が無い（404）ことは「大会を配信していない」として扱い、アプリは失敗にしない。

```jsonc
// /v2/competitions/{id}.json — /v2/index.json と同じ形に、節 / 回戦のラベルと表を足したもの
{
  "schemaVersion": 2,
  "matches": [
    {
      "slug": "2026-10-12-tigers-vs-falcons",
      "displayName": "Tigers vs Falcons",
      "description": "2026-27 レギュラーシーズンのタイマーモード版（動画なし）",
      "date": "2026-10-12T04:00:00Z",
      "homeScore": 24,
      "awayScore": 22,
      "hasVideo": false,
      "homeTeamName": "Tigers",
      "awayTeamName": "Falcons",
      "round": "第1節"
    }
  ],
  "tables": [
    {
      "id": "standings",
      "title": "順位表",
      "columns": [
        { "title": "順位", "align": "trailing" },
        { "title": "チーム", "align": "leading" },
        { "title": "勝点", "align": "trailing" }
      ],
      "rows": [
        ["1", "Tigers", "2"],
        ["2", "Falcons", "0"]
      ]
    }
  ]
}
```

- 項目の形と slug の規則は `/v2/index.json` と同じ。**`homeTeamName` / `awayTeamName` は必須**（`corpus/missingTeamName`）— アプリは大会の一覧を**本体を取らずに**項目だけで描く（数百試合の本体を一覧のたびに取らないため）
- **`matches[].round`（String、任意）は節 / 回戦の表示ラベル**（例: `"第1節"`・`"準決勝"`・`"プレーオフ 準決勝"`・`"予選リーグ A組"`）。アプリは一覧の並びのまま、**同じラベルが続く行を 1 つのセクション**にまとめ、ラベルを見出しに出す。ラベルの無い行は見出しの無いセクションになる。文言は配信側が組む（アプリは語を足さない）
- **大会の一覧の並びは配信側が決める**（`date` 降順の縛りは `/v2/index.json` だけ）。`promote.py` は節 / 回戦の新しい順 → `date` 降順に並べる（ラベル中の数字で順を取り、数字の無いラベル — 決勝・プレーオフなど — を先頭に置く）。延期した試合も本来の節に入る。検証 CLI は、同じ `round` が離れて 2 回現れないこと（`corpus/roundNotContiguous`）と、同じ `round` の中が `date` 降順であること（`corpus/indexNotDateDescending`）を見る
- **本体は `/v2/matches/{slug}.json` を共有する。** 試合 URL（`?match=<slug>`）とアプリの直接取得は本体の置き場しか見ないので、大会の試合も同じ URL で開ける
- **1 つの本体は `/v2/index.json` か 1 つの大会のどちらか一方にだけ載せる**（`corpus/slugListedTwice`）。どこにも載らない本体は `corpus/orphanMatchFile`
- 大会の一覧にはフル試合だけを載せる（ハイライトは `/v2/highlights/`）。アプリは `hasVideo` からモードを決める（false = タイマー、true = 動画）
- 検証 CLI の突合: `competitions/index.json` の id に一覧ファイルが無い（`corpus/missingCompetitionFile`）・index に無い一覧ファイル（`corpus/orphanCompetitionFile`）・id の重複（`corpus/duplicateCompetitionID`）

### 大会 1 つ = 順位表 1 つ・山 1 つ（2026-10-15 — [handball-project#567](https://github.com/kinjo-ryura/handball-project/issues/567)）

男子と女子のように**順位表や山が別になるものは、別の大会として配信する**（例: `league-h-2026-27-men` / `league-h-2026-27-women`）。
リーグ戦でもトーナメントでも、この規則 1 つで配信側が分けられ、アプリは「大会 1 つに表 1 組」しか知らなくてよい。
レギュラーシーズンとプレーオフのような**段階は同じ大会に入れ**、`round` のラベル（「第1節」「プレーオフ 準決勝」）で分ける。

**大会の形式（リーグ / トーナメント）を表す項目は持たない。** アプリの表示に要るのは「試合をどう区切るか」（`round`）と
「どの表を出すか」（`tables`）の 2 つだけで、リーグ戦らしい画面（節 + 順位表）もトーナメントらしい画面（回戦だけ）も
データの中身からそうなる。形式名でアプリが分岐すると、形式が増えるたびにアプリの改修になる。山形（対戦の親子）を
描くときは、そのときに必要な構造を任意項目として足す（任意項目の追加は `schemaVersion` を上げず、旧アプリは無視する）。

### 表（`tables[]`。2026-10-15 — handball-project#567）

順位表・得点ランキング・星取表のような**大会ごとの表**。大会の一覧ファイル（`competitions/{id}.json`）の任意項目で、
`/v2/index.json` には書かない（`corpus/unexpectedTables`）。

| フィールド | 型 | 必須 | 説明 |
|---|---|---|---|
| `tables[].id` | String | ✓ | 表の識別子（英数字と `-`）。同じ一覧の中で一意（`corpus/duplicateTableID`・`corpus/unsafeTableID`） |
| `tables[].title` | String | ✓ | 表の題。アプリはこれを切替のラベルに出す（例: `"順位表"`） |
| `tables[].columns[]` | Object[] | ✓ | 列の定義。1 列以上（`corpus/tableWithoutColumns`） |
| `tables[].columns[].title` | String | ✓ | 列見出し |
| `tables[].columns[].align` | String? | | `"leading"`（既定）/ `"trailing"` / `"center"`。数値の列は `trailing` |
| `tables[].rows[][]` | String[][] | ✓ | 行。各行のセル数は `columns` の数と一致（`corpus/tableRowWidthMismatch`）。セルは表示する文字列そのまま（数値も文字列で書く） |

- **表は汎用で、アプリは列の意味を知らない。** 配信側が題・列・寄せ・中身を決め、アプリは「見出し行を太字・`align` どおりに寄せ・幅が足りなければ横スクロール」の 1 つの描き方で全部の表を出す。順位表でも得点ランキングでも将来の表でも、アプリの改修なしに足せる。列の意味を型で持つと、表の種類が増えるたびに配信とアプリの両方を直すことになる
- **中身は配信側が自分のデータから計算する。** 計算できないもの（配信していない数値から作る表）は載せない。表の並び（順位の決め方）は作る側の tool が持ち、その規則は tool の README に書く
- 表の順は配列順。アプリは「試合」の後に表の題を並べる。`tables` が無い・空なら試合の一覧だけを出す（今までと同じ画面）

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

### トップレベルの任意フィールド（試合ファイル用）

| フィールド | 型 | 必須 | 説明 |
|---|---|---|---|
| `generator` | Object? | | このファイルを書いたアプリ。`{ "name": String, "version": String, "build": String? }`（例: `{"name": "HandballRecorder", "version": "1.6.1", "build": "29"}`）。**配信物には乗せない**（本リポの JSON は python 生成で、アプリではない）。アプリ間で試合を移す「試合ファイル」（`.handrec`。中身はこの schema と同一）がサポート用に書く — 「外部生成」と「自アプリ + version 差」を切り分けるためのもので、改竄防止ではない（[handball-project#298](https://github.com/kinjo-ryura/handball-project/issues/298) / [#300](https://github.com/kinjo-ryura/handball-project/issues/300)） |

任意フィールドの追加は `schemaVersion` を上げない（読む側の serde は未知のキーを無視するので、旧アプリも読める）。

### `match.configuration` (tagged union)

`kind` discriminator で case を分岐:

| `kind` | サブペイロード | 説明 |
|---|---|---|
| `"timer"` | `timer.phaseDurationSeconds: Double` | タイマーモード (動画なし) |
| `"video"` | `video.source: { provider, externalID, cloudIdentifier?, durationSeconds? }` | 動画モード (フル試合) |
| `"videoHighlight"` | `videoHighlight.source: { provider, externalID, cloudIdentifier?, durationSeconds? }` | ハイライト経路 (内部経路フラグ、 通常 `/v2/highlights/` で使う) |

#### `source`

| フィールド | 型 | 必須 | 説明 |
|---|---|---|---|
| `provider` | String | ✓ | `"youtube"` \| `"local"`。**配信物は `"youtube"` のみ**。`"local"` は試合ファイルだけに現れる（端末「写真」の動画。参照は端末固有で、別の端末では再生できない） |
| `externalID` | String | ✓ | `youtube`: 動画 ID / `local`: PHAsset の localIdentifier |
| `cloudIdentifier` | String? | | `local` のみ。PhotoKit の cloud identifier（端末をまたいで安定）。受け取った端末が同じ Apple ID で iCloud 写真を使っていれば、これをその端末の localIdentifier へ引き直して再生を復旧する。送り側で引けなければ省略（[#300](https://github.com/kinjo-ryura/handball-project/issues/300)） |
| `durationSeconds` | Double? | | `local` のみ。送り側で紐付いていた動画の尺（秒）。受け取った端末で動画を選び直すとき、別の切り出しへ差し替えて記録の時刻が静かにずれるのを止めるために比べる |

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

**場面別の起点**（2026-09-05 に詰めた・[handball-project#148](https://github.com/kinjo-ryura/handball-project/issues/148)）:

| 場面 | 起点 |
|---|---|
| 得点の後 | スローオフが実行された瞬間 |
| キーパーが持った（セーブ / 枠外のどちらでも） | **キーパーのスローが実行された瞬間** |
| ディフェンス側のコートプレーヤーが取った | 取った瞬間 |
| 攻撃側がリバウンドを拾って攻撃が続く | 拾った瞬間に**同じチームの開始をもう 1 件**置く（数える単位はチームの切り替わり回数なので 1 ポゼッションのまま） |

キーパーだけスローに寄せるのは、**セーブと枠外が中継映像で見分けられない**ため（どちらも「キーパーが持って投げる」という同じ絵になる）。フィールドプレーヤーの奪取にこの曖昧さは無いので取った瞬間でよい。**既知の偏り**: セーブの場合、キーパーが保持していた数秒は直前のポゼッション（シュートを外した側）に足される。end を明示すれば消えるが、カメラの向きから検出する生成側は end を出せないので当面は残る。

> **攻撃側のポゼッションはシュートがシューターの手を離れた瞬間に終わる**（上の偏りを消す end の置き方）。ただし根拠になる `shotMissed` を生成側がまだ出せない（スコアボードの読み取りは得点しか拾えない）ため、**`shotMissed` を出せるようになってから効く規則**。それまで end は省いてよい（end は任意）。

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
