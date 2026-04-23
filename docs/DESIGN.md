# Tessera 設計書

> 版: 0.5 — 2026-04-23
> 著者: kazmit299
> ステータス: ドラフト (実装未着手)
>
> **変更履歴**
> - 0.5 (2026-04-23) — §5.6 人数規模別 tier (T1 Duel / T2 Squad / T3 Arena / T4 Battle) を追加。tier 境界の理由、ワークアラウンド、ゲーム設計制約、session 生成時の tier 決定ルールを整備
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
| 2–32 人規模のリアルタイムゲーム通信 (4 tier、§5.6) | MMO 規模の interest management |
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

### 5.6 人数規模別プロファイル (scaling tiers)

ネットワーク設計は人数で決定的に変わる。トポロジ・同期戦略・帯域予算・ゲーム設計制約が **階段状に切り替わる**ため、Tessera はスコープ内 (2–32) を 4 tier に分割し、tier ごとに独立した運用パラメタを持つ。

#### 5.6.1 Tier 一覧

| tier | 人数 | トポロジ | 同期戦略 | client DL 帯域 | サーバ CPU | 代表ジャンル |
|---|---|---|---|---|---|---|
| **T1 Duel** | 2 | mesh P2P | deterministic rollback | ~5 KB/s | 不要 | 格闘 / 1v1 タクティクス |
| **T2 Squad** | 3–8 | mesh or auth | rollback or reconciliation | ~15 KB/s | 軽 | co-op / 小規模 PvP / 軽量 RTS |
| **T3 Arena** | 9–16 | authoritative | reconciliation + lag comp | ~30 KB/s | 中 | 4v4–8v8 チーム戦 / アリーナ |
| **T4 Battle** | 17–32 | authoritative + AoI | reconciliation + lag comp + interest management | ~50 KB/s | 重 | 中規模 PvP / 軽量 BR |
| (範囲外) | 33+ | sharded MMO | — | — | — | MMO / 大規模 BR |

#### 5.6.2 Tier 境界が変わる理由

- **T1 → T2 (2→3)**: 第三の peer が存在するため peer-relay (§5.3 tier 3) が可能になり、mesh の救済経路ができる。同時に、死亡/脱落ゲーム設計が発生する。
- **T2 → T3 (8→9)**: rollback CPU コスト `O(state × player × depth)` が破綻する。さらに全員分入力の全員配信の帯域が厳しい。authoritative 一択へ。
- **T3 → T4 (16→17)**: full snapshot の帯域が破綻する (16p × 60Hz × 2KB = 2MB/s/client)。**AoI (interest management) 必須**。
- **T4 → 範囲外 (32→33)**: 単一サーバの entity 管理と per-client 帯域の両方が限界。**sharding / zone 分割アーキ** が必要になり、Tessera の守備範囲を超える。

#### 5.6.3 Tier 別ワークアラウンド (主経路が壊れた時)

| tier | 故障モード | ワークアラウンド |
|---|---|---|
| T1 | 両者 symmetric NAT + peer-relay 対象なし | TURN UDP → TURN TCP → WS relay 順に落下。遅延 +10–40 ms を許容、ゲーム側は入力遅延を動的に +1 frame 拡張 |
| T1 | 決定論が崩れた (非同期 event / float divergence) | **mesh を諦め、PeerId 小を arbiter とする authoritative モードに降格**。rollback 停止、reconciliation に切替 (session 継続は可) |
| T2 | ホスト品質劣化 (LTE 詰まる・移動中) | authoritative: host migration。mesh: rollback 深さを動的縮小 (10→5 frame) |
| T2 | 全員モバイル + 全員 symmetric NAT | Cloudflare Realtime TURN 強制。帯域は throttle で吸収 |
| T3 | サーバ CPU が tick 予算超過 | entity 更新頻度の優先度差別化 (自機近傍 30Hz / 中距離 15Hz / 遠景 5Hz) |
| T3 | 1 client の帯域枯渇 (モバイル データ制限) | AoI 半径を動的縮小 + delta field mask を絞る (見た目の詳細を下げる) |
| T4 | 特定エリアへのプレイヤー集中 (hot spot) | spatial cell ごとの snapshot rate 動的調整、過負荷時は非戦闘 entity を drop |
| T4 | サーバ移行中の切断 | 全 client に pause broadcast → 次席サーバで resume。2 s 以内の unfreeze を目標 |
| 全 tier | Cloudflare 側障害 | self-hosted tier に fail-over (§5.3.4 の transport trait で切替) |

#### 5.6.4 Tier 別ゲーム設計制約

ネットワークがゲーム設計を縛る箇所を明示。実装開始時の手戻りを防ぐ。

**T1 (2p rollback)**:
- **決定論必須** — physics は fixed-point か厳密 float。RNG は tick seed 固定。グローバル乱数禁止。
- **入力遅延 2–3 frame を UX 前提化** — 先行入力バッファ等でユーザから見えにくくする。
- **非可逆 I/O を sim に入れない** — BGM 再生・Haptic・振動は rollback 外で idempotent 発火。
- **マッチ中の非決定要素禁止** — ランダムアイテム降臨等は tick seed から導出。

**T2 (3-8 rollback/auth)**:
- **state サイズ予算 < 32 KB** — rollback 採用時。プレイヤー + projectile で自然に到達するため上限監視。
- **同時インタラクション数の上限** — AoE / 範囲 effect のイベント数を厳格に算定。
- **脱落設計** — 誰かが落ちた時の進行 (AI 代替 / 即終了 / 待ち) をゲームが決める。mesh 採用なら host migration 込みで 3 s 以内復帰を保証。
- **authoritative 降格耐性** — 決定論崩壊で降格しても破綻しないよう、ゲームロジックは auth モードでも動く抽象で書く。

**T3 (9-16 authoritative)**:
- **予測の限界を許容** — 自機以外は補間のみ = 遠距離の他プレイヤー位置は常に 100 ms 過去。近接判定は server authoritative、hit-confirm UI は慎重に。
- **match 時間の上限設計** — 長時間セッションは state leak / sync drift のリスク。30–60 分でリセット。
- **AoI 境界の演出** — 見えない → 見える遷移を自然に (fade-in, 通知音等)。

**T4 (17-32 auth + AoI)**:
- **マップ配置で密度管理** — 全員が 1 点に集まれない設計 (safe zone、spawn 分散、choke point 制限)。
- **同時 visible entity 上限 (per client)** — 典型 30–50。視界内 culling も設計に入れる。
- **respawn/観戦の別ストリーム化** — 死亡後のカメラは低 tick 観戦ストリームで帯域節約。
- **イベント確定の aggregate** — 同時多発 kill feed 等は 10/s 程度にまとめる。

#### 5.6.5 Session 生成時の tier 決定

- lobby が人数を決定した時点で tier が **確定**。
- **session 中に tier を切り替えない** — アーキテクチャが階段状に異なるため安全遷移不能。
- 人数が途中で減って境界を跨いでも、**残り時間は元 tier で続行**。次 match で再決定。
- 人数が増える方向 (脱落代替の new join) は tier 内で処理、tier 超過は拒否。
- 降格 (例: T1 rollback → T1 authoritative) は tier を跨がないため session 中に可能 (§5.6.3 ワークアラウンド参照)。

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
- [ ] §5.6 tier 境界 (8 / 16 / 32) の具体値 — 理論算定ベース、M4–M6 で実機ベンチ確定。特に T2 rollback 上限 8 は CPU/メモリ次第で 6 に下がる可能性。

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
