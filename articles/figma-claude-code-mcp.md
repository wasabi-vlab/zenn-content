---
title: "Figma×Claude Code連携が登場｜コードを書いたらそのままFigmaのデザインになる"
emoji: "🐢"
type: "tech"
topics: ["claude", "anthropic", "ai", "llm"]
published: true
---
わさびです。

2026年2月17日、Figma CEOのDylan Field氏がXで「Claude Code to Figma Design」を発表した。

これがなかなかすごい。Claude Codeで作ったUIを、Figma上の編集可能なデザインレイヤーにそのまま変換できる。逆に、Figmaで磨いたデザインをコードに戻すことも可能だ。

コードとデザインの行き来が、ほぼゼロコストになる。

## Code to Canvas とは

Figmaが発表した新機能「Code to Canvas」は、Claude Codeで構築した動作するUIをFigmaのキャンバスに取り込む仕組みだ。

やることは簡単。Figma MCPサーバーをインストールして、Claude Codeで「Send this to Figma」と打つだけ。ブラウザでレンダリングされた状態がそのまま、Figmaの編集可能なレイヤーに変換される。

つまり、AIでプロトタイプを作って、デザイナーがFigma上で仕上げるという流れが自然にできる。

## 実例で見る3つの使い方

実際にどんなことができるのか、3つのパターンで見てみよう。

### パターン1: コードで作ったUIをFigmaに送る

Claude Codeでログインフォームを作って、「Send this to Figma」と打つ。すると、ボタンやテキストフィールドがFigmaの個別レイヤーとして展開される。


デザイナーはここから色やフォントを調整できる。コードを読む必要がない。

### パターン2: FigmaのデザインからReactコンポーネントを生成

逆パターン。Figma上でデザインしたカードコンポーネントを選択して、Claude Codeに「これをReact + Tailwindで実装して」と依頼する。


Figmaのスタイル値（角丸、色、フォントサイズ）を自動で読み取って、コードに反映してくれる。手動でピクセル値を拾う作業がなくなる。

### パターン3: デザイントークンを自動抽出

FigmaファイルのURLを渡して「デザイントークンをTailwind configに変換して」と指示する。Figmaの変数コレクションから色・角丸・フォントサイズを引っ張ってきて、設定ファイルを自動生成する。


デザインシステムの更新があっても、Figmaとコードでトークンがずれなくなる。

## 仕組み：MCPで繋がる

この連携はMCP（Model Context Protocol）で実現されている。MCPはAnthropicが提唱したオープン標準で、AIツールと外部アプリケーションを接続するためのプロトコルだ。

```
Claude Code（UIをコードで構築）
    ↕ MCP プロトコル
  Figma MCP Server
    ↕
  Figma キャンバス（編集可能なレイヤー）
```

Figma MCPサーバーには2つの導入方法がある。

| 方式 | 特徴 |
|------|------|
| リモートサーバー | Figmaデスクトップアプリ不要。クラウド経由で接続 |
| デスクトップサーバー | ローカルのFigmaアプリ経由（localhost:3845） |

リモートサーバーの場合、Claude Codeで以下を実行するだけでセットアップ完了。

```
claude mcp add --transport http figma-remote-mcp https://mcp.figma.com/mcp
```

認証はClaude Code上で `/mcp` → figma選択 → Authenticate → Allow Accessの流れだ。

## なぜこれが大きいのか

これまで、デザイナーとエンジニアの間には「デザイン→実装」の一方通行しかなかった。デザインカンプをもらって実装し、修正があればデザイナーに戻す。この往復に時間がかかっていた。

Code to Canvasは、この壁を壊す。

エンジニアがClaude Codeでプロトタイプを作り、それをFigmaに送る。デザイナーがFigma上で調整し、その結果をまたコードに反映する。どちらの方向からでも始められる。

Dylan Field氏はCNBCの取材で、AIが高速にソフトウェアを生み出す時代だからこそ、デザインの視点やクラフトが差別化の鍵になると語っている。「Code to Canvas」はその思想を形にしたものだ。

## MCPエコシステムの広がり

Figmaの参入は、MCPエコシステムにとっても大きな追い風だ。

MCPはすでにWordPressやGitHub、Slackなど多くのサービスで採用が進んでいる。Figmaという巨大デザインツールが加わったことで、開発ワークフロー全体がMCPで繋がる未来がさらに近づいた。

| MCP対応サービス | 用途 |
|----------------|------|
| Figma | UIデザイン ↔ コード変換 |
| WordPress | コンテンツ管理・記事投稿 |
| GitHub | コードリポジトリ操作 |
| Slack | チーム通知・連携 |

1つのAIエージェントから、デザイン・実装・デプロイ・コンテンツ管理まで全部つながる。これがMCPの目指す世界だ。

## 関連記事

その他のClaude/AI記事は [あかはらVラボ](https://akahara-vlab.com) で公開中です。


---
この記事は [あかはらVラボ](https://akahara-vlab.com) からの転載です。最新のClaude/AI情報を日本語で発信中。
