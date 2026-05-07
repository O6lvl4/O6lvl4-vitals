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
vitals rules                 List all loaded rules
vitals help                  Show this message
```

Date format: `YYYY-MM-DD`。

### Logging examples

```bash
# 朝、起きてすぐ
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

### 日次データ: `data/YYYY-MM-DD.json`

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
  "meals": [
    { "type": "breakfast", "time": "07:30", "protein_ok": true,  "carb_size": "M" },
    { "type": "lunch",     "time": "12:30", "protein_ok": true,  "carb_size": "S" },
    { "type": "dinner",    "time": "19:00", "protein_ok": true,  "carb_size": "M" }
  ],
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
