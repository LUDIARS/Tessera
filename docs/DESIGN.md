# Tessera 設計書

> 版: 0.1 (初稿) — 2026-04-23
> 著者: kazmit299
> ステータス: ドラフト (実装未着手)

---

## 1. 命名

**Tessera** — ローマ時代の小さな方形片で、以下の役割を一つのモノが兼ねていた:

1. **Tessera militaris** — 軍の合言葉を刻んだトークン。持っていることが身元証明。
2. **Tessera lusoria** — ボードゲームやサイコロ遊戯の駒・札。
3. **Tessera hospitalis** — 客人識別標。二つに割って片方ずつ保持し、再会時に合わせて本人確認する (現代の暗号学的 challenge-response に近い)。

ed25519 真性認証を持つゲーム通信基盤の性格と完全に一致する。

## 2. 位置付けとスコープ

### 2.1 Synergos との関係

Tessera は Synergos の網から **トランスポート下層だけを流用** し、アプリ層を書き換える。

```
┌─────────────────────────────────────────┐
│  Game / Application layer               │  ← 新規 (Tessera)
│  (tick sync, input, RPC, lobby)         │
├─────────────────────────────────────────┤
│  Session / Security                     │  ← 流用 + 薄い追加
│  (ed25519 identity, HLO1 hello)         │
├─────────────────────────────────────────┤
│  Transport                              │  ← Synergos からそのまま
│  QUIC + STUN/TURN + relay (WS→UDP)      │
└─────────────────────────────────────────┘
```

`synergos-net` の中で `mesh/`, `tunnel/`, `relay/`, QUIC 基盤 (`transport/`, `session/`) は再利用対象。`chain/`, `content/`, `catalog/`, `gossip/` は Tessera では **不使用**。

### 2.2 スコープ

| in scope | out of scope |
|---|---|
| 2–32 人規模のリアルタイムゲーム通信 | MMO 規模の interest management |
| authoritative server トポロジ | 完全 P2P (cheat を本気で抑えたい用途) |
| peer-to-peer mesh (小規模対戦・協力) | サーバサイドゲームロジック (別プロジェクト) |
| モバイル (iOS / Android) 最適化 | マッチメイキング (別サービス想定) |
| lockstep 型 / snapshot 型 両対応 | 物理演算・ゲームエンジン |

### 2.3 非目標

- **マッチメイキング** はしない (Tessera は room 到達後の通信のみ担う)。
- **ゲームロジック** を書かない (ライブラリとして呼ばれる)。
- **アンチチート** は authoritative server + ed25519 署名入力に留める。行動分析は範囲外。

## 3. 設計目標 (優先度順)

1. **P50 < 50 ms / P99 < 150 ms** (同リージョン LTE → relay → LTE)。
2. **接続断 3 秒以内で自動復帰** — Wi-Fi / LTE ローミング中もセッション維持。
3. **電池と通信量** — アイドル tick は 1 Hz まで落とす。5 分静止で suspend に降格。
4. **Symmetric NAT で必ず繋がる** — モバイルキャリア NAT 前提のため TURN/UDP relay 必須。
5. **ed25519 真性認証** — 入力に署名、rollback/replay 不能。
6. **crate/言語移植性** — Rust コアを FFI で Unity / Godot / モバイルから呼べる。

## 4. プロトコル設計

### 4.1 Stream magic

Synergos 既存 (`HLO1` / `DHT1` / `TXFR` / `GSP1` / `BSW1`) と衝突しない 4 byte を追加:

| magic | 用途 |
|---|---|
| `TSR1` | Tessera 制御ストリーム (QUIC bidi)。join / leave / ready / room 状態 |
| `TSRI` | Input stream (QUIC uni, client→server)。入力を署名付きで流す |
| `TSRS` | Snapshot / delta stream (QUIC uni, server→client)。権威状態の配信 |
| `TSRR` | Reliable RPC (QUIC bidi)。装備変更やチャットなど遅延許容イベント |

datagram (unreliable) は QUIC DATAGRAM 拡張 (RFC 9221) を使い、`TSR1` ハンドシェイク後に `stream_id` なしで流す。

### 4.2 パケット分類

| 種別 | 方向 | 保証 | 典型サイズ | 頻度 |
|---|---|---|---|---|
| Input | C→S | 期限付き信頼 | 16–64 B | 30–60 Hz |
| Snapshot (full) | S→C | reliable stream | 1–4 KB | ≤ 1 Hz |
| Delta | S→C | unreliable datagram | 100–400 B | 20–30 Hz |
| RPC | 両方向 | reliable stream | 任意 | 稀 |
| Ping | 両方向 | unreliable | 12 B | 2 Hz |

### 4.3 期限付き信頼性 (input)

- クライアントは入力に `(tick, seq, sig)` を付けて送信。
- サーバは受信した最大 seq までの入力をバッファ。古い入力は破棄 (late → drop)。
- 連続欠落 8 tick で client に **resync request** を送り、full snapshot で復帰。

### 4.4 Delta 圧縮

- サーバは各クライアントごとに `last_acked_snapshot_id` を保持。
- その snapshot からの差分のみ送信 (bit-packed field mask + varint)。
- ack は datagram で piggyback (入力パケットに `(last_seen_snapshot_id)` を付ける)。

### 4.5 Tick / interpolation

- サーバ tick 30 Hz 既定 (ゲーム側で 20 / 30 / 60 切替可能)。
- クライアントは 100 ms interpolation buffer (= 3 tick)。
- 入力は client-side prediction → サーバ権威 → reconciliation。

### 4.6 シリアライザ

bincode は使わない (スキーマレスで破壊的変更が起こる)。候補:

- **`postcard` + serde** — 組込み向け varint、小さい、Rust エコシステム親和性高。
- 手書き bit-packed (delta / input のみ) + postcard (制御系)。

→ 既定: **postcard for control, 手書き bit-pack for hot path**。

## 5. トポロジ

### 5.1 Authoritative server (既定)

- サーバは **relay と同居可能** (`tessera-server` crate)。
- セルフホスト、VPS、あるいは Synergos 既存の cloudflared tunnel 経由で LAN 公開も可。
- ed25519 で身元認証したクライアントのみ join。

### 5.2 Peer-to-peer mesh (オプション)

- ≤ 8 人の対戦・協力向け。
- 1 人が **host 権** を持つ (deterministic 選出: PeerId 最小)。
- host が落ちたら次席が即座に引き継ぐ (state は全員が直近 snapshot を保持)。

### 5.3 Relay 経由接続フロー

```
C ─── STUN probe ──→  type 判定
│
├─ Full cone / restricted → 直接 QUIC (hole punch)
│
└─ Symmetric NAT (LTE 典型)
       └─→ TURN (UDP relay) ─→ Server
              ↑ Synergos Issue #23 完遂が前提
```

フォールバック順: 直接 UDP → TURN UDP → TURN TCP → WS relay (synergos-relay)。

## 6. モバイル最適化

### 6.1 QUIC connection migration

- Wi-Fi ↔ LTE 切替時、QUIC CID を保持したまま endpoint を差し替える (RFC 9000 §9)。
- PATH_CHALLENGE/RESPONSE で新経路検証。
- セッションは切れない (信頼性高)。

### 6.2 バッテリー / 通信量プロファイル

| 状態 | tick | 備考 |
|---|---|---|
| active | 30–60 Hz | ゲーム中 |
| background (アプリ裏) | 2 Hz | keep-alive + RPC のみ |
| suspend (5 分静止) | 0 | 次回 foreground で resume |
| low-power (OS 要求) | active/2 | tick 半減 |

### 6.3 再接続戦略

- 切断検出後 exponential backoff: 0.25 / 0.5 / 1 / 2 / 4 s。
- 3 s 以内に復帰できれば snapshot 再送なしで続行。
- 超過は full snapshot 要求。

### 6.4 データ通信量目安

- 入力 32 B × 30 Hz × 2 方向 = **~2 KB/s** (上り)。
- Delta 300 B × 30 Hz = **~9 KB/s** (下り)。
- 1 時間プレイ ≈ **40 MB**。モバイルデータでも 1 日数時間は許容範囲。

## 7. セキュリティ

| 脅威 | 対策 |
|---|---|
| 入力なりすまし | ed25519 署名 (Synergos Identity 流用) |
| Replay | `(tick, seq)` 単調増加、サーバ側で重複 drop |
| Room 侵入 | サーバが招待トークン (tessera hospitalis 文字通り) を検証 |
| 改竄クライアント | authoritative server → クライアント状態は信用しない |
| MITM | QUIC TLS 1.3 + peer_id pinning (Synergos HLO1 の再利用) |
| DoS (relay) | room ごとの帯域上限、送信者ごと token bucket |

**アンチチート**: Tessera は署名と権威サーバまで。挙動分析は上位レイヤーの責務。

## 8. Crate 構成 (想定)

Synergos の命名規約に合わせる。

| crate | 役割 |
|---|---|
| `tessera-proto` | ワイヤプロトコル型・シリアライズ (no_std 可) |
| `tessera-net` | QUIC + relay + NAT traversal (synergos-net の薄いラッパ) |
| `tessera-server` | authoritative server。room 管理 + tick ループ |
| `tessera-client` | クライアントライブラリ。prediction / reconciliation |
| `tessera-mesh` | P2P mesh モード (≤ 8 人) |
| `tessera-ffi` | C ABI。Unity / Godot / iOS / Android 向け |
| `tessera-sim` | ヘッドレス負荷試験 (N クライアント模擬) |
| `tessera-demo` | Bevy ベースの最小対戦デモ |

## 9. 依存関係

| 使う | 用途 |
|---|---|
| `quinn` | QUIC 実装 (Synergos と同じ) |
| `ed25519-dalek` | 署名 |
| `postcard` + `serde` | 制御系シリアライズ |
| `bevy_ecs` (optional) | デモ |
| `tokio` | ランタイム |

Synergos の `synergos-net` から **再公開する抽象** を受け、直接依存はしない (循環防止)。

## 10. テスト戦略

- **unit**: protocol エンコード/デコード round-trip (既存 Synergos パターン踏襲)。
- **integration**: `tessera-sim` で N=100 クライアント × 60 秒、損失 5% / 遅延 100ms ± 30ms でサーバ負荷検証。
- **NAT matrix**: Symmetric × Full cone 組合せを Docker で再現 (Synergos の既存 fixture 流用)。
- **mobile soak**: 実機 iOS / Android で Wi-Fi ⇄ LTE 切替を 30 分繰り返し、切断回数 0 を目標。

## 11. マイルストーン

| M | 内容 | 依存 |
|---|---|---|
| **M0** | `tessera-proto` + `tessera-net` スケルトン、ping が通る | — |
| **M1** | authoritative server + 2 client、input/snapshot が流れる | M0 |
| **M2** | delta compression + client prediction | M1 |
| **M3** | TURN フォールバック統合 | Synergos #23 |
| **M4** | P2P mesh モード (≤ 8 人) | M2 |
| **M5** | モバイル FFI + 実機テスト | M3 |
| **M6** | Bevy demo 公開 | M4 / M5 |

## 12. 未決事項

- [ ] シリアライザの hot path 実装方式 (手書き vs bitcode vs rkyv)。ベンチで決める。
- [ ] P2P mesh の lag compensation モデル (lockstep か rollback か)。
- [ ] 複数 region にまたがる relay の選択アルゴリズム (static config か動的 RTT か)。
- [ ] FFI の ABI 形式 — C ABI 直か、UniFFI か。モバイル側の開発体験で決める。
- [ ] mesh モードでの host migration 中に入力を欠落なく引き継ぐ方法。

## 13. 参考

- RFC 9000 (QUIC) §9 Connection Migration
- RFC 9221 (QUIC Datagram Extension)
- Overwatch GDC 2017 "Networking Scripted Weapons and Abilities"
- Glenn Fiedler, "Networked Physics" シリーズ
- Valve Source Multiplayer Networking (lag compensation の原型)
- Synergos 内部設計 (`synergos-net/src/lib.rs`, `synergos-core/src/daemon.rs`)
