# AUTOFIX — Tessera (2026-05-13)

- 対象: LUDIARS/Tessera (E:/Document/Ars/Tessera)
- 実装: **未着手** (コード修正対象なし)
- autofix_count: **0**
- 本リストは「**列挙のみ**」 (実適用なし)

## 方針

- Tessera は実装コードがゼロで `.rs` ファイルが存在しない。 修正対象は README.md と docs/DESIGN.md の Markdown のみ。
- AIFormat 5 の AUTOFIX は「**安全範囲の自動修正候補**」 を列挙する。 Tessera では Markdown 軽微修正のみが候補となるが、 **設計書本体は版管理 (v0.7) されており勝手な書換えは設計意図を損なう** ため、 今回は全件 列挙のみ・適用なし とする。
- 実適用は v0.8 設計改訂時に著者 (kazmit299) が判断する。

## 列挙された候補 (適用なし)

### docs/DESIGN.md

| # | 種別 | 箇所 | 内容 | 適用? |
|---|---|---|---|---|
| 1 | crate 名 | §8 (docs/DESIGN.md:709) | `tessera-codex` → `tessera-codex-client` (Codex 別リポ移行 970590f との整合) | **No** (設計判断の領域、 著者確定後) |
| 2 | 節構造 | §5.7 (docs/DESIGN.md:410-650) | 240 行を 30 行要約 + LUDIARS/Codex リンク + API 境界節へ圧縮 | **No** (設計改訂、 v0.8) |
| 3 | マイルストーン | §11 M7 (docs/DESIGN.md:746) | 「Codex 統合 (LUDIARS/Codex への依存)」 へ書換 | **No** (設計判断) |
| 4 | 未決事項 | §12 (docs/DESIGN.md:751-763) | M0-M8 との対応表を追加 | **No** (設計判断) |
| 5 | テスト KPI | §10 (docs/DESIGN.md:728-734) | 各 profile に数値 KPI を埋める | **No** (設計判断、 実機ベンチ要) |
| 6 | フローチャート | §5.5.4 (docs/DESIGN.md:308-336) | ASCII 罫線 → mermaid 化 | **No** (見た目、 著者好み) |

### README.md

| # | 種別 | 箇所 | 内容 | 適用? |
|---|---|---|---|---|
| 7 | ステータス節 | README.md:22-24 | pause 理由 (Codex 切り出し) + 関連リポリンク + DESIGN.md セクション索引を追記 | **No** (README は著者声明、 自動修正領域でない) |
| 8 | バッジ | README.md (先頭) | LUDIARS ダッシュボード badge / CI status badge を追加 | **No** (実装着手まで延期) |

### .gitignore

| # | 種別 | 箇所 | 内容 | 適用? |
|---|---|---|---|---|
| 9 | パターン追加 | .gitignore (末尾) | `.DS_Store`、 `Thumbs.db`、 `*.swp` 等 OS noise を追加 | **No** (実装着手時に Synergos の .gitignore を流用予定) |

### 新規ファイル

| # | 種別 | 箇所 | 内容 | 適用? |
|---|---|---|---|---|
| 10 | 新規 | CLAUDE.md | AI 用ガイドライン (DESIGN.md 単一情報源 / §5.7 は Codex 参照 / pause 中の旨) | **No** (実装再開時に追加) |
| 11 | 新規 | .github/workflows/ci.yml | Synergos CI を流用 (cargo test + clippy -D warnings + deny advisories) | **No** (M0 着手時) |
| 12 | 新規 | Cargo.toml | 8 crate workspace スケルトン (`tessera-proto`/`-net`/`-server`/`-client`/`-mesh`/`-codex-client`/`-ffi`/`-sim`/`-demo`) | **No** (M0 着手時) |

## 適用基準

このレビュー run では「**設計判断を含むためすべて列挙のみ**」 とし autofix_count=0。 著者が v0.8 改訂時に上記 12 件のうち以下を取り込むことを推奨:

- v0.8 改訂時に取り込み: #1, #2, #3, #4 (Codex 関連 + マイルストーン整理)
- README 更新時に著者判断: #7
- M0 着手時に自動化: #9, #10, #11, #12
- v0.8 以降の議論: #5, #6, #8

## 注記

- 通常 autofix の対象になる「lint / typo / formatting」 は、 Tessera では設計書 Markdown と README にしか存在しないため、 自動適用しない方針。
- 実装着手後 (M0 以降) は cargo fmt / clippy / deny の自動適用が AUTOFIX 対象になる。
