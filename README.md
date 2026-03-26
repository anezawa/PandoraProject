# 🎮 Pandora Project

**ローカルLLMベースのキャラクターAIシステム** — クラウドAPIに依存しない、完全オンプレミスで動作するAI VTuberシステムの個人開発プロジェクト。

Girls' Frontlineのキャラクター「RPK-16」をモデルに、自然な対話・音声出力・キャラクター記憶を実現する。設計思想の詳細は [`docs/PHILOSOPHY.md`](docs/PHILOSOPHY.md) を参照。

---

## ✨ 特徴

- **完全ローカル動作** — OpenAI等のクラウドAPIに依存しない、データ主権を重視した設計
- **キャラクター記憶** — RAGによる長期記憶・エピソード記憶の実装
- **リアルタイム音声対話**（Phase 3開発中）— Moonshine VoiceによるSTT + VOICEVOXによるTTS
- **高速推論** — RTX 5090（32GB VRAM）環境でのvLLM推論

---

## 🛠️ 技術スタック

| 役割 | 技術 | 選定理由 |
|------|------|----------|
| LLM推論エンジン | vLLM | PagedAttentionによる高スループット、複数コンポーネントからの同時リクエストに強い |
| 言語モデル | Qwen2.5-14B (GPTQ量子化) | 14Bクラスで日本語性能が高く、RTX 5090の32GB VRAMに収まる |
| RAG / ベクトルDB | ChromaDB | 軽量・ローカル動作・Python親和性が高い |
| TTS（音声合成） | VOICEVOX | 無料・ローカル動作・Docker対応。将来的にStyle-Bert-VITS2へ移行予定 |
| STT（音声認識） | Moonshine Voice（日本語Mediumモデル） | CPU動作のためRTX 5090をLLM推論に全振りできる。VAD内蔵でパイプライン実装コストが低い |
| 言語 | Python | — |
| 動作環境 | Ubuntu 24.04 / RTX 5090 32GB VRAM | — |

---

## 🖥️ ハードウェア構成

| PC | スペック | 役割 |
|----|----------|------|
| **DEFY** | RTX 5090 (32GB VRAM) / Ryzen 9 9950X3D / 64GB RAM / Ubuntu 24.04 | LLM推論・VOICEVOX・Moonshine Voice（CPU動作）・ゲーム実行 |
| **FUCHAN** | RTX 3050 / Ryzen 5 5600 / 32GB RAM / Windows | OBS配信・音声再生・VTube Studio |
| **3台目**（構築中） | Ryzen 7 9700X / 96GB RAM / Proxmox | 監視・オーケストレーション・管理。GPU（RTX 5070 Ti予定）入手次第、VLM（画面認識）・LoRAファインチューニングに活用 |

---

## 🏗️ システムアーキテクチャ

```
マイク入力（FUCHAN）
    ↓ 音声データ送信
[STT] Moonshine Voice - Japanese Medium（DEFY / CPU動作）
    ↓ テキスト
[RAG] ChromaDB — キャラクター記憶・知識検索（DEFY）
    ↓ コンテキスト付きプロンプト
[LLM] vLLM + Qwen2.5-14B — 応答生成（DEFY / RTX 5090）
    ↓ 応答テキスト
[TTS] VOICEVOX（DEFY / CPU動作）
    ↓ 音声データ送信
音声再生（FUCHAN）
```

詳細な設計判断・リソース試算は [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) を参照。

---

## 📦 実装進捗

| Phase | 内容 | 状態 |
|-------|------|------|
| Phase 1 | vLLM + Qwen2.5-14B 推論環境構築・RAGパイプライン実装・キャラクタープロンプト設計 | ✅ 完了 |
| Phase 2 | VOICEVOX統合（LLM応答の音声出力） | ✅ 完了 |
| Phase 3 | Moonshine Voice統合によるSTT・ハンズフリー会話 | 🔧 開発中 |
| Phase 4 | VLM（画面認識）統合 — 3台目GPU入手後 | 📋 予定 |
| Phase 5 | LoRAファインチューニングによるキャラクター精度向上 | 📋 予定 |
| Phase 6 | Style-Bert-VITS2によるVoice Cloning | 📋 予定（だいぶ先） |

各フェーズの詳細は [`docs/PROGRESS.md`](docs/PROGRESS.md) を参照。

---

## 🚀 クイックスタート（Phase 2時点の環境）

### 前提条件

- DEFY（Ubuntu 24.04 / RTX 5090）上で以下が稼働中であること
  - vLLM: `localhost:8000`（モデル: Qwen2.5-14B-GPTQ）
  - VOICEVOX Engine（Docker）: `localhost:50021`

### セットアップ

```bash
# リポジトリクローン
git clone <this-repo>
cd pandora-project

# 仮想環境の作成・有効化
python3 -m venv venv
source venv/bin/activate

# 依存ライブラリのインストール
pip install openai requests sounddevice soundfile chromadb sentence-transformers
sudo apt install libportaudio2  # sounddeviceの依存
```

### 実行

```bash
# テキストチャット（Phase 1）
python3 chat_rpk16.py

# 音声出力付きチャット（Phase 2）
python3 chat_rpk16_v2.py

# RAG統合チャット（会話ログ保存あり）
python3 chat_rpk16_rag_cpu.py
```

---

## 📝 開発背景

音声認識システムのサポート業務（AmiVoice / Commsuite）で培ったコールセンター向けAIの知見と、LLM・RAG・音声技術を組み合わせて実用的なキャラクターAIを構築する個人プロジェクト。

ローカルLLMにこだわる理由は技術的な選択ではなく設計思想的な必然から来ており、その経緯は [`docs/PHILOSOPHY.md`](docs/PHILOSOPHY.md) に詳しくまとめている。
