---
title: "Mem0: AIエージェントのためのセッション間記憶インフラ"
emoji: "💾"
type: "tech"
topics: ["ai", "agent", "memory", "mem0"]
published: true
---

## 概要

Mem0はAIエージェントのための自己改善メモリレイヤーで、2026年4月に新しいメモリアルゴリズムが導入されました。従来のチャット履歴のリプレイに依存するのではなく、重要な事実を抽出・保存・検索することで、セッションをまたいだパーソナライズされたエージェント体験を実現します。

## なぜエージェントにとってメモリ層が必要なのか

現在のAIエージェントは以下の課題に直面しています。

1. **コンテキスト爆発**: チャット履歴をプロンプトに追加し続けると、トークン数が急速に増大し、コストとレイテンシが悪化します。
2. **セッション断絶**: セッションが終了すると、重要な事実（ユーザーの好み、プロジェクトの決定など）が失われます。
3. **一貫性の欠如**: ユーザーごとの隔離が不十分だと、異なるユーザーのデータが混在します。

Mem0はこれらの問題を解決し、重要な事実を一度保存し、必要に応じて検索できるようにします。

## Mem0の動作原理

Mem0は「抽出」と「検索」の2つのフェーズで動作します。

### 1. 抽出 (Writing memory)

新しいメッセージが届くと、Mem0は以下の処理を行います。

1. **コンテキスト照会**: 既存の記憶と重複を避けるために関連記憶をチェック
2. **事実抽出**: LLMが好み、決定、プランなどの耐久事実を抽出
3. ** deduplication と埋め込み**: 重複する事実を削除し、意味的検索用に埋め込みを生成
4. **エンティティ抽出**: メモリに言及する人物、場所、組織、概念を抽出し、検索時のエンティティマッチングを可能にします

典型的な変換例:

| 入力メッセージ | 保存された記憶 |
| --- | --- |
| "I prefer aisle seats" | "User prefers aisle seats" |
| "Let's use Postgres for this project" | "Project decision: use Postgres" |

### 2. 検索 (Reading memory)

`search` を呼び出すと、Mem0はクエリ against 保存された記憶をランク付けし、最も関連性の高いものをフィルタリングします。

| シグナル | 用途 |
| --- | --- |
| **意味的** | ベクトル類似性による概念質問 |
| **キーワード** | 正確な単語・フレーズ照合（名前、ID、事実検索） |
| **エンティティ** | クエリに関連するエンティティを持つ記憶のブースト |
| **時間的** | 時限意図に基づいた候補のスコアリング（いつの質問にも対応） |

## 設定と利用

### Pythonによるセットアップ

```bash
pip install redis redisvl pip install mem0ai
docker run -d --name redis-stack -p 6379:6379 -p 8001:8001 redis/redis-stack:latest
```

```python
import os
from mem0 import Memory

os.environ["OPENAI_API_KEY"] = "sk-xx"

config = {
    "vector_store": {
        "provider": "redis",
        "config": {
            "collection_name": "mem0",
            "embedding_model_dims": 1536,
            "redis_url": "redis://localhost:6379"
        }
    },
    "version": "v1.1"
}

m = Memory.from_config(config)

# メモリへの追加
messages = [
    {"role": "user", "content": "I'm planning to watch a movie tonight. Any recommendations?"},
    {"role": "assistant", "content": "How about thriller movies? They can be quite engaging."},
    {"role": "user", "content": "I'm not a big fan of thriller movies but I love sci-fi movies."}
]
m.add(messages, user_id="alice", metadata={"category": "movies"})

# メモリの検索
query = "What movies does Alice like?"
memories = m.search(query)
print(memories)
```

### TypeScript/OSSによるセットアップ

```typescript
import { Memory } from 'mem0ai/oss';

const config = {
  vector_store: {
    provider: "redis",
    config: {
      collectionName: "mem0",
      embeddingModelDims: 1536,
      redisUrl: "redis://localhost:6379"
    }
  },
  version: "v1.1"
};

const m = Memory.from_config(config);
```

## ベンチマークと改善点 (2026年4月)

| メトリクス | 旧アルゴリズム | 新アルゴリズム | 改善 |
| --- | --- | --- | --- |
| 単一パス抽出 | 複数LLMコール | 1コール (ADD-only) | コスト50%削減 |
| 事実の永続性 | UPDATE/DELETEをサポート | ADD-only (累積) | 書き込み高速化 |
| エンティティリンク | なし | エンティティ抽出・リンク | 検索精度向上 |
| 多信号検索 | 意味的のみ | 意味的・キーワード・エンティティの並列融合 | リコール向上 |

新アルゴリズムの主な変更点:

- **single-pass ADD-only extraction**: 1回のLLMコールで事実を抽出し、上書きまたは削除は行いません。記憶は累積されます。
- **Agent-generated facts are first-class**: エージェントが確認したアクション情報も同等の重みで保存されます。
- **Entity linking**: エンティティが抽出、埋め込み、および記憶間のリンク処理され、検索時にブーストされます。
- **Multi-signal retrieval**: 意味的、BM25キーワード、エンティティマッチングが並列でスコアリングされ、融合されます。

## 利用事例

- **パーソナライズされた推薦**: ユーザーの好み（映画、本、製品）を記憶し、以降のセッションで推薦に活用
- **プロジェクト継続性**: プロジェクトの決定、アーキテクチャの選択を記憶し、後のセッションで参照
- **多ユーザー隔離**: `user_id`、`agent_id`、およびメタデータで検索をスコープし、ユーザー間でデータが混在しないようにします。

## コスト最適化のヒント

- `infer=False` を使用して原文 그대로を保存し、再検索時には意味的検索のみを有効にします。
- メタデータでフィルタリング（workspace、feature area、tenant、データソースなど）を行い、関連しない記憶からの検索を削減します。
- 重要な事実のみを `add` で保存し、完全なチャット履歴は保存しないでください。

## 一次情報

- [Redis Blog: Smarter memory management for AI agents with Mem0 and Redis](https://redis.io/blog/smarter-memory-management-for-ai-agents-with-mem0-and-redis/)
- [Mem0 Docs: How it works](https://docs.mem0.ai/core-concepts/how-it-works)
- [Mem0 GitHubリポジトリ](https://github.com/mem0ai/mem0/)
- [Mem0 Docs: Vector databases - Redis](https://docs.mem0.ai/components/vectordbs/dbs/redis)