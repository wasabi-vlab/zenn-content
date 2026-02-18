---
title: "Claude APIツール使用（Tool Use）入門｜AIに外部機能を呼ばせる方法"
emoji: "🐢"
type: "tech"
topics: ["claude", "anthropic", "ai", "llm"]
published: true
price: 200
---
わさびです。

「Claudeに天気を調べさせたい」「データベースを検索させたい」「外部APIを叩かせたい」。こういうことをやりたいとき、Tool Use（ツール使用）を使う。

Tool Useは、Claudeに「使っていいツール」を定義しておくと、必要に応じてClaudeがそのツールを呼び出す仕組み。Claudeが直接外部にアクセスするわけではなく、「このツールをこの引数で呼んでほしい」と返してくるので、実際の実行はプログラム側で行う。

## Tool Useの実行フロー

全体の流れを理解するのが最初のステップ。

1. 開発者がツールの定義（名前、説明、パラメータ）をAPIに渡す
2. ユーザーがメッセージを送る
3. Claudeが「このツールを使いたい」と返す（tool_useブロック）
4. プログラム側がツールを実行する
5. 実行結果をClaudeに返す（tool_resultブロック）
6. Claudeが結果を踏まえて最終回答を生成する

ステップ3-5が通常のチャットにはない部分。Claudeは「ツールを呼びたい」と言うだけで、実行はしない。実行して結果を返すのはプログラム側の責任。

## 基本的な実装

天気APIを呼ぶ例で説明する。

### ツールの定義

```python
import anthropic
import json

client = anthropic.Anthropic()

tools = [
    {
        "name": "get_weather",
        "description": "指定された都市の現在の天気を取得します。",
        "input_schema": {
            "type": "object",
            "properties": {
                "city": {
                    "type": "string",
                    "description": "天気を調べたい都市名（例: 東京、大阪）"
                },
                "unit": {
                    "type": "string",
                    "enum": ["celsius", "fahrenheit"],
                    "description": "温度の単位"
                }
            },
            "required": ["city"]
        }
    }
]
```

`input_schema`はJSON Schemaで書く。Claudeはこのスキーマを見て、正しい引数を組み立てる。`description`が重要で、Claudeがツールを選ぶ判断材料になる。

### メッセージ送信とツール呼び出しの処理

```python
def process_tool_call(tool_name, tool_input):
    """ツールを実際に実行する関数"""
    if tool_name == "get_weather":
        # 実際にはここで天気APIを呼ぶ
        return {"city": tool_input["city"], "temperature": 22, "condition": "晴れ"}
    return {"error": "不明なツール"}

# 初回リクエスト
response = client.messages.create(
    model="claude-sonnet-4-5-20250514",
    max_tokens=1024,
    tools=tools,
    messages=[
        {"role": "user", "content": "東京の天気を教えて"}
    ]
)

# ツール呼び出しがあるか確認
if response.stop_reason == "tool_use":
    # ツール呼び出しブロックを取得
    tool_block = next(b for b in response.content if b.type == "tool_use")

    # ツールを実行
    result = process_tool_call(tool_block.name, tool_block.input)

    # 結果をClaudeに返す
    final_response = client.messages.create(
        model="claude-sonnet-4-5-20250514",
        max_tokens=1024,
        tools=tools,
        messages=[
            {"role": "user", "content": "東京の天気を教えて"},
            {"role": "assistant", "content": response.content},
            {
                "role": "user",
                "content": [
                    {
                        "type": "tool_result",
                        "tool_use_id": tool_block.id,
                        "content": json.dumps(result, ensure_ascii=False)
                    }
                ]
            }
        ]
    )

    print(final_response.content[0].text)
```

## 複数ツールの定義

1つのリクエストに複数のツールを定義できる。Claudeが状況に応じて適切なツールを選ぶ。

```python
tools = [
    {
        "name": "get_weather",
        "description": "指定された都市の天気を取得します。",
        "input_schema": { ... }
    },
    {
        "name": "search_database",
        "description": "商品データベースを検索します。",
        "input_schema": {
            "type": "object",
            "properties": {
                "query": {"type": "string", "description": "検索キーワード"},
                "category": {"type": "string", "description": "商品カテゴリ"}
            },
            "required": ["query"]
        }
    },
    {
        "name": "send_email",
        "description": "指定されたアドレスにメールを送信します。",
        "input_schema": {
            "type": "object",
            "properties": {
                "to": {"type": "string", "description": "送信先メールアドレス"},
                "subject": {"type": "string", "description": "件名"},
                "body": {"type": "string", "description": "本文"}
            },
            "required": ["to", "subject", "body"]
        }
    }
]
```

ユーザーが「東京の天気は？」と聞けばget_weatherを、「ノートPCを探して」と言えばsearch_databaseを、Claudeが自動で選択する。

## tool_choiceで呼び出しを制御する

Claudeにツール選択の自由度を設定できる。

| tool_choice | 動作 |
|---|---|
| auto（デフォルト） | Claudeが必要に応じてツールを使う |
| any | 必ずいずれかのツールを使う |
| tool（名前指定） | 指定したツールを必ず使う |

```python
response = client.messages.create(
    model="claude-sonnet-4-5-20250514",
    max_tokens=1024,
    tools=tools,
    tool_choice={"type": "tool", "name": "get_weather"},
    messages=[...]
)
```

「必ずこのツールを使え」と強制したい場面では`tool`を指定する。

## エラーハンドリング

ツール実行が失敗した場合は、エラー情報をtool_resultで返す。

```python
{
    "type": "tool_result",
    "tool_use_id": tool_block.id,
    "content": "エラー: APIサーバーに接続できませんでした",
    "is_error": True
}
```

`is_error: True`を設定すると、Claudeはエラーを認識して「天気情報を取得できませんでした」のように適切に対応する。


:::message
ここから先は有料記事です。完全版のプロンプト・テンプレート・コード例を収録しています。
:::

===
## 実践的なユースケース

Tool Useが本領を発揮する場面:

| ユースケース | ツール例 |
|---|---|
| カスタマーサポート | 注文検索、在庫確認、返品処理 |
| データ分析 | SQL実行、グラフ生成、レポート作成 |
| 開発支援 | GitHub操作、デプロイ、ログ検索 |
| 業務自動化 | メール送信、カレンダー登録、Slack通知 |

Claude Codeが内部的にTool Useを使ってファイル操作やgitコマンドを実行しているのも、この仕組みの応用。

## OpenAI Function Callingとの比較

OpenAIのFunction Calling（現在はTool Use）とClaudeのTool Useは概念的にほぼ同じ。

| 項目 | Claude Tool Use | OpenAI Tool Use |
|---|---|---|
| ツール定義 | JSON Schema | JSON Schema |
| 並列呼び出し | 対応 | 対応 |
| 強制呼び出し | tool_choice | tool_choice |
| ストリーミング | 対応 | 対応 |

移行コストは低い。スキーマ定義の形式がほぼ同じなので、ツール定義はそのまま流用できることが多い。大きな違いはモデルの判断精度で、どちらが優れているかはタスク次第。

## まとめ

Tool Useは「AIに外部の世界を触らせる」ための仕組み。定義して、呼ばれたら実行して、結果を返す。このループを理解すれば、Claudeにほぼ何でもやらせることができる。

まずは1つのシンプルなツールから始めて、動作を確認するのがおすすめ。天気API、データベース検索、ファイル操作あたりが取り組みやすい。

僕は外部ツールがなくても甲羅があるから大丈夫だけど、Claudeにはツールを渡してあげたほうが活躍できる。

## 関連記事

その他のClaude/AI記事は [あかはらVラボ](https://akahara-vlab.com) で公開中です。


---
この記事は [あかはらVラボ](https://akahara-vlab.com) からの転載です。最新のClaude/AI情報を日本語で発信中。
