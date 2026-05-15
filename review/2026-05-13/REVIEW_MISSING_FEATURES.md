# REVIEW_MISSING_FEATURES — Tessera

- 対象: LUDIARS/Tessera (E:/Document/Ars/Tessera)
- 日付: 2026-05-13
- 実装: **未着手** (全 8 マイルストーン M0–M8 未着手)
- 評価: **C** (実装が一切無いため低評価必須、 ただし設計書では機能網羅性は良好)

## 総評

実装はゼロなので、 マイルストーン §11 (docs/DESIGN.md:737-747) に列挙された M0–M8 全てが「未実装機能」。 ただし設計書 v0.7 (docs/DESIGN.md:1-8) には機能の網羅性は高く、 「**何が無いか**」というよりは「**何が未着手か**」のリスト化が中心となる。 本レビューでは (A) マイルストーン未着手機能、 (B) 設計書に書かれていない機能ギャップ、 (C) §12 未決事項からくる暗黙の未実装を分けて列挙する。

## A. マイルストーン未着手機能 (docs/DESIGN.md:737-747)

| M | 内容 | ブロッカー |
|---|---|---|
| M0 | `tessera-proto` + `tessera-net` skeleton, ping | なし。 着手可 |
| M1 | authoritative server + 2 client, input/snapshot 流れる | M0 |
| M2 | delta compression + client prediction | M1 |
| M3 | TURN 経路統合 (self-hosted or Cloudflare) | Synergos #23 OR Cloudflare 契約 |
| M4 | P2P mesh (≤8人) + 決定論 sim + rollback + arbiter migration | M2 |
| M5 | モバイル FFI + 実機テスト | M3 |
| M6 | Bevy demo 公開 | M4/M5 |
| M7 | Codex 台帳 | M1 (現状は **LUDIARS/Codex 別リポへ移行中**, docs/DESIGN.md:411-412) |
| M8 | 新興網プロファイル検証 (RTT 400ms + loss 15% 下の Codex 収束 + T3 安定性) | M3 / M7 |

## B. 設計書に書かれていない機能ギャップ

設計書 v0.7 を読み終えた時点で、 「ゲーム通信基盤として当然欲しい」 が現状未記述の項目:

### B.1 Lobby / Matchmaking との接続境界

- §2.3 (docs/DESIGN.md:60-63) で「マッチメイキングはしない」「room 到達後のみ」とスコープ明示。 ただし **「room 到達」のプロトコル境界が未記述**。 Lobby が Tessera に渡すべき情報 (peer 一覧・ed25519 公開鍵・tier 確定値・managed/self-hosted の transport 選択) が `JoinSessionParams` のような型として定義されていない。 → §5.6.5 (docs/DESIGN.md:402-408) で「tier は lobby が確定」 とあるので、 そこに 1 節追加すべき。

### B.2 Replay / 観戦

- §5.7.8 (docs/DESIGN.md:597) で「T1 Duel: 勝敗 replay 署名」、 §5.7.11 (docs/DESIGN.md:646) で「replay 保存時のみ ~100 KB/match」と触れているが、 **replay フォーマットが未定義**。 入力 stream + snapshot stream を録画する方式か、 Codex entry の列を保存して決定論再生する方式か、 ハイブリッドかが未決。
- 観戦モード (live spectate) の帯域・遅延予算も未記述。 §5.6.4 T4 (docs/DESIGN.md:399) に「死亡後のカメラは低 tick 観戦ストリームで帯域節約」とだけ書かれているが、 仕様化が必要。

### B.3 Voice chat / VoIP

- ゲーム通信基盤としてしばしば併設される voice。 §2.2 (docs/DESIGN.md:51-58) で「ゲームエンジン・物理は out of scope」 とあるが、 voice についての言及が無い。 設計判断として「Voice は別チャネル (例: Discord SDK) を呼ぶ」 か「QUIC DATAGRAM で opus を流す」 かの **明示的な non-goal 宣言が必要**。

### B.4 Metric / Observability

- Excubitor (LUDIARS の可観測性) との接続が未記述。 §10 テスト戦略 (docs/DESIGN.md:728-734) は test harness の話だが、 production での RTT / loss / tier downgrade などの metric 出力が無い。
- 最低限: `tessera-server` が prometheus exposition format で room 数・接続数・tick lag を出すべき。 §8 crate 構成 (docs/DESIGN.md:702-712) に `tessera-metrics` を追加するか、 `tessera-server` 内部で完結させるか要決定。

### B.5 Hot reload / config 変更

- Cloudflare 障害時に self-hosted へ fail-over (§5.6.3 docs/DESIGN.md:373) が想定されているが、 **稼働中 session を維持したまま transport を切り替えるか、 session 終了させるか** が未記述。 fail-over 中の UX (1 秒停止 / 黒画面 / 続行) を決める必要あり。

### B.6 多 region / latency-aware relay 選択

- §12 (docs/DESIGN.md:752) で「複数 region にまたがる relay の選択アルゴリズム」 が未決。 設計書本文には GeoDNS / anycast / latency probe のどれを使うかの議論が無い。 T3-T4 で東京 PoP / シンガポール PoP を選び分ける運用は必須。

### B.7 Bot / AI 充填 (drop-in replacement)

- §5.6.4 T2 (docs/DESIGN.md:386) で「脱落設計 — 誰かが落ちた時の進行 (AI 代替 / 即終了 / 待ち) をゲームが決める」 と書かれているが、 **Tessera が AI 代替の input 注入をサポートするか否か** が未明示。 サポートしないなら non-goal、 サポートするなら署名された「bot peer」 の扱いが必要。

### B.8 IPv6 prefix delegation との関係

- §5.3.1 (docs/DESIGN.md:174-188) で IPv6 inbound 遮断の現実を分析したが、 **キャリア側 prefix が短時間で変わる場合** (再接続で /64 が変わる) の挙動が未記述。 connection migration (§6.1, docs/DESIGN.md:653-657) が prefix 変化も吸収する想定なのか明示すべき。

## C. §12 未決事項 (docs/DESIGN.md:749-763) → 暗黙の未実装

12 項目あり、 そのうち M0-M3 をブロックする可能性のあるもの:

1. シリアライザ hot path 実装方式 — M0 着手で決める必要
2. M3 で self-hosted TURN を既定とするか Cloudflare managed を既定とするか — M3 ブロッカー
3. FFI の ABI 形式 (C ABI 直 / UniFFI) — M5 ブロッカー
4. mesh プロファイル決定論実装方式 (fixed-point / 厳密 float) — M4 ブロッカー
5. Codex 永続化先 — M7 ブロッカー (ただし Codex 別リポへ移行で Tessera 側の決定範囲外になる可能性)

## 優先度付き推奨アクション

**Must (M0 着手前)**
- §8 から `tessera-codex` を削除 → `tessera-codex-client` 化
- §12 シリアライザ決定 + QUIC config 数値表追加

**Should (M3 着手前)**
- B.1 JoinSessionParams 仕様化
- B.4 Excubitor 連携設計
- §12 self-hosted vs managed 既定の判断

**Nice-to-have (M6 demo まで)**
- B.2 Replay フォーマット
- B.3 Voice の明示的 non-goal 宣言
- B.6 多 region relay 選択

## 結論

設計の網羅性は高いが、 M0 着手前にケリをつけるべき「**ゲーム基盤の外周仕様**」 (lobby boundary / replay / observability / voice の non-goal 宣言) が薄い。 Codex 別リポ移行の決定 (970590f) で本体の機能集合は確定したので、 上記 7 件を §5.6.5 / §10 / §12 に追記すれば「設計完了 → 実装可」の signal を出せる状態。
