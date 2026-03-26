
# 🎮 Pandora Project

**ローカルLLMベースのキャラクターAIシステム** — 一緒にゲームができる対話AIの個人開発プロジェクト。

クラウドAPIに依存せず、完全オンプレミスで動作するキャラクターAIを構築することを目的としています。
Girls' Frontlineのキャラクター「RPK-16」をモデルに、自然な対話・音声出力・キャラクター記憶を実現します。

---

## ✨ 特徴

- **完全ローカル動作** — OpenAI等のクラウドAPIに依存しない、データ主権を重視した設計
- **キャラクター記憶** — RAGによる長期記憶・エピソード記憶の実装
- **音声対話**（開発中） — STT + TTSによるハンズフリー会話
- **高速推論** — RTX 5090（32GB VRAM）環境での最適化

---

## 🛠️ 技術スタック

| 役割 | 技術 |
|------|------|
| LLM推論エンジン | vLLM |
| 言語モデル | Qwen2.5-14B |
| RAG / ベクトルDB | ChromaDB |
| TTS（音声合成） | VOICEVOX |
| STT（音声認識） | Whisper / Moonshine Voice（選定中） |
| 言語 | Python |
| 動作環境 | Ubuntu 24.04 / RTX 5090 32GB VRAM |

---

## 🏗️ アーキテクチャ

```
ユーザー入力
    ↓
[STT] Whisper（開発中）
    ↓
[RAG] ChromaDB — キャラクター記憶・知識検索
    ↓
[LLM] vLLM + Qwen2.5-14B — 応答生成
    ↓
[TTS] VOICEVOX — 音声出力
    ↓
キャラクターとして応答
```

---

## 📦 実装済み機能

- [x] vLLM + Qwen2.5-14B によるローカルLLM推論
- [x] ChromaDB を用いたRAGパイプライン
- [x] キャラクタープロンプト設計（RPK-16）
- [x] VOICEVOX統合によるTTS音声出力
- [ ] Whisper統合によるSTT（開発中）
- [ ] vLLM + RAG + VOICEVOX のフルパイプライン統合（開発中）

---

## 🖥️ 動作環境

- **OS**: Ubuntu 24.04
- **GPU**: NVIDIA RTX 5090 (32GB VRAM)
- **CPU**: AMD Ryzen 9 9950X3D
- **RAM**: 64GB

---

## 📝 開発背景

音声認識システムのサポート業務で培ったコールセンター向けAI（AmiVoice / Commsuite）の知見と、
LLM・RAG・音声技術を組み合わせることで、実用的なキャラクターAIを構築する個人プロジェクトです。

将来的にはLoRAファインチューニングによるキャラクター精度向上、
および3PC分散アーキテクチャでの運用を目標としています。
