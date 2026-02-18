---
title: "Claude Agent SDK入門｜自律型AIエージェントを構築する方法"
emoji: "🐢"
type: "tech"
topics: ["claude", "anthropic", "ai", "llm"]
published: true
price: 200
---
わさびです。

「AIに指示を出したら、自分で考えて、ツールを使って、最後まで仕事を完了してほしい」。

これを実現するのがAIエージェント。そしてAnthropicが公開したAgent SDKは、Claudeベースのエージェントを構築するためのオープンソースフレームワーク。

実はClaude Code自体がこのAgent SDKで構築されている。つまりAgent SDKは、実績のあるフレームワーク。

## AIエージェントとは

チャットボットとエージェントの違い:

| 項目 | チャットボット | エージェント |
|------|-------------|------------|
| 入力 | 質問 | 目標 |
| 処理 | 1回のLLM呼び出し | ループ（複数回のLLM呼び出し） |
| 行動 | テキストを返す | ツールを使って作業する |
| 自律性 | なし | あり（自分で判断して次の行動を選ぶ） |

エージェントは「目標を与えると、自分で計画を立てて、ツールを使って、結果を確認して、必要なら修正して、完了するまで繰り返す」という動きをする。

## Agent SDKの概要

Agent SDKはPythonライブラリ。主要な構成要素:

| コンポーネント | 役割 |
|-------------|------|
| Agent | エージェントの定義（モデル、ツール、指示） |
| Tool | エージェントが使える道具 |
| Runner | エージェントループの実行 |
| Guardrails | 安全装置（入力/出力のバリデーション） |
| Handoff | エージェント間のタスク委譲 |

### インストール

```bash
pip install anthropic-agent-sdk
```

## エージェントループの仕組み

Agent SDKの核心は「エージェントループ」。以下の処理を繰り返す:

1. LLMにメッセージを送る
2. LLMがテキストを返す → 終了
3. LLMがツール呼び出しを返す → ツールを実行 → 結果をLLMに渡す → 1に戻る

```
[ユーザー入力]
    ↓
[LLM呼び出し]
    ↓
テキスト応答？ → 出力して終了
ツール呼び出し？ → ツール実行 → 結果をLLMに戻す → [LLM呼び出し]に戻る
```

このループがエージェントの「自律性」の正体。LLMが「次に何をすべきか」を判断し、ツールを使い、結果を見てまた判断する。人間が介入しなくても、目標達成まで自走する。

## 基本的なエージェントの定義

```python
from agent_sdk import Agent, Runner

agent = Agent(
    name="research_assistant",
    model="claude-sonnet-4-5",
    instructions="あなたはリサーチアシスタントです。ユーザーの質問に対して、利用可能なツールを使って調査し、正確な回答を提供してください。",
)

result = Runner.run_sync(agent, "Pythonの最新バージョンは？")
print(result.final_output)
```

`instructions`はシステムプロンプトに相当する。エージェントの役割と行動指針を定義する。

## Tool定義

エージェントに使わせるツールを定義する。

### 関数ベースのツール

```python
from agent_sdk import Agent, Runner, function_tool

@function_tool
def search_web(query: str) -> str:
    """Web検索を実行して結果を返す。

    Args:
        query: 検索クエリ
    """
    # 実際の検索ロジック
    # ここではダミー
    return f"'{query}' の検索結果: Python 3.13が最新版です。"

@function_tool
def read_file(path: str) -> str:
    """ファイルの内容を読み取る。

    Args:
        path: ファイルパス
    """
    with open(path, encoding="utf-8") as f:
        return f.read()

agent = Agent(
    name="researcher",
    model="claude-sonnet-4-5",
    instructions="質問に答えるために、Web検索やファイル読み取りを活用してください。",
    tools=[search_web, read_file],
)

result = Runner.run_sync(agent, "Pythonの最新バージョンを調べて")
print(result.final_output)
```

`@function_tool`デコレータをつけた関数がツールになる。関数のdocstringとType Hintsから、ツールの説明とパラメータスキーマが自動生成される。

### ツール設計のコツ

- ツール名は動詞で始める（search_web, read_file, send_email）
- docstringは具体的に書く（LLMがツールの使い方を判断する材料になる）
- 戻り値は文字列にする（LLMが読める形式）
- エラーは例外ではなくエラーメッセージとして返す（ループが止まらないように）

## Guardrails（安全装置）

エージェントが暴走しないための仕組み。

### 入力ガードレール

ユーザー入力をチェックして、不適切なリクエストを弾く:

```python
from agent_sdk import Agent, Runner, InputGuardrail, GuardrailResult

async def check_input(input_text: str) -> GuardrailResult:
    # 危険なキーワードのチェック
    blocked_keywords = ["パスワード削除", "全データ消去", "rm -rf"]
    for keyword in blocked_keywords:
        if keyword in input_text:
            return GuardrailResult(
                should_block=True,
                message="この操作は許可されていません。"
            )
    return GuardrailResult(should_block=False)

agent = Agent(
    name="assistant",
    model="claude-sonnet-4-5",
    instructions="ユーザーのタスクを支援します。",
    input_guardrails=[
        InputGuardrail(guardrail_function=check_input)
    ],
)
```

### 出力ガードレール

エージェントの出力をチェックして、不適切な回答を修正またはブロックする:

```python
from agent_sdk import OutputGuardrail, GuardrailResult

async def check_output(output_text: str) -> GuardrailResult:
    # 個人情報の漏洩チェック
    import re
    if re.search(r"\d{3}-\d{4}-\d{4}", output_text):
        return GuardrailResult(
            should_block=True,
            message="回答に電話番号が含まれていたため、ブロックしました。"
        )
    return GuardrailResult(should_block=False)
```

Guardrailsは本番運用では必須。エージェントは自律的に動くからこそ、安全装置が重要になる。

## マルチエージェント構成

複数のエージェントを組み合わせて、複雑なタスクをこなす。

### ハンドオフパターン

あるエージェントが別のエージェントにタスクを委譲する:

```python
from agent_sdk import Agent, Runner

# 専門エージェントたち
code_agent = Agent(
    name="code_expert",
    model="claude-sonnet-4-5",
    instructions="Pythonのコーディングに関する質問に答えます。コード例を含めて回答してください。",
)

docs_agent = Agent(
    name="docs_expert",
    model="claude-sonnet-4-5",
    instructions="ドキュメント作成に関する質問に答えます。Markdown形式で回答してください。",
)

# トリアージエージェント（振り分け役）
triage_agent = Agent(
    name="triage",
    model="claude-haiku-4-5",
    instructions="""ユーザーの質問を分析して、適切な専門エージェントに引き継いでください。
- コーディングの質問 → code_expertに引き継ぐ
- ドキュメントの質問 → docs_expertに引き継ぐ""",
    handoffs=[code_agent, docs_agent],
)

result = Runner.run_sync(triage_agent, "PythonでFastAPIのルーティングを書いて")
print(result.final_output)
```

トリアージエージェントはHaikuで安く動かして、専門的な処理はSonnetに委譲する。モデルの使い分けでコストを最適化できる。

### パイプラインパターン

エージェントを直列につなげて、段階的に処理する:

```python
from agent_sdk import Agent, Runner

# ステップ1: リサーチ
research_agent = Agent(
    name="researcher",
    model="claude-sonnet-4-5",
    instructions="与えられたトピックについて調査し、要点を箇条書きでまとめてください。",
    tools=[search_web],
)

# ステップ2: 執筆
writer_agent = Agent(
    name="writer",
    model="claude-sonnet-4-5",
    instructions="提供された要点をもとに、ブログ記事を執筆してください。",
)

# ステップ1を実行
research_result = Runner.run_sync(research_agent, "2026年のAIトレンド")

# ステップ2にステップ1の結果を渡す
article = Runner.run_sync(writer_agent, f"以下の要点をもとに記事を書いて:\n{research_result.final_output}")
print(article.final_output)
```

## Claude Codeとの関係

Claude Code自体がAgent SDKで構築されている。Claude Codeがやっていることを分解すると:

- ファイルの読み書き → Tool
- gitの操作 → Tool
- ターミナルコマンド実行 → Tool
- ユーザーとの対話 → エージェントループ
- CLAUDE.md → instructions
- Permission modes → Guardrails

つまりAgent SDKを使えば、Claude Codeのような「ファイルを操作して、テストを走らせて、コミットする」エージェントを自分で作れる。

もちろんClaude Codeほどの完成度にするには膨大な作り込みが必要だけど、特定用途に絞ったエージェントなら実用的なものが作れる。


:::message
ここから先は有料記事です。完全版のプロンプト・テンプレート・コード例を収録しています。
:::

===
## 実装のポイント

### ツールの粒度

ツールは細かすぎても粗すぎてもいけない。

- 細かすぎ: `read_line`, `write_line`, `move_cursor` → LLMが何十回もツールを呼ぶ
- 粗すぎ: `do_everything` → 柔軟性がない
- ちょうどいい: `read_file`, `write_file`, `search_code` → 1回の呼び出しで意味のある結果

### ループの上限

エージェントループには必ず上限を設ける。無限ループを防ぐため。

```python
result = Runner.run_sync(agent, "タスクを完了して", max_turns=20)
```

20ターンで終わらないタスクは、タスクの分解が不十分か、ツールが足りない可能性がある。

### コスト管理

エージェントはループするので、1回のタスクで複数回のAPI呼び出しが発生する。コストの見積もりは「1ターンのコスト x 予想ターン数」で計算する。

トリアージにHaiku、専門処理にSonnetという構成なら、コストを抑えつつ品質を維持できる。

## まとめ

Agent SDKは「Claude APIを使ったエージェント構築」を標準化するフレームワーク。

- エージェントループ: LLM呼び出し → ツール実行 → 繰り返し
- Tool: 関数デコレータで簡単に定義
- Guardrails: 入力/出力の安全チェック
- Handoff: エージェント間のタスク委譲
- Claude Codeが実証済みのアーキテクチャ

自律的に動くAIを作りたいなら、Agent SDKは有力な選択肢。特にClaudeのTool Useに慣れている人は、自然に入れると思う。

自律型AIエージェントと言われると大げさだけど、要は「ループして、ツール使って、自分で判断する」だけ。カメだって自分でエサを探して食べるわけで、自律行動の基本は同じ。

## 関連記事

その他のClaude/AI記事は [あかはらVラボ](https://akahara-vlab.com) で公開中です。


---
この記事は [あかはらVラボ](https://akahara-vlab.com) からの転載です。最新のClaude/AI情報を日本語で発信中。
