# Tessera

**リアルタイムゲーム向け低遅延通信基盤** — モバイル主ターゲット。

> *tessera* (ラテン語): ローマ軍の合言葉トークン・ゲーム駒・客人識別標を兼ねた小片。
> 真性認証つきゲーム通信という Tessera の役割にそのまま重なる。

## 位置付け

Synergos のネットワーク下層 (QUIC + relay + STUN/TURN + ed25519) を **ゲーム通信プロファイル** として切り出した姉妹プロジェクト。

| | Synergos | Tessera |
|---|---|---|
| 目的 | ファイル/状態の最終整合性コラボ | 毎フレーム状態同期・入力転送 |
| 転送単位 | chain diff / CID ブロック | tick パケット / 入力イベント |
| 配送保証 | 最終的整合 (Want/Offer ledger) | 期限付き信頼性 (unreliable + sequenced) |
| トポロジ | Gossipsub mesh | star (authoritative) / mesh (small group) |
| 典型 RTT 予算 | 秒単位可 | 20–150 ms |

設計書: [docs/DESIGN.md](docs/DESIGN.md)

## ステータス

設計段階 (PoC 未着手)。Synergos 側 Issue #23 (TURN 完遂) が前提依存。
