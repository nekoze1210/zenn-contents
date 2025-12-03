---
title: "AI開発時代のプロダクト運用におけるラストワンマイルの実現"
emoji: "🧭"
type: "tech"
topics: ["LLM", "Claude", "GauDev" , "アドベントカレンダー"]
published: true
published_at: 2026-12-05 12:00
---

この記事は[#GauDev Advent Calendar 2025](https://adventar.org/calendars/11616)の5日目です。

---

今年は AI 開発が加速する1年でした。Claude Code をはじめとする AI コーディングエージェント や MCP(Model Context Protocol) が公開されてから約半年しか経っていないなんて、信じられませんね...

Gaudiy でも全社的にAI活用を積極的に進めてきました。この記事では、プロダクト運用現場における**AIを活用したラストワンマイル**として試していることをご紹介します。

## AI開発時代におけるプロダクト運用上の課題
AI開発が爆発的に流行っている中、プロダクトの保守・運用の現場では、依然として以下のような課題が残っていました。

- 不具合発生時に原因調査をエンジニアに毎回お願いする必要がある。起票内容も言語化が難しく時間がかかる
- コミュニティ運用時に発生した障害のポストモーテムの記入が手動で、運用負荷が高い（Slackでやりとりしている内容をそのまま要約できることが理想）
- など...

ChatGPT、Gemini、Claude などのLLMアプリケーションにプロダクトのことを聞いても、ハルシネーションを起こすか「分かりません」と返ってくるだけです。なぜなら、プロダクトのコンテキスト（実装・仕様・変更履歴）を持っていないからです。

では、なんとかしてプロダクトのソースコードを Clone して、AIコーディングエージェントが使えるエディタ（VSCode with GitHub Copilot、Cursor など）を用いてエージェントと会話すれば解決するのでしょうか？ 実際に試している非エンジニアの方も多いと思います。

こうしたツールの出力は基本的に**エディタ内に閉じています**。コードを書くことが主業務ではないCSやQA、PdMにとって、エディタ上でAIが動いてくれても、それが日々の業務フローに直結しているかというと、正直なところ微妙な印象です。MCP を活用することで外向きの出力もできますが、設定コストやAPIキーなどの露出などなるべく防ぎたいため、デファクトではないと思われます。

## プロダクト運用におけるラストワンマイルとは
AIコーディングエージェントの強みは、テキストファイル（CLAUDE.md、AGENTS.md など）を起点にプロダクトに関するコンテキストを持った働きができることです。

しかし、前述した通り、すべての職能の人がAIコーディングエージェントを使える環境を整備するのは現実的ではありません。[^1]

**AIコーディングエージェントが持つプロダクトコンテキストを活用できるという強みを、エンジニア以外の職能にも届けること = ラストワンマイル**として定義しました。

[^1]: 厳密には Claude Code Web や GitHub MCP を活用してアクセスできますが、そのためには GitHub アカウントの作成や MCP の設定など障壁が多くあるため、すべての人が実現できるには現実的ではない、という考えを前提に置いています。

## Claude Code GitHub Action を操る Slack Bot の導入
前置きが長くなってしまいましたが、実際にどのようにラストワンマイルを実現するか、様々な手法を検討したところ、

- 業務上の会話における開始地点の**ほとんどがSlack**
- **コンテキストの二重管理**を防ぎたい
- **Claude Code の様々な機能**を活用してみたい
  - Skills, SubAgent, Slash Command など

といった以上の理由を踏まえて、 **[Claude Code GitHub Action ](https://code.claude.com/docs/github-actions)** と **Slack App** の組み合わせに着地しました。

### 利用フロー
たとえば社内でプロダクトの不具合のような挙動を社内のコミュニティマネージャーが発見した場合、以下のようなワークフローが実現されます。
1. コミュニティーマネージャーは Slack で Bot に起きている事象と社内で管理している Notion の不具合 DB に起票してほしい旨をメンションで伝える
2. Bot はメンションをトリガーに Claude Code GitHub Actions のジョブを実行するワークフローを起動する
3. ワークフロー中に実装の調査・原因の特定を行い、結果を Notion の不具合 DB に登録する
4. Claude Code GitHub Actions は調査結果を最終的なアウトプットとして Slack のスレッドに返信する

![](/images/ai-dev-ops-experiments-in-gaudiy-2025/architecture.png)

### 仕組み

#### GitHub Actions の設定
以下のようなワークフローをプロダクトのレポジトリに作成します。

workflow_dispatch イベントの inputs に、後述する Slack Bot からの情報を受け取り、Claude Code に渡すようにしています。
これによって、どのチャンネルで、どのスレッドで、どのようなプロンプトで Claude Code を実行するかを指定できるようになります。

全体として、このようなワークフローになっています。

```yaml:.github/workflows/claude-code-slack-action.yml
name: Claude Code Slack Action

on:
  workflow_dispatch:
    inputs:
      mcp_tools:
        description: "MCP tools to use"
        type: string
        required: false
      slack_channel:
        description: "Slack channel to send the response to"
        type: string
        required: true
      slack_thread_ts:
        description: "Slack thread timestamp to send the response to"
        type: string
        required: true
      prompt:
        type: string
        description: "Prompt for Claude Code Chat"
        required: true

jobs:
  claude-code-slack-action:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v5
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
      - uses: actions/setup-node@v6
        with:
          node-version: 22
      - name: Run Claude Code Base Action
        uses: anthropics/claude-code-action/base-action@v1.0.21
        with:
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
          claude_args: |
            --allowedTools "View,Read,Write,Edit,GlobTool,GrepTool,BatchTool,Task,${{ inputs.mcp_tools }}"
            --mcp-config '{"mcpServers":{"slack":{"command":"npx","args":["-y","@zencoderai/slack-mcp-server"],"env":{"SLACK_BOT_TOKEN":"${{secrets.SLACK_BOT_TOKEN}}","SLACK_TEAM_ID":"XXXXXXXX"}}}}'
          prompt: |
            ## Request Context
            You are an expert AI agent of this repository. You are in Slack channel and thread below.
            channel_id: ${{ inputs.slack_channel }}
            timestamp(ts): ${{ inputs.slack_thread_ts }}

            ## Workflow
            1. Understand the context by fetching the conversation history using mcp__slack__slack_get_thread_replies from the channel and timestamp above.
            2. Understand the user's request.
            3. Answer, implement, or call subagent according to the user's request.
            4. Post the response to the Slack thread using mcp__slack__slack_reply_to_thread.

            Now, you are in a conversation with a user. You need to answer the user's request. You can call subagent according to the user's request.

            ## User's Prompt
            User: ${{ inputs.prompt }}
```

Claude Code の実行におけるポイントは、claude-code-action ではなく、claude-code-base-action を利用することです。
[claude-code-action](https://github.com/anthropics/claude-code-action) は Pull Request や Issue など GitHub 上での操作を起点に実行されるワークフローで、workflow_dispatch に対応していません。
[claude-code-base-action](https://github.com/anthropics/claude-code-base-action) は 直接プロンプトを実行するシンプルな機能を提供しており、MCP など CLI で実行する args も渡せたりなど低レイヤーな機能を提供してくれています。

claude-code-base-action は本来ミラーレポジトリとして管理されていますが、不定期に同期が行われているため、できるだけ最新のものを使えるように、ミラー元の claude-code-action を直接利用することにしました。

```yaml:
uses: anthropics/claude-code-action/base-action@v1.0.21
```

#### Slack Bot の実装

次に Slack Bot 側の実装です。Events API を利用して、メンションをトリガーに GitHub Actions を実行するワークフローを起動します。
今回は Cloudflare Workers x Hono の組み合わせで Events API を実装してみました。

サンプルコードは次のようになっています。

```typescript:index.ts
import { Hono } from "hono";
import { Octokit } from "@octokit/rest";
import { allowedTools } from ./tools";

const app = new Hono<Env>()

app.use('*', verifySlackSignature); // signature の検証は省略

const routes = app.post('/events', async (c) => {
  const body = await c.req.json()
  const event = body.event;

  if (body.type === 'event_callback' && body.event.type === 'app_mention') {
    const client = new WebClient(c.env.SLACK_BOT_TOKEN);
    const threadTs = event.thread_ts || event.ts;
    const octokit = new Octokit({
      auth: env.GITHUB_TOKEN,
    });
    await octokit.actions.createWorkflowDispatch({
      owner: 'owner',
      repo: 'repository',
      workflow_id: 'claude-code-slack-action.yml',
      ref: 'main',
      inputs: {
        prompt: event.text,
        slack_channel: event.channel,
        slack_thread_ts: thread_ts,
        mcp_tools: allowedTools.join(','),
      }
    });
    return c.json({ ok: true });
  }
  return c.json({ ok: true });
});
```

カスタムで追加した MCP で許可したい tool を Slack Bot 側で管理しています。([現状 Claude Code では、MCP の tools にワイルドカードが効かないため手動で列挙する必要があります。](https://code.claude.com/docs/en/iam#configuring-permissions%23MCP))

```typescript:tools.ts
export const allowedTools = [
  "mcp__github__get_file_contents",
  "mcp__notionApi__API-post-database-query",
  "mcp__notionApi__API-post-search",
  "mcp__notionApi__API-get-block-children",
  "mcp__notionApi__API-patch-block-children",
  "mcp__notionApi__API-retrieve-a-block",
  "mcp__slack__slack_reply_to_thread",
]
```

## おわりに


## 参考資料
https://tech.layerx.co.jp/entry/config-code-generation-with-claude-code-base-action

https://speakerdeck.com/yukukotani/scale-out-your-claude-code



---
[#GauDev Advent Calendar 2025](https://adventar.org/calendars/11616)、明日の担当はYuseiWhiteさんです。
