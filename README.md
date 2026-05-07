# O6lvl4 Vitals

[O6lvl4-protocol](https://github.com/O6lvl4/O6lvl4-protocol) の14プロトコルへの準拠を **定量的に検査する CLI**。almide で実装。

## What it does

毎日の生活データ（睡眠・食事・運動・呼吸など）と、各プロトコルの不変条件・違反検知ルールを突き合わせ、**どのプロトコルが守られていないか** を出力する。

```
$ vitals check
P01 睡眠
  ✓ INV-1  睡眠時間 ≥ 7h               (7.5h)
  ✗ INV-2  就寝時刻のずれ ≤ ±15分      (drift: 28m)
  ✓ VIO-1  入眠潜時 > 30m が3日連続    (no streak)

P04 食事
  ✓ INV-1  食事時刻ずれ ≤ ±30分
  ✗ INV-5  加工糖飲料禁止              (1 logged)

総合: 12/14 protocols compliant
違反: P01-INV-2, P04-INV-5
推奨復旧手順: protocols/p01-sleep.md#復旧手順, protocols/p04-diet.md#復旧手順
```

## Install

ツールチェーンは [qusp](https://github.com/O6lvl4/qusp) で管理する。`qusp.toml` で almide のバージョンが固定されている。

```bash
# 1. qusp を入れる
brew install o6lvl4/tap/qusp
# (almide backend が必要なので、qusp >= 0.31。古い場合はソースから:
#  cargo install --git https://github.com/O6lvl4/qusp --branch feat/almide-backend --bin qusp)

# 2. このリポを clone
git clone https://github.com/O6lvl4/O6lvl4-vitals.git
cd O6lvl4-vitals

# 3. qusp.toml に書かれた almide を入れる
qusp install

# 4. 実行
qusp run almide run src/main.almd -- check
```

### バイナリビルド（任意）

```bash
qusp run almide build src/main.almd -o vitals
cp vitals ~/.local/bin/
vitals check
```

## Usage

```
vitals init                  Initialize data/ and rules/ directories
vitals log <subcmd> [args]   Log a data point (see `vitals log help`)
vitals show [date]           Show logged data for date (default: today)
vitals check [date]          Evaluate all rules against data
vitals report [--month]      Markdown compliance report for the window (7d / 30d)
vitals rules                 List all loaded rules
vitals help                  Show this message
```

Date format: `YYYY-MM-DD`。

## 日々の使い方

毎日の入力はだいたい合計 **2〜3 分**。記録のタイミングは「ついで」で十分（食事後すぐ、入浴後すぐ、寝る前まとめて、など）。フラグを覚える必要はなく、サブコマンド名だけで対話モードに入る。

### 朝（起床直後 〜 30分以内）

```bash
vitals log morning      # 5項目の対話：就寝/起床/入眠潜時/中途覚醒/質
vitals log brush morning
```

朝の屋外光をすぐ浴びれた日は、外から戻ってきたタイミングで：

```bash
vitals log light --outdoor-morning 15
```

### 食事のたび（朝・昼・夜の3回）

```bash
vitals log meal breakfast    # 時刻 / タンパク質 Y/N / 主食量 S/M/L
vitals log meal lunch
vitals log meal dinner
```

食後に眠気が来たら忘れず夜の `evening` で記録する（`--hunger-crash` フラグや夕方眠気を加点）。

### 日中（都度）

```bash
vitals log water 500                         # 500ml 飲んだ。累積される
vitals log caffeine drip --time 09:30        # 飲料から mg を自動換算
vitals log session cardio --duration 30 --rpe 5    # 運動 1セッション
```

### 夜（就寝前 〜 30分前）

```bash
vitals log bath              # 時刻 / 温度 / 時間
vitals log brush evening
vitals log floss

# 1日の集計
vitals log evening           # 集中時間 + 痛み + 空腹崩壊 + (任意) off-day/rest-day/work-hours/steps
```

### その日の状態を見る

```bash
vitals check                 # 全プロトコルの合格・違反をその場で評価
vitals show                  # 当日 JSON の中身を確認
```

### 週次レビュー（金曜午後 or 日曜夜）

```bash
vitals report                # 直近7日のマークダウンレポート
vitals report --month        # 直近30日
vitals report > weekly.md    # ファイルに残す
```

レポートには：

- 直近7日の **per-protocol pass rate**（priority 順）
- **連続 Pass の streak**（≥3 日のみ）
- **当日の違反**と各 [O6lvl4-protocol](https://github.com/O6lvl4/O6lvl4-protocol) 復旧手順への直リンク
- **Recommended focus**：優先順位が最も高い違反プロトコル

### こうなったら

| 状況 | やる |
|---|---|
| 毎朝の入力が面倒に感じる | フラグ渡しで script 化（[Logging examples](#logging-examples) 参照）して shell alias / Raycast に登録 |
| データを別マシンと共有したい | `export VITALS_DATA=$HOME/Dropbox/vitals` を `~/.zshrc` に |
| 違反検知が連発する | report の **Recommended focus** の最上段（最 upstream の崩れ）から1つだけ復旧手順を実行 |
| 1日入力を忘れた | 翌日に過去日付で `--end <YYYY-MM-DD>` 等は使わず、欠損のまま運用。skip 扱いで violation は出ない |
| 機器（Apple Watch / Oura）を導入する | このリポは手動運用前提のままで OK。スクリプトで `vitals log` に流せばよい |

## Logging examples (full reference)

フラグなしで呼ぶと対話モードに入り、各項目をデフォルト付きで聞かれます（Enter で採用）。フラグを渡せば自動モード。

```bash
# 朝（対話）
vitals log morning
就寝時刻 [23:30]: 23:35
起床時刻 [07:00]:               # Enter でデフォルト
入眠まで(分) [10]:
中途覚醒(回) [0]:
睡眠の質(1-5) [4]:
→ data/2026-05-07.json

# 朝（自動 / シェルスクリプトから）
vitals log morning --bedtime 23:35 --wake 07:05 --latency 10 --wake-count 0 --quality 4
vitals log light --outdoor-morning 15 --outdoor-total 30
vitals log brush morning

# 食事のたびに
vitals log meal breakfast --time 07:30 --protein-ok --carb M
vitals log meal lunch     --time 12:30 --protein-ok --carb S
vitals log meal dinner    --time 19:00 --protein-ok --carb M

# 水分・カフェインは都度
vitals log water 500            # 累積で記録される
vitals log caffeine drip --time 09:30   # mg は飲料種別から自動換算

# 運動
vitals log session cardio   --duration 30 --rpe 5 --time 07:30
vitals log session strength --duration 40 --rpe 7 --kind A

# 夜
vitals log bath --time 21:00 --temp 40 --duration 12
vitals log brush evening
vitals log floss
vitals log evening --focus 220
```

各コマンドは `data/<today>.json` を読み込んで該当フィールドだけ更新するので、順序や回数は自由。

## Data model

### Storage paths

`data/` と `rules/` の場所は次の順で解決されます（最初に見つかったものを使用）：

1. 環境変数 `$VITALS_DATA` / `$VITALS_RULES`（明示的オーバーライド）
2. cwd の `./data` / `./rules`（リポジトリ内で動かすとき）
3. `$HOME/.local/share/vitals/{data,rules}`（デフォルトのユーザストア）

これにより、リポジトリの外から `vitals` バイナリを実行してもデータは `~/.local/share/vitals/data/` に蓄積されます。複数マシンで同期したい場合は `VITALS_DATA=$HOME/Dropbox/vitals` のように指定すれば即座に切り替え可能。

### 日次データ: `<data_dir>/YYYY-MM-DD.json`

```json
{
  "date": "2026-05-07",
  "sleep": {
    "bedtime": "23:35",
    "wake_time": "07:05",
    "duration_h": 7.5,
    "onset_latency_min": 10,
    "wake_count": 0,
    "subjective_quality": 4
  },
  "subjective": {
    "mood": 4,
    "focus_forecast": 4,
    "pain_flag": false
  },
  "meals": {
    "breakfast": { "time": "07:30", "protein_ok": true, "carb_size": "M" },
    "lunch":     { "time": "12:30", "protein_ok": true, "carb_size": "S" },
    "dinner":    { "time": "19:00", "protein_ok": true, "carb_size": "M" }
  },
  "hydration": { "water_ml": 1500, "caffeine_mg": 190 },
  "caffeine": [
    { "time": "09:30", "type": "drip" },
    { "time": "12:30", "type": "drip" }
  ],
  "sessions": [
    { "kind": "cardio", "duration_min": 30, "rpe": 5, "time": "07:30" }
  ],
  "bath": { "time": "21:00", "temp_c": 40, "duration_min": 12 },
  "oral": { "brush_morning": true, "brush_evening": true, "floss": true },
  "deep_work_min": 220,
  "outdoor_light_min": 25
}
```

すべて optional。記録できなかった項目は欠損扱いで evaluator 側が許容する。

### ルール: `rules/<protocol>.json`

```json
{
  "protocol": "P01",
  "name": "睡眠",
  "rules": [
    {
      "id": "P01-INV-1",
      "type": "invariant",
      "description": "睡眠時間 ≥ 7時間",
      "metric": "sleep.duration_h",
      "operator": "gte",
      "threshold": 7,
      "window": "1d",
      "recovery_ref": "p01-sleep.md#復旧手順"
    }
  ]
}
```

## Phase 1 スコープ

| 機能 | 状態 |
|---|---|
| ルール定義（14プロトコル分） | JSON 80ルール定義済み |
| `vitals init` / `show` / `check` / `rules` | 実装済み |
| `vitals log <subcmd>` フラグベース入力 | 実装済み（10サブコマンド） |
| 単日 op (gte / gt / lte / lt / eq / time_drift_lte / bool_eq) | 実装済み |
| `Nd_consecutive` window | 実装済み |
| `weekly_count_lte` / `weekly_count_gte` | 実装済み |
| `rolling_avg_drop_pct`（前週比トレンド） | 実装済み |
| `monthly` window | 実装済み |
| `baseline_pct_lte`（HRV ベースライン） | 機器導入後 |
| iPhone Health 自動取り込み | Phase 3 |
| RescueTime 連携 | Phase 3 |
| 違反トレンドダッシュボード / 朝の通知 | Phase 3 |

## Roadmap

- **Phase 1** (完了): ルール定義 + 評価器 + `vitals log` フラグ入力
- **Phase 2** (完了): 多日 window オペレータ（consecutive / weekly_count / rolling_avg_drop / monthly）
- **Phase 3**: iPhone Health 取り込み + RescueTime 連携 + Web ダッシュボード + 朝の通知

## License

MIT
