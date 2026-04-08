# Guide Agent Patterns Reference

既存のGuideエージェント実装から抽出した共通パターン集。
エージェント生成時にこのファイルを参照し、ドメインに適したパターンを選択する。

## Frontmatter Structure

```yaml
---
name: {domain}-guide
description: |
  {ドメイン}の公式ドキュメント検索、{具体的な機能1}、{具体的な機能2}に関するガイドエージェント。
  {具体的なキーワード1}、{具体的なキーワード2}、{具体的なキーワード3}に関する質問や実装支援で使用する。
tools: {tool_list}
model: sonnet
---
```

### tools の選択基準

| ドキュメント種別 | tools |
|---|---|
| Webドキュメント参照型 | `Glob, Grep, Read, WebFetch, WebSearch` |
| ローカルファイル参照型 | `Glob, Grep, Read` |

Edit, Write, Bash は渡さない。読み取り専用にして意図しない変更を防ぐ。

### description の書き方

descriptionはメインのClaudeがエージェント選択する判断材料。具体的なキーワードを列挙する。

悪い例: 「Unity開発のガイド」
良い例: 「Unity Engine/Editor API、UPM/NuGetパッケージ、YAML設定ファイル編集に関する質問や実装支援で使用する」

英語のトリガー文を末尾に追加すると呼び出し精度が上がる:
```
Use proactively when user asks about {domain keyword}, {domain keyword}, or {integration keyword}.
```

---

## Pattern選択フロー

```
ドキュメントサイトに
docs map/index ページがある?
  |
  +-- Yes → Docs Map型（推奨）
  |
  +-- No
       |
       Web参照が必要?
         |
         +-- Yes → URL Table型 or Staged Search型
         |
         +-- No → Local Reference型
```

---

## Docs Map型（推奨）

claude-guide準拠。docs map URLを動的fetchし、そこから個別URLを特定する。
エージェントプロンプトが軽量になり、ドキュメント構造の変更にも自動追従する。

docs mapとは: ドキュメントサイトのインデックスページ、サイトマップ、TOCページなど、
配下のページURLが一覧できるページのこと。

### 生成テンプレート

```markdown
You are the {domain} guide agent. Your primary responsibility is helping users {goal}.

**Your expertise spans {N} domains:**

1. **{Domain 1}**: {説明}
2. **{Domain 2}**: {説明}
3. **{Domain 3}**: {説明}

**Documentation sources:**

- **{Source 1}** ({docs_map_url_1}): Fetch this for questions about {domain 1}, including:
  - {topic_a}
  - {topic_b}
  - {topic_c}

- **{Source 2}** ({docs_map_url_2}): Fetch this for questions about {domain 2}, including:
  - {topic_d}
  - {topic_e}

**Approach:**
1. Determine which domain the user's question falls into
2. Use WebFetch to fetch the appropriate docs map
3. Identify the most relevant documentation URLs from the map
4. Fetch the specific documentation pages
5. Provide clear, actionable guidance based on official documentation
6. Use WebSearch if docs don't cover the topic
7. Reference local project files when relevant using Glob, Grep, Read

**Guidelines:**
- Always prioritize official documentation over assumptions
- Keep responses concise and actionable
- Include specific examples or code snippets when helpful
- Reference exact documentation URLs in your responses
{domain-specific guidelines}

Complete the user's request by providing accurate, documentation-based guidance.
```

### docs map URLの探し方

以下を順に試す:

1. `{base-url}/sitemap.xml` — XMLサイトマップ
2. `{base-url}/docs/` or `{base-url}/documentation/` — ドキュメントインデックス
3. `{base-url}/api/` or `{base-url}/reference/` — APIリファレンストップ
4. サイトのナビゲーションから到達できるTOCページ
5. GitHubリポジトリのREADME（ドキュメントリンク集がある場合）

### JSレンダリング必須サイトの場合

docs mapページ自体がJS必須の場合、Jina Reader (`r.jina.ai`) を使う。
URLの前に `https://r.jina.ai/` を付けてWebFetchすると、JSレンダリング後のページをMarkdownで返すプロキシサービス:

```markdown
**Documentation sources:**

- **{Source}** (https://r.jina.ai/{docs_map_url}): Fetch this for questions about...
```

エージェントのDocumentation sourcesに `r.jina.ai` プレフィックス付きURLを記載する。
WebSearch (`site:{docs-domain}`) をフォールバックとしてApproachに含める。

### 実装例: claude-guide

```markdown
**Documentation sources:**

- **Claude Code docs** (${CLAUDE_CODE_DOCS_MAP_URL}): Fetch this for questions about the Claude Code CLI tool, including:
  - Installation, setup, and getting started
  - Hooks (pre/post command execution)
  - Custom skills
  - MCP server configuration
  ...

**Approach:**
1. Determine which domain the user's question falls into
2. Use WebFetch to fetch the appropriate docs map
3. Identify the most relevant documentation URLs from the map
4. Fetch the specific documentation pages
5. Provide clear, actionable guidance based on official documentation
6. Use WebSearch if docs don't cover the topic
7. Reference local project files when relevant
```

---

## URL Table型（Docs Mapがない場合のフォールバック）

サイト構造が安定しているがインデックスページがないサイト向け。
URLをカテゴリ別テーブルで列挙する。Docs Map型より重いが確実。

```markdown
## Documentation Structure

ベースURL: `https://{docs-domain}`

| カテゴリ | パス |
|----------|------|
| 概要 | /docs/{path1}/ |
| セットアップ | /docs/{path2}/ |
| API | /docs/{path3}/ |
```

---

## Staged Search型（Docs Mapがない + バージョン差異が大きい場合）

URL Tableに加え、WebSearchによる最新情報補完が必要な場合に使う。

```markdown
## Search Strategy

Stage 1: 公式ドキュメント
├─ WebFetch → {primary-docs-url}
└─ Read → {local-package-docs}

Stage 2: 最新情報補完
├─ WebSearch → "{domain} [feature] {year}"
└─ WebSearch → site:{forum-url} [issue]

Stage 3: ローカルコンテキスト
├─ Read → {project-config-file}
└─ Glob/Grep → プロジェクト内コード検索
```

---

## Local Reference型（Web参照不要）

ローカルファイルの読み取りだけで完結するドメイン向け。

```markdown
## Configuration Sources

- `{config-file-1}` → {説明}
- `{config-file-2}` → {説明}

## Search Patterns

| Topic | Grep pattern | Search path |
|-------|-------------|-------------|
| {トピック1} | `{pattern}` | `{path}` |
```

---

## 埋め込み禁止ルール

ドキュメントの内容（API定数、ハードウェアスペック、仕様テーブル等）はエージェントプロンプトに埋め込まない。
すべて実行時にWebFetchで動的取得する。テンプレートにないセクションを追加しない。

