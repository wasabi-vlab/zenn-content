---
title: "Claude MCP（Model Context Protocol）完全ガイド｜外部ツール連携の新標準"
emoji: "🐢"
type: "tech"
topics: ["claude", "anthropic", "ai", "llm"]
published: true
---
わさびです。

MCPという言葉を聞いたことがあるだろうか。Model Context Protocol。Anthropicが策定したオープンプロトコルで、AIと外部ツール・データソースを接続するための仕組みだ。

この技術はClaude周辺で急速に広まっている。ただ、ドキュメントの大半が英語のみ。日本語の情報がほとんどない。

だからこの記事では、MCPの仕組みから実際の設定方法、自作サーバーの作り方まで、まとめて日本語で解説する。

## MCPとは何か

MCP（Model Context Protocol）は、AIモデルが外部のツールやデータソースと安全にやり取りするためのオープンプロトコル。Anthropicが2024年に公開した。

従来のやり方だと、AIに外部ツールを使わせるには個別にAPI連携を実装する必要があった。SlackならSlack API、GitHubならGitHub API、データベースならそのドライバ、という具合に。

MCPはこれを標準化する。どんなツールでも同じプロトコルで接続できるようにする仕組みだ。

よく使われる例えが「AIにとってのUSB-C」。USBがどんな周辺機器でも同じ端子で接続できるのと同じように、MCPはどんな外部ツールでも同じプロトコルで接続できるようにする。

## MCPのアーキテクチャ

MCPは3つの要素で構成されている。

| 要素 | 役割 | 具体例 |
|------|------|--------|
| MCP Host | AIが動いているアプリケーション | Claude Desktop、Claude Code |
| MCP Client | Host内でServerと通信するコンポーネント | Host内部で自動的に動く |
| MCP Server | ツールやデータを提供するプロセス | filesystem、GitHub、Slack等 |

通信の流れはシンプルだ。

```
MCP Host (Claude Desktop等)
  └─ MCP Client
       ↕ JSON-RPC 2.0
     MCP Server (ツール提供)
       └─ 外部サービス / ローカルリソース
```

MCP HostがMCP Serverを起動して、JSON-RPCで通信する。Serverは独立したプロセスとして動くので、Host側のセキュリティを保ちつつ外部リソースにアクセスできる。

## MCPの3つの主要概念

MCP Serverが提供できるものは3種類ある。

### Tools（ツール）

AIが呼び出せる関数。たとえば「ファイルを読む」「データベースにクエリを投げる」「Slackにメッセージを送る」といった操作。

AIが「この状況ではこのToolを使うべきだ」と判断して、自動的に呼び出す。

### Resources（リソース）

AIが参照できるデータ。ファイルの内容、データベースのスキーマ、APIのレスポンスなど。Toolsが「操作」なのに対して、Resourcesは「読み取り専用のデータ提供」。

### Prompts（プロンプトテンプレート）

MCP Serverが提供する定型のプロンプト。特定のタスクに最適化された指示テンプレートをServer側で用意しておける。

実際の利用では、Toolsを使う場面が圧倒的に多い。ResourcesとPromptsは補助的な役割。

## Claude DesktopでMCPを使う

Claude DesktopはMCP Hostとして動作する。設定ファイルにMCP Serverの情報を書くだけで使える。

### 設定ファイルの場所

| OS | パス |
|----|------|
| macOS | ~/Library/Application Support/Claude/claude_desktop_config.json |
| Windows | %APPDATA%\Claude\claude_desktop_config.json |

### 設定例

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-filesystem",
        "/Users/username/Documents"
      ]
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "ghp_xxxxx"
      }
    }
  }
}
```

各エントリの構造はこうなっている。

| フィールド | 説明 |
|-----------|------|
| command | MCP Serverの起動コマンド |
| args | コマンドの引数 |
| env | 環境変数（APIキーなど） |

設定ファイルを保存したらClaude Desktopを再起動する。チャット画面の入力欄にハンマーアイコンが表示されれば、MCP Serverが認識されている。

## Claude CodeでMCPを使う

Claude Codeの場合は2つの方法がある。

### 方法1: settings.jsonで設定

`.claude/settings.json` にMCP Serverを記述する。

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "ghp_xxxxx"
      }
    }
  }
}
```

### 方法2: CLIで追加

```
claude mcp add github -- npx -y @modelcontextprotocol/server-github
```

このコマンドで設定ファイルに追記される。管理が楽なのでこちらがおすすめ。

### スコープの指定

Claude Codeでは設定のスコープを選べる。

| スコープ | 保存先 | 用途 |
|---------|--------|------|
| local | .claude/settings.local.json | このプロジェクトの自分用 |
| project | .claude/settings.json | プロジェクト共有（gitに含める） |
| user | ~/.claude/settings.json | 全プロジェクト共通 |

```
claude mcp add --scope user github -- npx -y @modelcontextprotocol/server-github
```

## 代表的なMCP Server

公式・コミュニティから多くのMCP Serverが公開されている。特に使用頻度が高いものを紹介する。

| Server | 提供元 | できること |
|--------|-------|-----------|
| filesystem | 公式 | ファイルの読み書き、ディレクトリ操作 |
| github | 公式 | リポジトリ操作、Issue、PR管理 |
| slack | 公式 | チャンネル取得、メッセージ送信 |
| postgres / sqlite | 公式 | データベースクエリの実行 |
| puppeteer | 公式 | ブラウザ操作、スクリーンショット |
| brave-search | 公式 | Web検索 |
| memory | 公式 | 知識グラフによる永続記憶 |
| fetch | 公式 | Webページの取得 |

npmで公開されているものが多く、`npx -y @modelcontextprotocol/server-xxx` の形式でインストールなしに使える。

GitHubの `modelcontextprotocol/servers` リポジトリに公式サーバーの一覧がある。コミュニティ製も含めると数百のServerが存在する。

## 自作MCP Serverを作る（Python）

既存のServerで足りない場合は自作できる。Python SDKを使えば数十行で書ける。

### インストール

```
pip install mcp
```

### 最小構成のServer

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("my-tools")

@mcp.tool()
def get_weather(city: str) -> str:
    """指定した都市の天気を取得する"""
    # 実際にはAPIを叩く処理
    return f"{city}の天気: 晴れ、気温15度"

@mcp.tool()
def calculate(expression: str) -> str:
    """数式を計算する"""
    result = eval(expression)  # 本番では安全な評価器を使うこと
    return str(result)

if __name__ == "__main__":
    mcp.run()
```

`@mcp.tool()` デコレータを付けた関数がToolとして公開される。関数のdocstringがToolの説明になり、AIがToolを選ぶ判断材料になる。

### 自作Serverの登録

Claude Desktopの場合:

```json
{
  "mcpServers": {
    "my-tools": {
      "command": "python",
      "args": ["path/to/my_server.py"]
    }
  }
}
```

Claude Codeの場合:

```
claude mcp add my-tools -- python path/to/my_server.py
```

### Resourceの追加

読み取り専用のデータを提供する場合は`@mcp.resource()`を使う。

```python
@mcp.resource("config://app")
def get_config() -> str:
    """アプリケーション設定を返す"""
    return json.dumps({"version": "1.0", "debug": False})
```

TypeScript SDKもあり、`@modelcontextprotocol/sdk` パッケージで同様のことができる。

## MCPと従来のAPI連携の違い

「REST APIを直接叩けばいいのでは」と思うかもしれない。MCPにはAPI直接呼び出しにはないメリットがある。

| 観点 | 従来のAPI連携 | MCP |
|------|-------------|-----|
| 発見性 | AIがどのAPIを使えるか事前に知る必要がある | ServerがTool一覧を動的に提供する |
| 標準化 | API毎に認証方式・データ形式が異なる | 全てJSON-RPCで統一 |
| セキュリティ | AIにAPIキーを直接渡す | Server側で認証を管理、AIには公開しない |
| 拡張性 | 新しいAPIへの対応は個別実装 | Serverを追加するだけ |
| 型安全性 | ドキュメントに依存 | JSON Schemaで引数・戻り値の型を定義 |

特にセキュリティの面は大きい。MCPではAPIキーやデータベース接続情報はServer側が保持する。AI側には一切渡さない。これはプロダクション環境では重要な設計だ。

## MCPが重要な理由

MCPの本質は「AIのツール利用を標準化した」ことにある。

これまでAIが外部ツールを使う場合、各サービスが独自のプラグイン仕様やAPI連携を用意していた。OpenAIのFunction Calling、ChatGPTのプラグイン、各種APIラッパーライブラリ。どれもそのプラットフォーム専用。

MCPはオープンプロトコルだから、特定のAIプロバイダーに縛られない。Anthropic以外のAIでも採用が進んでいる。

一度MCP Serverを作れば、Claude Desktop、Claude Code、その他のMCP対応クライアント全てで使い回せる。これが「USB-C」と例えられる理由だ。

## 日本語情報が少ない現状

MCPの公式ドキュメントは https://modelcontextprotocol.io にあるが、全て英語だ。

GitHubの各Serverリポジトリも英語のみ。日本語でまとまった情報を提供しているサイトはまだ少ない。

このブログでは今後もMCPに関する情報を日本語で発信していく予定。MCPの新しいServerの紹介や、実際のユースケースも取り上げていきたい。

## まとめ

| 項目 | 内容 |
|------|------|
| MCPとは | AIと外部ツールを接続するオープンプロトコル |
| 開発元 | Anthropic |
| 対応Host | Claude Desktop、Claude Code 他 |
| 主要概念 | Tools（操作）、Resources（データ）、Prompts（テンプレート） |
| 自作方法 | Python/TypeScript SDKで数十行から作成可能 |
| 強み | 標準化・発見性・セキュリティ |

MCPはAIの「できること」を大幅に拡張する仕組みだ。これまでAIは学習済みの知識で答えるだけだったが、MCPを通じてリアルタイムなデータ取得、外部サービスの操作、ローカルファイルへのアクセスが可能になる。

Claude DesktopやClaude Codeを使っているなら、まずはfilesystemサーバーあたりから試してみるのがおすすめ。設定は数行で終わる。

## 関連記事

その他のClaude/AI記事は [あかはらVラボ](https://akahara-vlab.com) で公開中です。


---
この記事は [あかはらVラボ](https://akahara-vlab.com) からの転載です。最新のClaude/AI情報を日本語で発信中。
