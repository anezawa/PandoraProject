# 階層型記憶アーキテクチャ

> **設計哲学**: キャラクター一貫性の鍵は「動的検索の精度」だけでなく、  
> 「静的に常にプロンプトに存在する意味記憶層の設計」にある。  
> モデルサイズではなく、コンテキストに何を入れるかが品質を決定する。

---

## 概要

現行の単一コレクション方式（個別セリフの類似検索）が抱える「ぶつ切りの記憶」問題を解決するため、  
**CoALA認知アーキテクチャ**に基づく3層記憶分離を採用する。

```
人間の記憶モデル → LLMシステムへのマッピング

意味記憶  (Semantic)   → rpk16_semantic   : RPK-16とは何者か（固定注入）
エピソード記憶 (Episodic) → rpk16_episodes  : 何が起きたか（動的検索）
手続き記憶 (Procedural)  → rpk16_dialogues : どう話すか（Few-shot検索）
```

---

## アーキテクチャ詳細

### Layer 1 — 意味記憶 `rpk16_semantic`

RPK-16の核心的性格・哲学・人間関係を **50〜100件の事実文** で格納。  
毎回のプロンプトに**固定注入**される層。トークン予算 約200〜300トークン。

```python
# 格納例
{
    "id": "personality_core_001",
    "document": "RPK-16は知的で計算高く、陽気な外見の下に深い策略を持つ。"
                "哲学的な話題に強い関心を示し、文学的な比喩を好む。",
    "metadata": {"category": "personality", "importance": 5}
}
{
    "id": "philosophy_identity",
    "document": "RPK-16はアイデンティティの可変性を信じている。"
                "忠誠心は固定された自己を前提とするが、自己が可変であれば"
                "忠誠は毎瞬間の選択の再構築であると考える。",
    "metadata": {"category": "philosophy", "importance": 5}
}
```

**検索なし・常時存在**が重要。RPK-16の哲学的基盤は検索に頼るべきではない。

---

### Layer 2 — エピソード記憶 `rpk16_episodes`

ストーリーシーンを **シーンチャンク（200〜400トークン）** と **アーク要約（100〜200トークン）** の2階層で格納。  
会話内容に応じて動的検索される。

```python
# メタデータスキーマ
metadata = {
    "story_arc":           "mirror_stage",       # 鏡像論、零電荷 等
    "chapter":             "chapter_3",
    "scene_type":          "dialogue",            # dialogue|monologue|narration|battle
    "characters_present":  ["RPK-16", "AN-94", "AK-15"],
    "emotional_tone":      "playful_menacing",
    "theme":               "betrayal",            # betrayal|identity|loyalty|free_will
    "speaker":             "RPK-16",
    "timestamp_order":     45,                   # 時系列順序
    "importance":          4,                    # 1〜5
    "doc_type":            "scene"               # scene|arc_summary|character_fact
}
```

アーク要約は「何が起きたか」ではなく、  
**「RPK-16が何を考え、なぜそう動き、何を得たか」** を中心に記述する。

---

### Layer 3 — 手続き記憶 `rpk16_dialogues`

RPK-16の話し方パターン・口癖・特定トピックへの反応を格納。  
検索結果をFew-shotとして注入し、LLMに「どう話すか」を示す。

**PList + Ali:Chat形式**（PygmalionAIコミュニティ実績あり）を採用。  
- PList（性格キーワード）→ システムプロンプト末尾（Author's Note位置）  
- Ali:Chat対話例 → プロンプト冒頭  

> **Lost in the Middle効果**: LLMは入力の最初と最後に最も注意を払う。  
> この「サンドイッチ配置」がキャラクター一貫性を最大化する。

---

## 多段検索パイプライン

```python
class RPK16RetrievalPipeline:
    def __init__(self, chroma_client, embedding_fn):
        self.summaries = chroma_client.get_or_create_collection(
            "rpk16_summaries", embedding_function=embedding_fn)
        self.scenes = chroma_client.get_or_create_collection(
            "rpk16_scenes", embedding_function=embedding_fn)
        self.facts = chroma_client.get_or_create_collection(
            "rpk16_facts", embedding_function=embedding_fn)

    def retrieve(self, user_message, conversation_context=""):
        query = f"{conversation_context} {user_message}".strip()

        # Stage 1: アーク要約 — 「今のRPK-16がどの文脈にいるか」を確定
        arc = self.summaries.query(
            query_texts=[query], n_results=1,
            where={"doc_type": "arc_summary"})

        # Stage 2: 関連シーン — 「具体的に何が起きたか」を補強
        scenes = self.scenes.query(
            query_texts=[query], n_results=8,
            where={"$and": [
                {"characters_present": {"$contains": "RPK-16"}},
                {"importance": {"$gte": 2}}
            ]})

        # Stage 3: 対話例 — 「どう話すか」をFew-shotで示す
        dialogues = self.scenes.query(
            query_texts=[user_message], n_results=3,
            where={"$and": [
                {"speaker": "RPK-16"},
                {"scene_type": "dialogue"}
            ]})

        return self._assemble_context(arc, scenes, dialogues)
```

---

## Contextual Retrieval（埋め込み精度強化）

チャンクを埋め込む**前**にメタデータを文脈として付加する（Anthropic提唱手法）。

```python
def enrich_before_embedding(chunk_text, metadata):
    """埋め込み精度向上のため、チャンクに文脈を付加して埋め込む"""
    prefix = (
        f"[ドールズフロントライン - {metadata['story_arc']}] "
        f"[登場: {', '.join(metadata['characters_present'])}] "
        f"[テーマ: {metadata['theme']}] "
    )
    enriched = prefix + chunk_text
    return enriched  # 埋め込みにはenrichedを使用、プロンプト挿入はchunk_textのみ
```

「面白いわね」という短いセリフが、  
「鏡像論のRPK-16がAN-94に対して裏切りのテーマで発した」という文脈を持つベクトルになる。

---

## トークン予算設計（`max-model-len 4096`）

> 研究知見: コンテキスト利用率 **60〜70%** が最適（Llama3系検証より）。  
> 4096トークンすべてを使い切る必要はない。

| 領域 | トークン数 | 内容 |
|---|---|---|
| システムプロンプト | 300 | キャラクター指示・行動規則 |
| 性格プロファイル（固定） | 200 | 意味記憶から核心的事実 |
| アーク要約（動的） | 150 | 関連ストーリー要約 1件 |
| シーン文脈（動的） | 400 | 関連シーン 1〜2件 |
| 対話例（動的） | 250 | Few-shot例 2件 |
| 会話履歴 | 400 | 直近 2〜3ターン + 古い履歴の要約 |
| ユーザー入力 | 100 | 現在の発言 |
| 生成予約 | 512 | 応答生成用 |
| バッファ | 約1,700 | 安全マージン・動的拡張用 |

**動的予算再配分の考え方**:
- 挨拶など単純な入力 → 検索を最小化、応答予算を増加
- 哲学的な議論 → 検索を最大化、応答を絞る
- 会話履歴が長くなるほど → ローリング要約（5〜6ターンごと）で圧縮

---

## 実装ロードマップ

### Phase 1 — 即時実装（コレクション分割）
- [ ] 単一ChromaDBコレクションを3コレクションに分割
  - `rpk16_semantic`（性格事実 50〜100件）
  - `rpk16_episodes`（シーンチャンク + アーク要約）
  - `rpk16_dialogues`（対話例、既存コレクションをメタデータ拡張）
- [ ] ストーリーデータを「アーク → 章 → シーン → 対話」階層に手動整理
- [ ] 各シーンに `story_arc` / `theme` / `emotional_tone` / `characters_present` のメタデータ付与

### Phase 2 — 1〜2週間（パイプライン実装）
- [ ] 各ストーリーアーク要約をローカルモデルで生成
  - 観点: 「RPK-16がどう考え、なぜ動き、何を学んだか」
- [ ] Contextual Retrieval 導入（埋め込み前の文脈付加）
- [ ] 多段検索パイプライン実装（Stage 1→2→3）

### Phase 3 — 発展（精度・長期記憶強化）
- [ ] 会話履歴のローリング要約機能
- [ ] トピック検出による動的メタデータフィルタリング
- [ ] Cross-encoderによるリランキング
- [ ] NetworkXでキャラクター関係グラフを構築（GraphRAGコンセプトの軽量実装）

---

## 先行研究・参考

| プロジェクト | 知見 |
|---|---|
| **ChatHaruhi** (2023) | 性格記述より**対話例のFew-shot検索**が最も効果的 |
| **MOOM** (2025) | 7Bモデルが記憶管理の改善だけで大型モデルを凌駕。忘却メカニズム導入 |
| **RAPTOR** (ICLR 2024) | 意味的ツリー構造RAGでQuality +20%（物語データとの親和性高） |
| **SCORE** (2025) | ストーリー一貫性とコンテキスト検索の強化 |
| **MemGPT/Letta** | ローリング要約による階層的記憶管理 |
| **GraphRAG** (Microsoft) | 散在する哲学的発言を200トークン要約に凝縮 |
| **SillyTavern Lorebook** | キーワードトリガー型注入（意味検索の補完） |

---

## 関連ドキュメント

- [システム全体アーキテクチャ](./architecture.md)
- [RAGパイプライン実装](../src/rag/)
- [RPK-16キャラクター設定](./character_profile.md)
