# REVIEW_DESIGN — Tessera

- 対象: LUDIARS/Tessera (E:/Document/Ars/Tessera)
- 日付: 2026-05-13
- 版: docs/DESIGN.md v0.7 (2026-04-23) / 実装未着手
- 評価: **B+**

## 総評

Tessera は「Synergos 下層 (QUIC + TURN + ed25519 + chain) をモバイルゲーム通信プロファイルとして切り出す」設計書のみのプロジェクト。v0.7 で 4 章まで一通り書き終え、Codex を §5.7 から独立リポへ切り出した時点で意図的に pause している。設計書は 776 行・密度が高く、6 段フォールバック (docs/DESIGN.md:158-168)、4 tier (T1 Duel / T2 Squad / T3 Arena / T4 Battle, docs/DESIGN.md:346-352)、2 軸の同期戦略 (auth × mesh / reconciliation × rollback, docs/DESIGN.md:226-232)、CQRS 分離した Codex 読み出しパス (docs/DESIGN.md:460-539) など、書く順序・粒度ともに筋がよい。

特筆できる強み:

- **foreground-only 前提を §2.4 で明文化 (docs/DESIGN.md:65-77)** したことで、tick 階層 (§6.2) と再接続戦略 (§6.3) が大幅に簡略化され、設計負債を未然に削っている。Synergos との根本的なスコープ差をうまく言語化できている。
- **「権威の所在」と「誤差補正戦略」を直交 2 軸として §5.4 で表化 (docs/DESIGN.md:226-232)**。よくある「auth は reconciliation、mesh は rollback」という暗黙の対応をマトリクスで明示し、ハイブリッド/降格の議論 (T1 故障モード, docs/DESIGN.md:367) に接続している。
- **Tier 境界が変わる「理由」を §5.6.2 で明示 (docs/DESIGN.md:354-359)**。 2→3 (peer-relay)、8→9 (rollback CPU `O(state × player × depth)`)、16→17 (AoI)、32→33 (sharding) と CPU/帯域/トポロジが「階段状」に変わる根拠を式で説明しており、tier 数の選択に説得力がある。

## 評価できる点

- 経路 6 段フォールバック (docs/DESIGN.md:161-168) を「並列試行・採用は `RTT + jitter × 2` 最小、復活時は 30s 安定で migration」(docs/DESIGN.md:194-198) と運用ルールまで規定。チャタリング防止まで思考が及んでいる。
- 日本キャリア IPv6 inbound 遮断の現実 (docs/DESIGN.md:175-188) を表で書き、「直接 P2P 主経路にしない、TURN への outbound 品質チャネル扱い」と結論を明示。理想論と現実の橋渡しが明快。
- Cloudflare Realtime を §5.3.4 で managed 置換オプションとして組込み (docs/DESIGN.md:200-222)、self-hosted との切替を `transport trait + config` で表現。Synergos #23 (TURN) との優先度関係も整理されている。
- Codex (§5.7) は「§5.4 同期戦略との関係」を §5.7.1 で先に切り分け (docs/DESIGN.md:417-422)、tick loop と独立に動くことを冒頭で宣言。読者がレイヤを取り違えないようにしている。

## 改善余地

- **§5.7 Codex は別リポ切り出し決定済 (docs/DESIGN.md:411-412)** にも関わらず本文 200 行が残置されたまま。「移行完了後に要約節へ置換される」と明記されているのは誠実だが、現状の DESIGN.md を新規読者が読む時、§5.7 のどこからが「移行対象」か曖昧。**設計版 0.8 で §5.7 を 30 行程度の要約 + `LUDIARS/Codex` リンク + `tessera-codex-client` 境界仕様だけに圧縮する課題を §12 未決事項に追加すべき** (docs/DESIGN.md:749-763)。
- **Crate 構成 §8 (docs/DESIGN.md:702-712)** に `tessera-codex` がまだ含まれている。Codex 切り出し決定と整合させ、`tessera-codex-client` (API 境界のみ) に置換する必要あり。
- **未決事項 §12 (docs/DESIGN.md:751-763)** は 12 項目あるが、 解決順序・依存関係 (例えばシリアライザ決定は M0 までに、tier 境界は M6 までに) が書かれておらず、再開時の優先付けで迷う。「マイルストーン × 未決事項」のマトリクスがあると pause 後の再開コストが下がる。
- **§5.5.3 split-brain (docs/DESIGN.md:303-304)** で「QUIC keep-alive 3s 超 + 過半数合意で旧 arbiter を降格」とあるが、 過半数が取れない偶数人数 (T1=2, T2 偶数) のケースが未定義。 2 人 mesh で arbiter が落ちた時の挙動を明示する必要あり (§5.5.3 か §5.6.3 T1 故障モード)。
- **§7 セキュリティ (docs/DESIGN.md:687-694)** は 6 項目だが、 「Sybil (大量の偽 PeerId 作成)」「招待トークンのリーク (tessera hospitalis の半分が漏れた場合)」「relay 帯域上限の DoS 計算」が抽象的。 特に invitation token のライフサイクル (発行・失効・revoke) が未記述。

## 設計の優先課題 (再開時)

1. §5.7 を「Codex client API + 境界規約」だけに圧縮し、Tessera DESIGN.md の自己完結性を回復する。
2. Crate 構成 §8 を `tessera-codex-client` 化、`Cargo.toml` workspace 雛形 (M0 スコープ) を spec として追加。
3. §12 未決事項に「M との対応表」を追加し、再開時の作業順を明示。
4. §7 にinvitation token のライフサイクル節を追加 (T1–T4 全 tier で必要)。

## 結論

実装着手前の設計書としては LUDIARS 内でも筆頭格の完成度。 v0.7 が「Codex 切り出し」という大きな構造変更で意図的に止まっていることが透けて見えるので、再開前に §5.7 圧縮 + §8 整理 + §12 マイルストーン紐付けの 3 点を片付けると、 M0 着手時の摩擦が大きく下がる。
