# フェーズ進捗

## Phase 1：LLM連携・RAGパイプライン構築 ✅ 完了（2026年2月）

**目標**: vLLMによるLLM推論・RAGパイプラインの実装・キャラクタープロンプト設計

### 実装内容

**LLM連携（chat_rpk16.py）**
- vLLM（`localhost:8000`）への接続（OpenAI互換API経由）
- systemプロンプトによるRPK-16キャラクター設定
- `messages`リストによる会話履歴の保持
- while-loopによる連続対話
- try-exceptによるエラーハンドリング
- 応答速度計測（`time.time()`）

**RAG統合チャット（chat_rpk16_rag_cpu.py）**
- ChromaDB + sentence-transformersによるRAGパイプライン
- 会話履歴の保持（`conversation_history`）
- タイムスタンプ付き会話ログのファイル保存（`~/rpk16_logs/`）
- RAGデータ：RPK-16セリフ69件収録（2026年2月時点）

**キャラクタープロンプトの改良（3段階）**
- 初期実装 → 語尾混入（他キャラのデータがRAGから混入）問題を修正
- RAGデータ品質の改善で自然なキャラクター表現を実現

### 技術的な学び
- vLLMのOpenAI互換API構造（`base_url="http://localhost:8000/v1"`）
- Pythonの基礎：辞書・リスト・while・try-except・f-string
- インデントがプログラムの構造そのものであること
- `messages`リストの設計：systemロール（意味記憶）とuserロール（会話）の分離

---

## Phase 2：VOICEVOX統合 ✅ 完了（2026年2月）

**目標**: LLMの応答テキストをVOICEVOX経由で音声出力する

### 実装内容

**VOICEVOX連携（chat_rpk16_v2.py）**
- VOICEVOX Engine（Docker / `localhost:50021`）との接続
- 2ステップAPIの実装
  1. `POST /audio_query` — テキスト → 音声パラメータ
  2. `POST /synthesis` — 音声パラメータ → wavデータ
- `sounddevice`による音声再生
- `speak(text)`関数に処理をカプセル化

**使用ボイス**: 冥鳴ひまり（スピーカーID: 14） ※仮採用

**環境構築での学び**
- venv（仮想環境）の必要性と作成方法
- `pip`（Pythonライブラリ）と`apt`（OSライブラリ）の違い
- ストレージ管理：ルートパーティション（94GB）の整理（waydroid・flatpak等削除で5GB以上回収）

### 現在の動作環境

| サービス | URL | 起動方法 |
|---------|-----|---------|
| vLLM | `localhost:8000` | 常駐 |
| VOICEVOX Engine | `localhost:50021` | `docker start voicevox` |

### ファイル構成（`~/Projects/study/`）

```
~/Projects/study/
├── venv/                     ← Python仮想環境
├── chat_rpk16.py             ← Phase 1：テキストチャット
├── chat_rpk16_v2.py          ← Phase 2：音声出力付きチャット
└── test_voicevox.py          ← VOICEVOX動作確認用

~/chat_rpk16_rag_cpu.py       ← RAG統合チャット（対話ループ＋ログ保存）
~/rpk16_logs/                 ← 会話ログ保存先
```

---

## Phase 3：Moonshine Voice統合（STT） 🔧 開発中

**目標**: マイク入力をリアルタイム音声認識してハンズフリー会話を実現

### 採用技術：Moonshine Voice（日本語Mediumモデル）

WhisperからMoonshineに切り替えた経緯は [`ARCHITECTURE.md`](ARCHITECTURE.md) のSTTセクションを参照。

**実装手順（予定）**

```bash
# Step 1: インストール
pip install moonshine-voice

# Step 2: 日本語精度の確認
python -m moonshine_voice.mic_transcriber --language ja

# Step 3: VOICEVOXとのパイプライン接続テスト
# Step 4: レイテンシ・精度の実測
# Step 5: 本格統合
```

**完成形のパイプライン**

```
マイク入力 → Moonshine Voice（DEFY/CPU）→ vLLM（DEFY/GPU）→ VOICEVOX（DEFY/CPU）→ 音声出力
```

---

## Phase 4：VLM（画面認識）統合 📋 予定

**前提条件**: 3台目PCへのRTX 5070Ti入手後

- 採用モデル候補：Qwen2-VL 7B（4bit量子化 / 5GB VRAM）
- DEFYがゲーム画面のスクリーンショットを差分検出で3台目に送信
- VLMで解析した結果を構造化JSONでDEFYに返し、RPK-16への負担を最小化

---

## Phase 5：LoRAファインチューニング 📋 予定

- ファインチューニング実行PC：3台目（RTX 5070Ti入手後）
- 必要データ量：RPK-16のセリフ・会話データ 500〜1000サンプル
- 現在のRAGDBへの会話ログ蓄積が学習データ収集を兼ねている

---

## Phase 6：Style-Bert-VITS2（Voice Cloning） 📋 予定（だいぶ先）

- あねざわさん自身がRPK-16の声真似を録音して学習
- 権利的にクリアな方法でRPK-16専用の音声モデルを作成
- 完成後にVOICEVOXと差し替え

---

## インフラ・環境構築の記録

Phase実装と並行して構築した環境のメモ。

| 対応内容 | 詳細 |
|---------|------|
| DEFYのSSHハードニング | Ed25519鍵認証・UFW・Fail2ban導入 |
| UFW設定後のDeskflow復旧 | port 24800開放で解決 |
| Ubuntu起動トラブル | RTX 5090 BlackwellはOpenカーネルモジュール必須（`nvidia-open`）|
| RTX 5090 BTO受領 | FRONTIER製、2026年1月 |
| 3台目PC組み立て | MSI Pro B850M-A WIFI + Ryzen 7 9700X + 2TB SSD（2026年2月運用開始）|
