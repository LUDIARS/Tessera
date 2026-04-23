# Tessera 設計書

> 版: 0.4 — 2026-04-23
> 著者: kazmit299
> ステータス: ドラフト (実装未着手)
>
> **変更履歴**
> - 0.4 (2026-04-23) — Foreground-only 前提を §2.4 に明文化。§5.3.4 managed relay (Cloudflare Realtime) 追加。§6.2 tick 階層を簡略化、§6.3 再接続戦略を route migration 内に閉じる
> - 0.3 (2026-04-23) — §5.3 を全面書き直し: 6 段フォールバック + mesh の peer-relay 特典 + 日本モバイル IPv6 現実 + RFC 4941 取扱
> - 0.2 (2026-04-23) — §5 に同期戦略 / 権威的決定の節を追加。§12 の mesh lag compensation と host migration を確定に移動
> - 0.1 (2026-04-23) — 初稿

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

### 2.4 前提: foreground-only 運用

Tessera は **「アプリが画面にある = セッション生存、背景に落ちる = セッション終了」** という明示的モデルを採る。

**帰結**:

- iOS の background 制限・Android の doze / app standby を考慮しない。
- 通信中の背景遷移は **graceful disconnect** として扱う (pause 状態を broadcast → セッション離脱)。
- 画面復帰時は **新規接続 + full snapshot** で再参加。短時間の再接続を特別扱いしない。
- Push 通知は「招待」「マッチ成立」「リマッチ依頼」など **セッション外** イベントに限定。realtime keepalive には使わない。
- VoIP / VPN extension の workaround は採用しない。

この前提で §6.2 tick 階層と §6.3 再接続戦略が大幅に簡略化される。

## 3. 設計目標 (優先度順)

1. **P50 < 50 ms / P99 < 150 ms** (同リージョン LTE → relay → LTE)。
2. **画面復帰時の再参加が 2 秒以内** — 新規接続 + snapshot 取得までを含めた UX 予算。
3. **session 中の route migration** — Wi-Fi ⇄ LTE 切替で体感断絶ゼロ。QUIC connection migration で吸収。
4. **Symmetric NAT で必ず繋がる** — モバイルキャリア NAT 前提のため TURN/UDP relay 必須。
5. **ed25519 真性認証** — 入力に署名、replay 不能。
6. **crate/言語移植性** — Rust コアを FFI で Unity / Godot / モバイルから呼べる。
7. **foreground 時の電池/通信量** — active 30–60 Hz で 40 MB/h 以下。背景は §2.4 により対象外。

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

## 5. トポロジと同期戦略

### 5.1 Authoritative server (既定)

- サーバは **relay と同居可能** (`tessera-server` crate)。
- セルフホスト、VPS、あるいは Synergos 既存の cloudflared tunnel 経由で LAN 公開も可。
- ed25519 で身元認証したクライアントのみ join。

### 5.2 Peer-to-peer mesh (オプション)

- ≤ 8 人の対戦・協力向け。
- 1 人が **host 権** を持つ (deterministic 選出: PeerId 最小)。
- host が落ちたら次席が即座に引き継ぐ (state は全員が直近 snapshot を保持)。

### 5.3 接続経路とフォールバック

経路候補を ICE 的に並列試行し、最初に成立したものを採用する。

```
1. IPv6 直接         — 両側が inbound 許可された v6 を持つ時のみ成立
2. IPv4 直接 (STUN)  — 双方が Full cone / Restricted / Port-restricted NAT
3. Peer-relay        — mesh プロファイル限定。到達可能な第三 peer 経由
4. TURN UDP          — Synergos #23 完遂が前提
5. TURN TCP          — UDP 閉塞網・企業 firewall 対策
6. WS relay          — synergos-relay。最終手段
```

- Symmetric NAT (モバイル LTE 典型) 同士は 2 が失敗し、3 または 4 に落ちる。
- **モバイル 2 人対戦** (格闘など) は第三 peer が居ないため 3 が存在せず、4 が実質 mandatory。これが #23 を非交渉的依存として扱う理由。
- **3 人以上の mesh** で誰か 1 人でも reachable なら、その peer が TURN 代替になり外部依存を減らせる — mesh プロファイルの大きな利点。

#### 5.3.1 IPv6 直接到達性の現実 (日本モバイル、2026-04 時点)

| キャリア | v6 割当 | inbound 許可 | 実用直接 P2P 成功率 |
|---|---|---|---|
| docomo (4G/5G) | ほぼ全契約 | 原則 filter | 低 (ほぼ不可) |
| au / KDDI | 5G 全 / 4G 一部 | 原則 filter | 低 |
| SoftBank | 5G 中心 | 非一貫 | 低 |
| 楽天モバイル | native v6 | filter | 低 |
| 主要 MVNO | v4-only 多数 | — | 不可 |

**結論**:

- v6 割当は進んだが、**inbound はキャリア firewall で概ね遮断**される。mesh VPN (Tailscale 等) 運用データとも整合。
- Tessera の IPv6 取扱は **「直接 P2P の主経路」ではなく、TURN / relay への outbound を CGNAT44 を経ずに流すための品質向上チャネル**。RTT が 10–30 ms 稼げる可能性があるので並列試行はする。
- mesh プロファイルで全員モバイルの場合、1 は並列試行するが **成功を前提に設計しない**。テスト時も通らないケースを既定とする。

#### 5.3.2 IPv6 privacy address (RFC 4941) の扱い

モバイル端末は temporary address を ~24 h で回す。長時間セッションは QUIC connection migration (§6.1) で差替えを追従する。端末が RFC 7217 の stable privacy address を優先する設定なら migration 不要。

#### 5.3.3 経路選択の運用ポリシー

- 試行順は並列だが **採用順は `RTT + jitter × 2`** で最小のものを選ぶ。直接経路が通っても jitter が酷いと TURN より UX が悪いことがある。
- セッション中に上位経路が復活 (例: v6 inbound が突然通った) した場合、**最低 30 s 安定してから** migration。チャタリング防止。

#### 5.3.4 Managed relay オプション (Cloudflare Realtime)

self-hosted インフラを持たない場合の代替として、Cloudflare Realtime 系製品を tier 4–6 に差し込める。§2.4 の foreground-only 前提により WebSocket 経路も第一級の選択肢になる。

| 自前 tier | managed 置換 | 評価 |
|---|---|---|
| 4. TURN UDP | **Cloudflare Realtime TURN** | 最有力。~$0.05/GB、Tokyo PoP まで ~10 ms。Symmetric NAT 対策が managed で完結 |
| 5. TURN TCP | 同上 (TCP 設定) | 同上 |
| 6. WS relay (synergos-relay) | **Workers + Durable Objects** | 1 room = 1 DO。outbound WS のみでインフラゼロ。foreground-only 前提で常時接続の弱点が消える |
| — | Realtime Calls (WebRTC SFU) | **非採用**。QUIC 本線と二重実装 (DTLS/SCTP) になるため別プロファイル扱い |

**Transport 抽象化**: `tessera-net` は tier ごとに transport trait を定義し、self-hosted / Cloudflare managed / hybrid を **config で切替可能** にする (例: tier 4 は self-hosted、tier 6 は Cloudflare といったクロスオーバー構成が可能)。

**Opt-out 原則** (Synergos から継承):

- 既定は **self-hosted** で privacy 側。
- 明示的 opt-in (config または lobby 側トグル) で managed 経路を有効化。
- session establishment 時にどの tier が managed かをクライアントへ通知し、**接続前にユーザが合意できる**ようにする。

**Synergos Issue #23 (TURN) との関係**:

- self-hosted TURN 実装の完遂は **引き続き必要**。privacy 前提のユーザ・自社ホスト希望のインテグレータ向け。
- managed を既定採用するなら #23 の優先度は下げられる。採否は「インディ向け摩擦最小化」と「プライバシ第一主義」のどちらを重視するかの判断。
- M3 マイルストーンは **「TURN 経路が通る」** という抽象条件に変更し、実装は self-hosted か managed のどちらか一方でクリア可とする。

### 5.4 同期戦略 (プロファイル別)

**2 軸の直交**: 「権威の所在」と「誤差補正戦略」は別物として扱う。

|  | **Reconciliation** (forward-only) | **Rollback** (re-simulate) |
|---|---|---|
| **Authoritative server** | **Tessera 既定** (Overwatch / Valorant 系) | 採用しない |
| **P2P mesh (決定論必須)** | 採用しない | **Tessera mesh プロファイル** (GGPO 系) |

双方を同時に提供し、ゲーム種別に応じて選択する。

#### 5.4.1 Authoritative プロファイル — reconciliation + lag compensation

```
Client: tick t で入力 I_t → 自分だけローカル予測 (p_t, p_t+1, ...)
Server: I_t 受理 → 権威 sim → snapshot S_t broadcast
Client: S_t 到着 → 自分の予測 p_t と照合
          差分あり → S_t を基点に I_t+1..I_now を再適用 (forward replay のみ)
```

- 自分以外の peer は **補間表示** (100 ms buffer)。予測しない。
- 発砲・被弾判定は **サーバ側 lag compensation** — 入力到着時にサーバが過去状態を巻き戻してヒット判定 (Valve Source モデル)。クライアントは巻き戻さない。
- 非決定論 OK。物理・RNG・float 自由。
- 適合: FPS / MOBA / 協力アクション / 中〜大規模対戦。

#### 5.4.2 Mesh プロファイル — deterministic rollback (GGPO 系)

```
各 peer: 自入力を全員に broadcast + 他 peer 入力は「直前入力リピート」で仮埋め
全員:    同一の決定論 sim を tick 単位で実行
         実入力が届く → 差分があれば該当 tick まで rollback → 全員分再実行
```

- 入力遅延 1–2 frame、rollback 深さ最大 8–10 frame が典型。
- ≤ 8 人、60–120 Hz、state < 32 KB が実用ライン。
- 適合: 格闘 / 小規模 PvP / 小規模協力 / 軽量 RTS。

#### 5.4.3 Rollback を採る代償 (mesh プロファイルのアーキ制約)

mesh を選んだ時点で以下を制約として受け入れる。設計判断が sim 層全体に伝播する。

1. **決定論**
   - fixed-point 算術、または seed を厳密管理した float (`-ffp-contract=off` 相当)。
   - RNG は tick 単位で seed 化。グローバル乱数禁止。
   - ECS system 実行順は固定。hashmap iteration order を sim に入れない (`FxHashMap` 等、順序保証のあるものに限定)。
   - 外部 API・async・OS 時刻は sim に入れない。
2. **State snapshot 予算**
   - 毎 tick 保存 → `rollback_depth × state_size` のメモリ。
   - ring buffer 管理。想定上限 state 32 KB × depth 10 = 320 KB。超過時は state 圧縮か分割保存。
3. **One-shot event の rollback 耐性**
   - 効果音・パーティクル・チャット送信・ネット外部作用は以下いずれかで吸収:
     - **idempotent 再発火**: `(tick, event_id)` 一致は 2 度目以降 no-op。
     - **確定 tick まで遅延発火**: rollback 範囲外に出てから実行。
   - どちらを既定とするかはゲーム側で選択可能にする (API レベルで両対応)。

### 5.5 権威的決定の管理

両プロファイル共通の「嘘を吐かせない」仕組み。

#### 5.5.1 署名入力

全入力パケット (`TSRI`) に ed25519 署名を付す。サーバ / arbiter は署名検証済みの入力のみ sim に投入。`(peer_id, tick, seq)` 単調増加で replay protection。

→ 「私はそれを押していない」という否認ができない監査トレイルになる。

#### 5.5.2 決定イベントの権威記録

ダメージ確定・アイテム取得・勝敗判定などの **確定イベント** は以下で発行する:

| プロファイル | 発行者 | クライアント側表示 |
|---|---|---|
| Authoritative | サーバ | 確定前は予測表示、確定到達で locked 表示に切替 |
| Mesh | arbiter (PeerId 最小) | 同上。arbiter 署名付きで broadcast |

確定イベントには `(tick, event_id, signer, sig)` を付与。重複・順序逆転はクライアント側で冪等に処理。

#### 5.5.3 Arbiter 選出と host migration (mesh 専用)

- **選出**: 参加 peer の PeerId 最小を arbiter とする決定論選出。参加 / 離脱で再選出。
- **落下時の引き継ぎ**: 全員が直近 snapshot + 確定 tick までの入力ログを保持しているため、次席 PeerId が確定 tick から決定論再生して状態を引き継ぐ。**保留中の未確定入力はログに残るため欠落しない**。
- **split-brain**: QUIC keep-alive 3 s 超 + 過半数合意で旧 arbiter を降格。少数派 peer は rejoin を試みる。

#### 5.5.4 戦略決定フロー (ゲーム別プロファイル選択)

```
       開始
         │
         ▼
  ┌─────────────────────┐
  │ プレイヤー数 > 8 ?  │────── Yes ─────► Authoritative 確定
  └─────────────────────┘
         │ No
         ▼
  ┌─────────────────────┐
  │ 決定論 sim が書ける?│────── No ──────► Authoritative 確定
  │ (物理, RNG, ECS順)  │
  └─────────────────────┘
         │ Yes
         ▼
  ┌─────────────────────┐
  │ 低遅延が最優先?     │────── No ──────► Authoritative 推奨
  │ (格闘・精密 PvP)    │
  └─────────────────────┘
         │ Yes
         ▼
  ┌─────────────────────┐
  │ チート耐性最優先?   │────── Yes ─────► Authoritative 確定
  │ (ランク戦・賞金)    │                  (rollback より安全)
  └─────────────────────┘
         │ No
         ▼
    Mesh (rollback) 確定
```

既定は Authoritative。Mesh は「全条件を満たしたときのみ」オプトインで選ぶ。

## 6. モバイル最適化

### 6.1 QUIC connection migration

- Wi-Fi ↔ LTE 切替時、QUIC CID を保持したまま endpoint を差し替える (RFC 9000 §9)。
- PATH_CHALLENGE/RESPONSE で新経路検証。
- セッションは切れない (信頼性高)。

### 6.2 tick プロファイル (foreground-only)

§2.4 により background / suspend 状態は持たない。tick 階層は以下に簡略化:

| 状態 | tick | 備考 |
|---|---|---|
| active | 30–60 Hz | 画面表示中、常態 |
| low-power (OS 要求) | active / 2 | iOS Low Power Mode / Android Battery Saver 応答 |
| backgrounded | — | セッション終了。画面復帰時は新規接続 |

keep-alive は QUIC の PING frame に任せ、アプリ層では送らない。画面復帰時の再参加は §6.3 参照。

### 6.3 再接続戦略 (route migration に閉じる)

foreground-only 前提により、「セッション中の切断」は実質 **route migration (Wi-Fi ⇄ LTE) のみ**が対象。

- QUIC connection migration (§6.1) が透過吸収。上位レイヤでは断絶を観測しない。
- migration 失敗時は tier の再探索 (§5.3.3) → 新経路で再接続 → delta から継続 (snapshot 再送不要)。
- exponential backoff: 0.25 / 0.5 / 1 / 2 s (4 s 超は session 失敗として諦め、画面再参加を促す)。

**画面復帰 (background → foreground) 時**: §2.4 により旧セッションは終了済。新規接続 + full snapshot で **2 s 以内の再参加** を狙う (§3 目標 2)。

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
| **M3** | TURN 経路統合 (self-hosted または Cloudflare Realtime、どちらか 1 つでクリア) | Synergos #23 または Cloudflare 契約 |
| **M4** | P2P mesh モード (≤ 8 人) — 決定論 sim + rollback + arbiter migration | M2 |
| **M5** | モバイル FFI + 実機テスト | M3 |
| **M6** | Bevy demo 公開 | M4 / M5 |

## 12. 未決事項

- [ ] シリアライザの hot path 実装方式 (手書き vs bitcode vs rkyv)。ベンチで決める。
- [ ] 複数 region にまたがる relay の選択アルゴリズム (static config か動的 RTT か)。
- [ ] FFI の ABI 形式 — C ABI 直か、UniFFI か。モバイル側の開発体験で決める。
- [ ] mesh プロファイル決定論の実装方式 — fixed-point vs seed 管理 float。ベンチと移植性で決める。
- [ ] rollback 深さ / state 予算の最終上限 — 実ゲームでのベンチ後に確定。
- [ ] M3 で self-hosted TURN を既定とするか Cloudflare managed を既定とするか — インディ摩擦 vs プライバシの重み付け判断。

### 解決済 (版 0.2)

- ~~P2P mesh の lag compensation モデル~~ → §5.4.2 で **deterministic rollback** に確定。
- ~~mesh モードでの host migration 中に入力を欠落なく引き継ぐ方法~~ → §5.5.3 で **arbiter 選出 + 確定 tick からの決定論再生 + 保留入力ログ** に確定。

## 13. 参考

- RFC 9000 (QUIC) §9 Connection Migration
- RFC 9221 (QUIC Datagram Extension)
- Overwatch GDC 2017 "Networking Scripted Weapons and Abilities"
- Glenn Fiedler, "Networked Physics" シリーズ
- Valve Source Multiplayer Networking (lag compensation の原型)
- Synergos 内部設計 (`synergos-net/src/lib.rs`, `synergos-core/src/daemon.rs`)
