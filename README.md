# GraphRAG on Databricks

MicrosoftのGraphRAGをDatabricks上で実行するためのサンプルノートブックです。

## GraphRAGとは

GraphRAGは、Microsoftが開発したナレッジグラフベースのRAG(Retrieval-Augmented Generation)手法です。従来のベクトル検索ベースのRAGでは難しかった「データセット全体に関する質問」に回答できることが特徴です。

## 従来のRAGとの違い

| 項目 | 従来のベクトルDB RAG | GraphRAG |
|------|---------------------|----------|
| データ構造 | フラットなチャンク | 階層的なグラフ構造 |
| 検索方式 | ベクトル類似度のみ | ベクトル + グラフ探索 |
| 全体要約 | 苦手 | 得意(Global Search) |
| 詳細検索 | 得意 | 得意(Local Search) |
| インデックス作成コスト | 低い | 高い(LLM呼び出し多数) |

## 前提条件

- Databricks Runtime 14.0 ML以上
- **クラスター**(サーバレスではなく)を使用
- Unity Catalogが有効なワークスペース

## Databricksでの注意事項

### LLMの選択

| モデル | インデックス作成 | クエリ |
|--------|----------------|--------|
| FMAPI Llamaモデル | ○ | × |
| FMAPI OpenAIモデル(databricks-gpt-5-2等) | ○ | ○ |
| OpenAI API / Azure OpenAI | ○ | ○ |

FMAPIのLlamaモデルはJSON mode制約によりGraphRAGのクエリと互換性がありません。FMAPIを使用する場合はOpenAIモデルを選択してください。

### FMAPI設定のポイント

```yaml
models:
  default_chat_model:
    type: openai_chat
    api_key: ${DATABRICKS_TOKEN}
    api_base: ${DATABRICKS_HOST}/serving-endpoints
    model: databricks-gpt-5-2
    model_supports_json: true
    max_tokens: 4096  # 必須: FMAPIはnullを受け付けない
```

### ストレージ

LanceDBの`rename`操作制約により、以下の構成を推奨します:

- 処理時: ローカルディスク(`/tmp`)を使用
- 永続化: Delta Tableに保存

| ストレージ | rename対応 | 永続化 | サーバレス対応 |
|-----------|-----------|-------|--------------|
| ローカルディスク(/tmp) | ○ | × | × |
| Workspaceファイル | × | ○ | ○ |
| Unity Catalogボリューム | × | ○ | ○ |

## ノートブック

| ファイル | 説明 |
|---------|------|
| `graphrag_databricks_fmapi.py` | FMAPI OpenAIモデル版 |
| `graphrag_databricks_openai.py` | OpenAI API版 |

## アーキテクチャ

```
[入力テキスト]
     ↓
[GraphRAG Index] ← FMAPI OpenAIモデル or OpenAI API
     ↓
[ローカルディスク(/tmp)]
  - entities.parquet
  - relationships.parquet
  - communities.parquet
  - community_reports.parquet
  - text_units.parquet
  - lancedb/
     ↓
[Delta Table] ← 永続化
     ↓
[GraphRAG Query] ← FMAPI OpenAIモデル or OpenAI API
     ↓
[回答]
```

## 参考リンク

- [GraphRAG GitHub](https://github.com/microsoft/graphrag)
- [GraphRAG Documentation](https://microsoft.github.io/graphrag/)
- [From Local to Global: A Graph RAG Approach to Query-Focused Summarization(論文)](https://arxiv.org/abs/2404.16130)
- [Qiita記事: MicrosoftのGraphRAGをDatabricksで動かしてみた](https://qiita.com/taka_yayoi/items/f343c4f3a3284e41a3ab)

## ライセンス

MIT
