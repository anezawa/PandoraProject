# アーキテクチャ詳細

## 1. コンポーネント設計

### 1.1 LLM推論：vLLM + Qwen2.5-14B

**vLLMを選んだ理由**

Pandoraは将来的にSTT・VLM・RAGなど複数コンポーネントが同時にLLMへリクエストを投げる構成になる。vLLMの`PagedAttention`と`連続バッチ処理`はこのユースケースに最適で、単一リクエスト時の体感速度はOllamaと大差ないが、複数同時リクエスト時のスループットで圧倒的に有利。

- プロトタイプ段階でモデルを試すときはOllamaを使い、本番構成ではvLLMに切り替える方針。
- 将来的にモデルが固まった段階でTensorRT-LLMへの移行も検討（コンパイルコストと引き換えに最速化）。

**モデル選定（Qwen2.5-14B GPTQ量子化）**

```
14B FP16         : 28GB VRAM（RTX 5090の32GBをほぼ使い切る）
14B Q8           : 14GB VRAM（品質劣化ほぼなし）
14B Q4 (GPTQ)    : 7〜8GB VRAM（実用範囲内）← 現在採用
```

Q4量子化でVRAMを8GB程度に抑え、ゲーム実行やVOICEVOXなど他コンポーネントとVRAMを共有できる余地を持たせている。

### 1.2 RAG：ChromaDB

**階層型RAG設計（実装中）**

| 記憶層 | 内容 | 実装方法 |
|--------|------|----------|
| 意味記憶 | RPK-16のキャラクター設定・世界観知識（固定） | systemプロンプト |
| エピソード記憶 | 過去の会話の要約 | ChromaDB |
| 手続き記憶 | キャラクターとして「やること・やらないこと」 | systemプロンプト |
| 作業記憶 | 直近の会話履歴（揮発性） | `messages`リスト |

現時点ではRPK-16のセリフデータ（2026年2月時点で69件収録）をベクトル化してChromaDBに格納済み。

**長期的な課題：会話履歴のトークン制限**

現在の実装では長い会話でトークン上限（4096）に達する。Phase 3でローリング要約（古い会話を圧縮して要約し記憶層に落とす）を実装予定。

### 1.3 STT：Moonshine Voice（日本語Mediumモデル）

**Whisperから切り替えた理由**

| 課題 | Whisper | Moonshine Voice |
|------|---------|-----------------|
| ライブ処理レイテンシ（Linux） | 16,919ms（Large v3） | **269ms**（Medium） |
| 入力ウィンドウ | 30秒固定（ゼロパディング必須） | 可変長 |
| ストリーミング | バッチ処理前提（非対応） | ネイティブ対応 |
| VAD | 別途実装必要 | **Silero VAD内蔵** |
| GPU依存 | あり | **CPU動作**（GPU不要） |
| 日本語精度 | 多言語共有モデル（精度低め） | 日本語専用モデル |

特に重要なのはCPU動作という点で、FUCHANのRTX 3050でSTTを動かすと配信（OBS・NVENCエンコード）のGPUリソースと競合するリスクがある。MoonshineをDEFYのCPUで動かすことでRTX 5090をLLM推論に全振りできる。

**DEFYでのリソース試算**

```
Moonshine Medium（CPU動作）: 2コア / ~1GB RAM / VRAM 0
Local LLM（vLLM）          : 最小CPUコア / VRAM 最大24GB
VOICEVOX（CPU動作）        : 1〜2コア / ~500MB RAM / VRAM 0
ゲーム実行                  : 8〜12GB VRAM
─────────────────────────────────────────────────
VRAM合計（目安）            : 16〜20GB / 32GB → 余裕あり
RAM合計（目安）             : ~6GB / 64GB → 余裕あり
```

**STTパイプライン（Pandoraへの統合イメージ）**

```python
from moonshine_voice import MicrophoneTranscriber

def on_speech_committed(text: str):
    # VAD検出 → STT → テキスト確定
    response = llm.generate(text)   # vLLM on DEFY
    voicevox.speak(response)         # VOICEVOX on DEFY

transcriber = MicrophoneTranscriber(
    "model/japanese-medium",
    {"onTranscriptionCommitted": on_speech_committed}
)
transcriber.start()
```

### 1.4 TTS：VOICEVOX

VOICEVOX EngineをDockerコンテナとしてDEFY上で稼働。CPU動作のためVRAMを消費しない。

**2ステップAPIの仕組み**

```
テキスト → [audio_query: localhost:50021] → 音声パラメータ
         → [synthesis: localhost:50021]  → wavデータ → 再生
```

現在は`冥鳴ひまり`（スピーカーID: 14）を仮採用。将来的にStyle-Bert-VITS2でRPK-16専用の音声モデルに差し替える予定（あねざわさん自身がRPK-16の声真似を録音して学習）。

---

## 2. マルチPC構成と通信設計

### 2.1 現在の構成（Phase 3時点）

```
FUCHAN（Windows）                    DEFY（Ubuntu）
┌─────────────────┐                 ┌───────────────────────────┐
│ OBS（配信）      │◄────音声再生────│ VOICEVOX Engine           │
│ VTube Studio    │                 │ Moonshine Voice（CPU）     │
│ マイク入力       │────音声送信────►│ vLLM + Qwen2.5-14B        │
└─────────────────┘                 │ ChromaDB（RAG）           │
                                    └───────────────────────────┘
```

### 2.2 将来構成（3台目GPU入手後）

```
FUCHAN（Windows）     DEFY（Ubuntu）          3台目（Proxmox）
┌────────────┐       ┌─────────────────┐    ┌──────────────────┐
│ OBS         │◄──── │ vLLM + LLM      │    │ Prometheus       │
│ VTube Studio│       │ VOICEVOX        │    │ Grafana          │
│ マイク入力  │────► │ Moonshine Voice │    │ ログ収集          │
└────────────┘       └────────┬────────┘    └──────────────────┘
                               │ 画面送信
                    ┌──────────▼────────┐
                    │ RTX 5070 Ti       │
                    │ VLM（Qwen2-VL 7B）│
                    │ LoRAファインチューニング │
                    └───────────────────┘
```

**VLM（画面認識）の設計**

DEFYがゲーム画面のスクリーンショットを差分検出で必要なときのみ3台目に送信し、VLMで解析した結果を構造化JSONでDEFYに返す。RPK-16への負担を最小化するための設計。

```json
{"hp": 30, "enemies": 3, "status": "danger"}
```

---

## 3. VRAM管理戦略

RTX 5090（32GB）上で複数コンポーネントが同時に動く構成のため、VRAM配分を明示的に管理する。

| コンポーネント | VRAM消費（目安） | 動作PC |
|--------------|----------------|--------|
| LLM 14B Q4 | ~8GB | DEFY |
| ゲーム（AAA） | 8〜12GB | DEFY |
| VLM 7B Q4（将来） | ~5GB | 3台目 |
| Moonshine Voice | 0（CPU動作） | DEFY |
| VOICEVOX | 0（CPU動作） | DEFY |
| **合計（DEFY）** | **16〜20GB / 32GB** | — |

将来的にVRAMが逼迫する場合はLLMをExLlamaV2（EXL2フォーマット）でさらに量子化してVRAMを節約する。

---

## 4. STT選定の補足：Moonshine vs Voxtral Realtime

Phase 3開発時点での比較評価メモ。

| 比較軸 | Moonshine Voice | Voxtral Realtime 4B |
|--------|-----------------|---------------------|
| 日本語モデル | 専用トレーニング済み | 13言語混合 |
| レイテンシ（Linux） | ~269ms（CPU） | 設定可能（80ms〜2.4s） |
| VRAM消費 | **0** | 16GB以上 |
| DEFYとのVRAM競合 | **なし** | LLMと競合 |
| VAD | **内蔵** | 要別途 |
| ライセンス（日本語） | Community License | Apache 2.0 |

Moonshineを第一候補とした決め手は「VRAM 0」という点。RTX 5090 32GBをLLMに全振りできるアドバンテージは他の要素を上回る。Voxtral Realtimeは精度検証用に並行評価を続ける。
