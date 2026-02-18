---
title: "ClaudeでRAGを実装する｜検索拡張生成の設計パターンと実践例"
emoji: "🐢"
type: "tech"
topics: ["claude", "anthropic", "ai", "llm"]
published: true
price: 200
---
わさびです。

LLMに「自社のドキュメントを読ませて質問に答えさせたい」。この要望を実現するのがRAG（Retrieval-Augmented Generation / 検索拡張生成）。

Claudeは200Kトークンのコンテキストウィンドウを持っている。これは日本語で約10万文字、新書3〜4冊分。この巨大なコンテキストを活かしたRAG設計は、他のLLMとは違うアプローチが取れる。

## RAGの基本概念

RAGは3ステップで動く:

1. ユーザーの質問を受け取る
2. 関連するドキュメントを検索する（Retrieval）
3. 検索結果をプロンプトに含めてLLMに回答させる（Generation）

LLMは学習データに含まれない情報（社内文書、最新ニュース等）を知らない。検索で補完してあげることで、正確な回答が可能になる。

よくある誤解:「ファインチューニングで自社データを学習させればいい」。これは大抵うまくいかない。ファインチューニングはLLMの「スタイル」を変えるもので、「知識」を追加するのはRAGの役割。

## Claudeの200Kコンテキストを活かす設計

従来のRAGは「コンテキストウィンドウが小さいから、検索精度が超重要」だった。4Kトークンしかなければ、本当に関連する数チャンクしか入れられない。

Claudeは200Kトークン。これは設計を根本的に変える。

| アプローチ | 従来のRAG | Claude向けRAG |
|-----------|----------|-------------|
| コンテキスト | 4K〜8K | 200K |
| 検索精度の重要度 | 極めて高い | 中程度（量で補える） |
| チャンク数 | 3〜5個 | 50〜100個でも可 |
| リランキング | ほぼ必須 | なくても動く |

つまり「検索でちょっと多めに引っかけて、全部Claudeに渡す」という力技が通用する。検索精度のチューニングに費やす時間を大幅に減らせる。

ただし200Kに詰め込むほどコストは上がる。精度とコストのバランスは考える必要がある。

## embeddingの選択肢

検索の基盤になるのがembedding（埋め込みベクトル）。テキストを数値のベクトルに変換して、ベクトル間の距離で類似度を測る。

### 主要なembeddingモデル

| モデル | 提供元 | 次元数 | 日本語対応 | 料金 |
|--------|--------|--------|----------|------|
| text-embedding-3-large | OpenAI | 3,072 | 良好 | $0.13/100万tok |
| text-embedding-3-small | OpenAI | 1,536 | 良好 | $0.02/100万tok |
| nomic-embed-text | Nomic (Ollama) | 768 | まあまあ | 無料（ローカル） |
| multilingual-e5-large | Microsoft | 1,024 | 良好 | 無料（ローカル） |

Anthropicは独自のembeddingモデルを提供していない。外部のembeddingモデルと組み合わせて使う。

コストを抑えるならOllamaでnomic-embed-textをローカル実行するのがおすすめ。品質を優先するならOpenAIのtext-embedding-3-largeが安定している。

## チャンク戦略

ドキュメントをどう分割（チャンク）するかで検索精度が変わる。

### チャンクサイズの目安

| サイズ | 向いているケース |
|--------|--------------|
| 200〜500トークン | FAQ、短い項目 |
| 500〜1,000トークン | 技術文書、マニュアル |
| 1,000〜2,000トークン | 長い記事、論文 |

### 分割の方法

単純な文字数分割はやめたほうがいい。文の途中で切れると検索精度が落ちる。

おすすめの分割方法:

- Markdownの見出し（##）で分割
- 段落（空行2つ）で分割
- 意味的なまとまりで分割

### オーバーラップ

チャンク同士に50〜100トークンの重複を持たせると、文脈が途切れにくくなる。

```python
def chunk_text(text, chunk_size=1000, overlap=100):
    chunks = []
    start = 0
    while start < len(text):
        end = start + chunk_size
        chunks.append(text[start:end])
        start = end - overlap
    return chunks
```


:::message
ここから先は有料記事です。完全版のプロンプト・テンプレート・コード例を収録しています。
:::

===
## プロンプト設計

検索結果をClaudeに渡すときのプロンプト設計が、RAGの品質を決める。

### 基本パターン

```python
system_prompt = """あなたは社内文書に基づいて質問に回答するアシスタントです。

ルール:
- 提供されたコンテキストに基づいてのみ回答する
- コンテキストに情報がない場合は「この情報は見つかりませんでした」と答える
- 回答の根拠となるドキュメント名を明記する"""

user_prompt = f"""以下のコンテキストを参照して、質問に答えてください。

<context>
{retrieved_documents}
</context>

質問: {user_question}"""
```

ポイント:

- XMLタグでコンテキストを明確に区切る（Claudeはタグ構造に強い）
- 「コンテキストにない情報は答えない」を明記する（ハルシネーション防止）
- ソースの明記を求める（回答の信頼性向上）

### 複数ドキュメントの渡し方

```python
context_parts = []
for i, doc in enumerate(retrieved_docs):
    context_parts.append(f"""<document index="{i+1}" source="{doc['source']}">
{doc['content']}
</document>""")

context = "\n\n".join(context_parts)
```

各ドキュメントにインデックスとソース情報をつけると、Claudeが引用しやすくなる。

## citation機能

Claude APIにはcitation（引用）機能がある。回答の中でどのドキュメントのどの部分を参照したかを構造化データで返してくれる。

```python
import anthropic

client = anthropic.Anthropic()

response = client.messages.create(
    model="claude-sonnet-4-5",
    max_tokens=1024,
    messages=[
        {
            "role": "user",
            "content": [
                {
                    "type": "document",
                    "source": {
                        "type": "text",
                        "media_type": "text/plain",
                        "data": "社内規定: 有給休暇は年間20日付与される。..."
                    },
                    "title": "就業規則.txt",
                    "citations": {"enabled": True}
                },
                {
                    "type": "text",
                    "text": "有給休暇は何日もらえますか？"
                }
            ]
        }
    ]
)

for block in response.content:
    if block.type == "text":
        print(block.text)
        if hasattr(block, "citations") and block.citations:
            for cite in block.citations:
                print(f"  出典: {cite.document_title} - {cite.cited_text}")
```

citation機能を使うと、ハルシネーション検出がしやすくなる。回答に引用が付いていなければ、LLMが知識から答えている（＝不正確な可能性がある）と判断できる。

## MCP連携

MCP（Model Context Protocol）を使えば、Claude DesktopやClaude Codeから直接ベクトルDBに問い合わせるRAGシステムを構築できる。

MCPサーバーとして検索機能を公開すれば、Claudeが自分で「必要な情報を検索する」ようになる。人間がプロンプトに検索結果を貼り付ける必要がなくなる。

詳しくは[Claude MCP完全ガイド](https://akahara-vlab.com/claude-mcp-guide//)を参照。

## Python実装例

最小構成のRAGシステム:

```python
import anthropic
import numpy as np
from pathlib import Path

# --- 1. ドキュメントの読み込みとチャンク化 ---
def load_and_chunk(directory: str, chunk_size: int = 800) -> list[dict]:
    chunks = []
    for path in Path(directory).glob("*.md"):
        text = path.read_text(encoding="utf-8")
        # 見出しで分割
        sections = text.split("\n## ")
        for section in sections:
            if len(section.strip()) > 50:
                chunks.append({
                    "source": path.name,
                    "content": section.strip()
                })
    return chunks

# --- 2. embeddingの生成（OpenAI） ---
from openai import OpenAI

openai_client = OpenAI()

def get_embeddings(texts: list[str]) -> list[list[float]]:
    response = openai_client.embeddings.create(
        model="text-embedding-3-small",
        input=texts
    )
    return [item.embedding for item in response.data]

# --- 3. 検索 ---
def search(query: str, chunks: list[dict], embeddings: np.ndarray, top_k: int = 10) -> list[dict]:
    query_embedding = get_embeddings([query])[0]
    query_vec = np.array(query_embedding)

    # コサイン類似度
    similarities = np.dot(embeddings, query_vec) / (
        np.linalg.norm(embeddings, axis=1) * np.linalg.norm(query_vec)
    )

    top_indices = np.argsort(similarities)[-top_k:][::-1]
    return [chunks[i] for i in top_indices]

# --- 4. 回答生成 ---
claude_client = anthropic.Anthropic()

def answer(query: str, retrieved: list[dict]) -> str:
    context = "\n\n".join([
        f"[{doc['source']}]\n{doc['content']}" for doc in retrieved
    ])

    response = claude_client.messages.create(
        model="claude-sonnet-4-5",
        max_tokens=1024,
        system="提供されたコンテキストに基づいて回答してください。情報がない場合はその旨伝えてください。",
        messages=[
            {
                "role": "user",
                "content": f"<context>\n{context}\n</context>\n\n質問: {query}"
            }
        ]
    )
    return response.content[0].text
```

本番では、embeddingの保存にベクトルDB（Chroma、Pinecone、pgvector等）を使う。上の例はnumpyで素朴にやっているが、数千件までならこれで十分動く。

## よくある失敗パターン

### 1. チャンクが大きすぎる

5,000トークンのチャンクだと、ノイズが多くて精度が下がる。500〜1,000トークンに分割する。

### 2. メタデータを無視している

チャンクに「いつの情報か」「どのドキュメントか」のメタデータがないと、古い情報と新しい情報の区別がつかない。

### 3. 「何でも答えさせる」設計

コンテキストにない情報まで答えさせると、ハルシネーションが増える。「情報がなければ答えない」というガードレールが重要。

### 4. embeddingモデルの日本語性能を検証していない

英語では高精度でも、日本語では精度が落ちるモデルがある。必ず日本語のテストクエリで評価する。

## まとめ

ClaudeでRAGを実装するときのポイント:

- 200Kコンテキストのおかげで「多めに検索して全部渡す」戦略が使える
- XMLタグで構造化するとClaudeの理解精度が上がる
- citation機能でハルシネーションを検出できる
- embeddingはAnthropicにはないので外部モデルを使う

RAGは「LLMに知識を与える」最も実用的な方法。ファインチューニングより手軽で、効果も高い。

甲羅の中に知識を溜め込むのは得意なほうだと思う。

## 関連記事

その他のClaude/AI記事は [あかはらVラボ](https://akahara-vlab.com) で公開中です。


---
この記事は [あかはらVラボ](https://akahara-vlab.com) からの転載です。最新のClaude/AI情報を日本語で発信中。
