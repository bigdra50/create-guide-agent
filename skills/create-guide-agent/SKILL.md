---
name: create-guide-agent
description: |
  特定ドメインの公式ドキュメントを自律的に参照する「Guideエージェント」を対話的に作成するスキル。
  ユーザーからドメイン情報を収集し、ドキュメントサイトを自動調査してエージェント定義ファイルを生成する。
  "guideエージェント作って", "ドキュメント参照エージェント", "create guide agent",
  "~のガイドエージェント", "公式ドキュメントを見に行くエージェント" で使用する。
---

# Create Guide Agent

特定ドメインの公式ドキュメントを参照してから回答するGuideエージェントを対話的に作成する。
ユーザーから情報を集め、不足分はClaude Codeが自動調査し、エージェント定義ファイルを生成する。

パターンリファレンス: [references/guide-agent-patterns.md](references/guide-agent-patterns.md)

## Workflow

### Phase 1: 情報収集

AskUserQuestionで以下を収集する。一度に全部聞かず、2〜3問ずつ段階的に聞く。

Round 1 — 基本情報:
- ドメイン名（例: "Unity", "Figma API", "Kubernetes"）
- 公式ドキュメントのURL（わかる範囲で）
- Web参照が必要か、ローカルファイルのみか

Round 2 — 専門領域:
- このエージェントがカバーすべき専門領域（2〜6個）
- よくある質問パターン（例: "APIの使い方", "設定方法", "トラブルシュート"）

Round 3 — 配置先:
- エージェント定義ファイルの配置先（`~/.claude/agents/` or `.claude/agents/`）

### Phase 2: ドキュメント調査

ユーザーから得た情報をもとにClaude Codeが自動調査する。

#### 2a. Docs Map URLの特定（最重要ステップ）

生成するエージェントの軽量さと保守性を決める。以下を順に試す:

1. `{base-url}/sitemap.xml` を WebFetch
2. `{base-url}/docs/` や `{base-url}/documentation/` を WebFetch
3. サイトのAPIリファレンストップページを WebFetch
4. WebSearch `site:{docs-domain} documentation index` で探す

WebFetchが空を返す場合（JSレンダリング必須サイト）、Jina Reader (`r.jina.ai`) を使う。
URLの前に `https://r.jina.ai/` を付けてWebFetchすると、JSレンダリング後のページをMarkdownで返す:

```
WebFetch → https://r.jina.ai/{original-url}
```

生成するエージェントのDocumentation sourcesにも同じプレフィックスを使用する。

`r.jina.ai` でも取得できない場合、最終手段として `mcp__claude-in-chrome__` を使用。

結果の判定:
- docs map URLが見つかった → Docs Map型で生成（推奨）
- 見つからない → URL Table型 or Staged Search型にフォールバック
- Web参照不要 → Local Reference型

docs mapが見つかった場合、そのURLから到達できるページのカテゴリ構造を把握し、
ユーザーが指定した専門領域ごとにどのトピックが該当するかを整理する。

ドキュメントの内容（API定数、スペック、仕様等）はエージェントプロンプトに埋め込まない。
すべて実行時にWebFetchで動的取得する。

### Phase 3: エージェント生成

references/guide-agent-patterns.md を読み、収集した情報からエージェント定義ファイルを生成する。

#### 3a. Docs Map型（推奨 — docs map URLが見つかった場合）

guide-agent-patterns.md の「Docs Map型 > 生成テンプレート」に従って生成する。
JSレンダリング必須サイトの場合、Documentation sourcesのURLを `r.jina.ai` プレフィックス付きにする。

#### 3b. フォールバック型（docs map URLが見つからなかった場合）

guide-agent-patterns.md の URL Table型 / Staged Search型 / Local Reference型 から
ドメインに適したパターンを選択して生成する。

### Phase 4: レビュー

生成したエージェント定義ファイルを2つの観点で並列レビューする。

Agentツールで以下を同時に起動:

1. 構造レビュー — テンプレート準拠の検証:
   - テンプレートにないセクション（テーブル、定数リスト等）が追加されていないか
   - ドキュメント内容（API定数、スペック、仕様等）が埋め込まれていないか
   - frontmatter（tools, model）が正しいか
   - descriptionに具体的なキーワードが含まれているか

2. 到達性レビュー — ドキュメントURLの検証:
   - Documentation sourcesの各URLにWebFetchで到達できるか
   - r.jina.ai が必要なサイトで適切にプレフィックスされているか

レビュー結果を統合し、問題があればユーザーに提示して修正方針を確認する。
問題がなければ生成結果の概要を報告:
- ファイルパス
- 選択したパターン（Docs Map型 / URL Table型 / etc.）
- docs map URL（Docs Map型の場合）
- カバーするドメイン一覧
