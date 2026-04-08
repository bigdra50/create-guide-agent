# create-guide-agent

特定ドメインの公式ドキュメントを自律的に参照する「Guideエージェント」を対話的に作成する Claude Code スキル。

## 概要

ユーザーからドメイン情報を収集し、ドキュメントサイトを自動調査してエージェント定義ファイル（`.md`）を生成する。
生成されたエージェントは、ユーザーの質問に対して公式ドキュメントをfetchしてから回答する読み取り専用のサブエージェントとして動作する。

## トリガー例

- 「guideエージェント作って」
- 「〜のガイドエージェント」
- 「公式ドキュメントを見に行くエージェント」
- "create guide agent"

## ワークフロー

```
Phase 1: 情報収集          Phase 2: ドキュメント調査       Phase 3: エージェント生成
+-----------------+       +------------------------+      +--------------------+
| ドメイン名       |       | docs map URL特定        |      | パターン選択        |
| ドキュメントURL   | ----> | sitemap.xml / index     | ---> | テンプレート適用     |
| 専門領域         |       | r.jina.ai フォールバック |      | .md ファイル出力    |
| 配置先           |       | ドメイン定数収集        |      |                    |
+-----------------+       +------------------------+      +--------------------+
```

## 生成パターン

| パターン | 条件 | 特徴 |
|----------|------|------|
| Docs Map型 | docs mapページが存在 | 軽量、ドキュメント変更に自動追従 |
| URL Table型 | docs mapなし、URL安定 | カテゴリ別テーブルで列挙 |
| Staged Search型 | docs mapなし、バージョン差異大 | WebSearch補完あり |
| Local Reference型 | Web参照不要 | ローカルファイルのみ |

## インストール

```bash
npx skills add bigdra50/create-guide-agent
```

## ディレクトリ構成

```
create-guide-agent/
├── skills/
│   └── create-guide-agent/
│       ├── SKILL.md                            # スキル定義
│       └── references/
│           └── guide-agent-patterns.md         # パターンリファレンス
└── README.md
```

## 生成されるエージェントの特徴

- tools: `Glob, Grep, Read, WebFetch, WebSearch`（読み取り専用、Edit/Write/Bashなし）
- model: `sonnet`
- JSレンダリング必須サイトには `r.jina.ai` プロキシを自動適用
- ドメイン固有の定数がある場合はQuick Referenceテーブルとして埋め込み

## ライセンス

MIT
