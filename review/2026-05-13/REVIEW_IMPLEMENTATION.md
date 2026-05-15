# REVIEW_IMPLEMENTATION — Tessera

- 対象: LUDIARS/Tessera (E:/Document/Ars/Tessera)
- 日付: 2026-05-13
- 実装: **未着手** (`git ls-files` は .gitignore / README.md / docs/DESIGN.md の 3 ファイルのみ)
- 評価: **N/A (Tentative C)**

## 総評

実装コードがゼロのため、 通常の意味での実装レビュー対象は無い。 評価は「実装着手時に M0–M8 (docs/DESIGN.md:738-747) を実行できる準備が整っているか」 という観点で行う。 結論としては **設計 → 実装の翻訳に必要な最低限の足場 (crate 名・依存・命名規約) は揃っているが、 着手前に詰めるべき具体仕様 (シリアライザ確定・FFI 形式・workspace 構成) が §12 未決事項 (docs/DESIGN.md:751-763) に残っており、 M0 が「動かしながら決める」式になりやすい**。

## 現状

- リポジトリ実体 (docs/DESIGN.md:0):
  - `.gitignore` — Rust + IDE ノイズの基本パターン (7 行)
  - `README.md` — 21 行。 位置付けと Synergos との比較表
  - `docs/DESIGN.md` — 776 行。 v0.7 ドラフト
- ブランチ: main のみ (推定、 git log 上は直列)
- 直近コミット (docs/DESIGN.md:1):
  - `970590f` Add Codex immediate-query path; pause Tessera to extract Codex
  - `14721a6` Add §5.7 Codex — deferred authoritative rights ledger
  - `5471005` Add §5.6 player-count tiers (T1–T4)
- 依存: Cargo.toml 未作成
- CI: ワークフロー未設定

## 設計 → 実装の翻訳に必要な準備

### M0 (`tessera-proto` + `tessera-net` skeleton, ping 通る) に必要なもの

| 項目 | 設計書での所在 | 状態 |
|---|---|---|
| Stream magic 一覧 | §4.1 (docs/DESIGN.md:91-104) | 確定済 |
| パケット分類 | §4.2 (docs/DESIGN.md:107-114) | 確定済 |
| Hello プロトコル | §2.1 + §7 (docs/DESIGN.md:30-47, 693) | Synergos HLO1 流用 = 既存実装あり |
| QUIC 設定 (idle timeout, max streams) | 未記述 | **不足** |
| Cargo workspace 構成 (8 crate) | §8 (docs/DESIGN.md:702-712) | 名前だけ確定、 dep graph 未記述 |
| シリアライザ | §4.6 (docs/DESIGN.md:134-140) | 既定 postcard + 手書き bit-pack まで決定、 hot path 詳細は §12 未決 |

→ M0 着手前に「**QUIC config 数値表**」 (idle timeout / max_concurrent_streams / 0-RTT 可否 / keep-alive interval) を §6 か §4 に追加すべき。

### M1 (authoritative server + 2 client, input/snapshot 流れる) に必要なもの

| 項目 | 状態 |
|---|---|
| Snapshot エンコード (full vs delta の境界条件) | §4.3 / §4.4 (docs/DESIGN.md:117-130) で方針はあるが、 field mask の bit 幅・varint 上限など実装数値が未記述 |
| Tick scheduler (server 30Hz 既定, jitter 吸収) | §4.5 (docs/DESIGN.md:128-132) で hand-wave、 実装方式未記述 |
| Client-side prediction の rollback 境界 | §5.4.1 (docs/DESIGN.md:236-247) で reconciliation 方針あり |
| Room state machine (creating / waiting / playing / ended) | 未記述 |

→ Room state machine の状態遷移図が無い。 §5.5 か §11 に追加すべき。

### M3 (TURN) と M5 (FFI) は外部依存ブロック

- **M3**: Synergos #23 (TURN 完遂) か Cloudflare 契約のどちらか。 M3 マイルストーン (docs/DESIGN.md:742) が「どちらか一方でクリア」になっており、 ここは設計判断済。 ただし §12 (docs/DESIGN.md:756) で「self-hosted を既定とするか managed を既定とするか」 が未決。 M3 着手前に判断必要。
- **M5**: FFI 形式 (C ABI 直 vs UniFFI) が §12 (docs/DESIGN.md:753) 未決。 UniFFI なら Kotlin/Swift binding が自動生成できるが、 Synergos と統一すべきか別かを決める必要あり。

## Codex 切り出しの影響

`970590f Add Codex immediate-query path; pause Tessera to extract Codex` で **Codex は別リポへ移動**。 これは Tessera 実装に以下を要求する:

1. **`tessera-codex` crate 名を §8 から削除** (docs/DESIGN.md:709)、 `tessera-codex-client` (LUDIARS/Codex への bind 専用) に置換。
2. **§5.7 全体 (docs/DESIGN.md:410-650)** を Codex リポ移管後に「30 行要約 + API 境界規約」に圧縮。
3. M7 マイルストーン (docs/DESIGN.md:746) を「Codex 統合 (LUDIARS/Codex への dependency)」に書き換え。

これらは Codex リポが固まってから一括で行うのが効率的で、 現状の pause 状態は妥当。

## 着手再開時の Day 1 タスク (推奨)

1. `Cargo.toml` workspace (8 crate scaffold)。 各 crate は `lib.rs` のみ + `pub fn version() -> &'static str`。
2. `tessera-proto/src/wire.rs` で Stream magic enum + パケット種別 enum を定義 (postcard derive)。
3. `cargo test` + `cargo clippy -- -D warnings` を CI に追加 (Synergos の `.github/workflows/ci.yml` を流用)。
4. README に「現状は M0 着手」を追記し、 ステータス節 (docs/DESIGN.md:22-24) を更新。
5. `cargo deny check advisories` を CI に追加 (Synergos と同条件)。

## 結論

実装が一切無いので評価は **N/A 扱い**だが、 設計から M0 への翻訳準備としては「**やれば 1 週間で M0 雛形は組める**」レベル。 ただし pause 中の §5.7 圧縮と §12 のうち M3 / M5 に関わる 2 項 (managed 既定 / FFI 形式) を着手前に決めると、 後戻りが減る。
