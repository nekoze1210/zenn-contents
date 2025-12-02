---
title: "プロダクトのAI開発におけるラストワンマイル活用"
emoji: "🧭"
type: "tech"
topics: ["LLM", "Claude Code", "GitHub Actions", "GauDev" , "アドベントカレンダー"]
published: true
published_at: 2025-12-05 00:00
---

この記事は[#GauDev Advent Calendar 2025](https://adventar.org/calendars/11616)の5日目です。

---

今年は AI 開発が加速する1年でした。Claude Code や MCP(Model Context Protocol) が公開されてから約半年しか経っていないだなんて、信じられませんね...

Gaudiy でも各チームで AI 開発を積極的に取り入れてきました。今回の記事では、その一部であるAI開発におけるラストワンマイル活用として試していることをご紹介します。

## AI 開発におけるラストワンマイルとは 
Claude Code をはじめとするAIコーディングエージェントの強みは、CLAUDE.md(AGENTS.md)を起点にプロダクトに関するコンテキストを持った働きができるということが挙げられます。
しかし、すべての職能の人が AIコーディングエージェント を使える環境を整備できないため、プロダクトのコンテキストをもったAI活用ができません。[^1]

コンテキストを持っていない LLM アプリケーション(Chat GPT, Gemini, Claude など) にプロダクトのことを聞いてもハルシネーションを起こすか、分からないと回答が返ってくるだけです。

このような障壁が存在するため、以下の課題がプロダクトの保守・運用の現場では起きていました。

- プロダクトの仕様確認がChat GPTをはじめとするデスクトップアプリではできないため、エンジニアに問い合わせる必要がある。
- 不具合発生時に原因調査をエンジニアに毎回お願いする必要があり、さらにドメインエキスパートが属人的。
- コミュニティ運用時に発生した障害のポストモーテムの記入が手動で、運用負荷が高い。(Slackでやりとりしている内容をそのまま要約できることが理想)
- etc...

このように、**AIコーディングエージェントが持つ「プロダクトコンテキストを活用できる」という強みを、エンジニア以外の職能にも届けること = ラストワンマイル** が、プロダクト運用における課題解決の鍵となります。

[^1]: 厳密には Claude Code Web や GitHub MCP を活用してアクセスできますが、そのためには GitHub アカウントの作成や MCP の設定など障壁が多くあるため、すべての人が実現できるには現実的ではない、という考えを前提に置いています。

## Claude Code GitHub Action を操る Slack Bot の導入
では実際にどのようにラストワンマイルを実現するか？
様々な手法を検討したところ、 Claude Code GitHub Action と Slack App の組み合わせに着地しました。

[Claude Code GitHub Action ](https://code.claude.com/docs/github-actions) とは GitHub Actions 上で Claude Code を実行することができるアクションです。


```yaml:claude-code-slack-action.yml
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
            --allowedTools "Bash,View,Read,Write,Edit,GlobTool,GrepTool,BatchTool,Task,${{ inputs.mcp_tools }}"
            --mcp-config '{"mcpServers":{"slack":{"command":"npx","args":["-y","@zencoderai/slack-mcp-server"],"env":{"SLACK_BOT_TOKEN":"${{secrets.SLACK_BOT_TOKEN}}","SLACK_TEAM_ID":"E08RYJJJNLX"}}}}'
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
</details>

```typescript:tools.ts
export const allowedTools = [
  "mcp__github__create_or_update_file",
  "mcp__github__search_repositories",
  "mcp__github__create_repository",
  "mcp__github__get_file_contents",
  "mcp__notionApi__API-post-database-query",
  "mcp__notionApi__API-post-search",
  "mcp__notionApi__API-get-block-children",
  "mcp__notionApi__API-patch-block-children",
  "mcp__notionApi__API-retrieve-a-block",
  // 中略
  "mcp__slack__slack_reply_to_thread",
]
```

```typescript:index.ts
import { Octokit } from "@octokit/rest";
import { allowedTools } from ./tools";

new Octokit({
  auth: env.GITHUB_TOKEN,
});

// octokit で workflow を実行する
await octokit.actions.createWorkflowDispatch({
  owner: 'owner',
  repo: 'repository',
  workflow_id: 'claude-code-slack-action.yml',
  ref: 'main',
  inputs: {
    prompt: input.promptText,
    slack_channel: input.channel,
    slack_thread_ts: input.thread_ts,
    mcp_tools: allowedTools.join(','),
  }
});
```


## おわりに


## 参考資料
https://tech.layerx.co.jp/entry/config-code-generation-with-claude-code-base-action

https://speakerdeck.com/yukukotani/scale-out-your-claude-code

---
[#GauDev Advent Calendar 2025](https://adventar.org/calendars/11616)、明日の担当はYuseiWhiteさんです。
