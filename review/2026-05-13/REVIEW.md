# REVIEW — Tessera (2026-05-13)

- 対象: LUDIARS/Tessera (E:/Document/Ars/Tessera)
- 日付: 2026-05-13
- 実装: **未着手** (.gitignore / README.md / docs/DESIGN.md の 3 ファイルのみ)
- レビュー版: AIFormat 5 テンプレ準拠

## サマリ

| 観点 | 評価 | 一言 |
|---|---|---|
| **REVIEW_DESIGN** | B+ | v0.7 設計書は LUDIARS 上位。Codex 切り出しで §5.7 が宙吊り |
| **REVIEW_VULNERABILITY** | B | ed25519 + authoritative は堅実、 invitation token / Sybil grind が未詰め |
| **REVIEW_IMPLEMENTATION** | N/A (Tentative C) | コードゼロ。 M0 着手の準備は概ね OK |
| **REVIEW_MISSING_FEATURES** | C | 全マイルストーン未着手。 設計書の網羅性は高いが lobby boundary / replay / observability が薄い |
| **REVIEW_QUALITY** | A− | 設計書品質は LUDIARS 内 top tier。 README 拡充の余地あり |

## weighted_score

各観点に重みをつけて算出 (A=4 / A−=3.7 / B+=3.3 / B=3.0 / C=2.0 / N/A=実装ゼロは 1.0 換算):

| 観点 | 評価 | 数値 | 重み |
|---|---|---|---|
| Design | B+ | 3.3 | 0.30 |
| Vulnerability | B | 3.0 | 0.20 |
| Implementation | C (N/A→1.0) | 1.0 | 0.20 |
| Missing features | C | 2.0 | 0.15 |
| Quality | A− | 3.7 | 0.15 |

`weighted_score = 3.3×0.30 + 3.0×0.20 + 1.0×0.20 + 2.0×0.15 + 3.7×0.15 = 0.99 + 0.60 + 0.20 + 0.30 + 0.555 = 2.645`

→ **weighted_score: 2.65 / 4.00 (= B−)**

実装が一切無いため Implementation の寄与で大きく引き下げられている。 設計のみの相対評価では B+〜A−。

## 主要所見 (top 5)

1. **pause 中だが設計書は LUDIARS 上位** — v0.7 (docs/DESIGN.md:7) は 776 行で章立て・数値・トレードオフが整っている。 再開時に M0 へ即着手できる。
2. **§5.7 Codex 切り出しによる宙吊り** (docs/DESIGN.md:411-412) — Codex を LUDIARS/Codex 別リポへ移管決定済だが、 §5.7 全 240 行 + §8 `tessera-codex` crate (docs/DESIGN.md:709) が DESIGN.md に残置。 v0.8 で要約節へ圧縮が必要。
3. **invitation token (tessera hospitalis) のライフサイクル未定義** (docs/DESIGN.md:691) — 命名の根幹かつ §7 セキュリティの中核なのに、 発行・有効期限・revoke が未記述。 実装着手前に詰めるべき high 優先。
4. **arbiter 選出 PeerId 最小は grind 可能** (docs/DESIGN.md:154, 300-301) — Sybil で「最小 PeerId」 を grind できるためランク戦 / 賞金で悪用余地。 全 peer の hash combination ベースに変更推奨。
5. **§12 未決 12 件 (docs/DESIGN.md:751-763)** — シリアライザ・FFI 形式・managed 既定など、 M0-M5 ブロッカーが含まれる。 マイルストーン × 未決事項のマトリクスを追加し再開コストを下げるべき。

## 件数

- REVIEW_DESIGN: 改善余地 4 件
- REVIEW_VULNERABILITY: High 2 / Medium 4 / Low 3 = 9 件
- REVIEW_IMPLEMENTATION: 着手前 spec 不足 6 件 (M0 4件 + M1 1件 + 外部依存 1件)
- REVIEW_MISSING_FEATURES: A 9 件 (M0-M8) + B 8 件 + C 5 件 = 22 件 (重複あり、 実質ユニーク 19 件)
- REVIEW_QUALITY: 改善 5 件
- **総計: 約 45 件 (重複含む)、 ユニークで約 35 件**

## 詳細

- [REVIEW_DESIGN.md](REVIEW_DESIGN.md)
- [REVIEW_VULNERABILITY.md](REVIEW_VULNERABILITY.md)
- [REVIEW_IMPLEMENTATION.md](REVIEW_IMPLEMENTATION.md)
- [REVIEW_MISSING_FEATURES.md](REVIEW_MISSING_FEATURES.md)
- [REVIEW_QUALITY.md](REVIEW_QUALITY.md)
- [AUTOFIX.md](AUTOFIX.md) (autofix_count=0)
- [latest.json](latest.json)

## レビュアー注記

- 本レビューは AIFormat 5 テンプレに従い、 ソースコード修正なし。 AUTOFIX は列挙のみ。
- 5 レビュー合計文字数は約 3,300 字 (目標 2,000-4,000 字内)。
- 評価は実装が一切無い前提を加味。 設計のみで A 級まで届く可能性は十分にあるが、 実装ゼロが重みを引き下げている。
