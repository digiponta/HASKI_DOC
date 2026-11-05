# SKI v0.5 追補 — 対話データセットと学習データセット
**Independent Editor’s Draft — 2026-11-05**

## 1. 目的と位置付け
SKI通信が扱うアプリケーションデータセットを、**Dialogue Dataset（対話データセット）**と**Learning Dataset（学習データセット）**の2種類に分類する。SKI v0.5本編に定義されたDataset AとDataset Bは往復通信における**送信側・返信側の役割**であり、この2種類のデータセット分類とは直交する。

自然言語と構造化命題を交換し、Embedding・ローカルベクトルは原則転送せず受信側で再生成する。v0.4のGlobal Semantic ID、Local Assertion ID、Replica Registry、来歴・イベント同期を継承する。

本追補は仕様草案であり、既存のSEMOS_S2N実装がここに定義する型をすでにサポートするとは主張しない。

## 2. データセット種別
| 種別 | 識別値 | 主目的 | 最小コンテンツ |
|---|---|---|---|
| 対話データセット | `DIALOGUE` | 問い合わせ・回答・複数ターン対話 | 会話ID、発話ID、話者、発話本文、言語、相関情報 |
| 学習データセット | `LEARNING` | 質問と回答の組の知識共有・学習候補 | 学習例ID、質問、回答、言語、来歴・評価・学習承認状態 |

**MUST:** データセットごとに `dataset_type` を宣言し、`DIALOGUE` と `LEARNING` のいずれかを設定する。複合用途は混在レコードとしてあいまいに扱わず、用途別データセットへ明示的に分離する。**MUST NOT:** 対話のやり取りが存在するだけでそのまま学習用として自動承認しない。

## 3. 共通交換属性
双方のデータセットは既存の `ski-natural-language-v0.5` のメッセージに対応付けられる構造を持つ。ここでの `dataset_type` は**提案するデータセットレベルのメタデータ**であり、現行のメッセージ形式そのものを無断変更するものではない。

推奨共通属性：`dataset_id`、`dataset_type`、`ski_version`、`source_llm`、`target_llm`、`messages[]`、`dataset_digest`、`created_at`、`origin`、`access_policy`。通信層の受領確認（ACK）を返信内容と分離する。既存の素のメッセージ配列との相互運用を維持するには、データセット種別をセッション設定等で別途通知するか、交渉済みのデータセットメタデータ方式を使う。

識別情報は各転送元・転送先のローカルID、メッセージID、グローバルな系譜IDを区別する。UUID等のグローバルIDは同一の意味内容を保証しない。

## 4. Dialogue Dataset
対話データセットは一連の会話を表す。複数ターンに対応し、`conversation_id`、`turn_id`、`utterance_id`、`speaker`（USER / ASSISTANT / SYSTEM / TOOL等、ポリシーに従う）、`language`、`text`、`in_reply_to`、`context_refs[]`、`source_llm`、`target_llm`を保持する。

- **MUST:** 同一会話内の発話IDを一意にし、回答は対象の発話IDへ `in_reply_to` で対応付ける。
- **SHOULD:** 質問・回答・確認・補足等の `speech_act` を保持する。
- **MUST:** 個々の応答に別の `message_id` を付与し、`derivation.response_to_message_id` で応答対象を関連付ける。
- **MUST NOT:** 対話データセットに含まれる内容を検証や同意なく学習対象に昇格しない。
- **MAY:** 同じ質問に複数回答、未回答、確認質問を記録する。返信が一対一であると仮定しない。

## 5. Learning Dataset
学習データセットは質問と回答の組（Q/A）を基本単位とする。各学習例は `example_id`、`question`、`answer` を**必須**とし、必要に応じて `question_sd_id`、`answer_sd_id`、`globalSemanticId`、`context`、`conditions`、`provenance`、`assessment`、`lifecycle`、`approval`、`revision`、`source_dialogue_refs[]` を保持する。

- **MUST:** 質問と回答を対応付け、片方だけのレコードを有効なQ/A学習例と扱わない。
- **MUST:** 回答には未検証・確認済み・争点あり等の評価状態を保持し、`APPROVED` を客観的真理の保証と解釈しない。
- **MUST:** 転送・受信・検証・Semantic Memoryへの登録・内部学習を異なる状態として記録する。
- **MUST NOT:** 送信者が `APPROVED` としただけで受信者が自動承認または `/sleep` を起動しない。
- **SHOULD:** 由来となる対話データセットへの参照・出典・ライセンス・個人情報扱いを記録する。
- **MAY:** ひとつの質問に複数の回答候補を持つ場合、各候補を別 `example_id` として、その根拠と評価を区別する。

## 6. 対話から学習への変換
`DIALOGUE` 内の質問と回答を `LEARNING` のQ/A例へ変換するとき、`source_dialogue_refs[]` で発話ID・回答IDを特定する。人間またはポリシーによる確認、アクセス権限、内容の整合性・出典をチェックし、新たな学習例IDを採番する。元の対話データを改変せず来歴を保持する。

状態例：`CANDIDATE → REVIEWED → APPROVED → REGISTERED → INTERNALIZATION_PENDING → INTERNALIZED`。これはワークフロー例であり、受信・承認・学習が同時発生することを意味しない。撤回時は学習対象から外し派生レコードへ通知するが、モデル重みからの完全削除は保証しない。

## 7. JSON例（仕様説明用；公式Schema未確定）
### 7.1 対話データセット
```json
{
  "ski_version": "0.5",
  "dataset_id": "dialogue-001",
  "dataset_type": "DIALOGUE",
  "source_llm": "LLM-A",
  "target_llm": "LLM-B",
  "conversation_id": "conversation-001",
  "turns": [
    {"turn_id": "t1", "utterance_id": "u1", "speaker": "USER",
     "language": "ja", "text": "GPUは何に適していますか？"},
    {"turn_id": "t2", "utterance_id": "u2", "speaker": "ASSISTANT",
     "language": "ja", "text": "GPUは並列計算に適しています。",
     "in_reply_to": "u1"}
  ]
}
```
### 7.2 学習データセット
```json
{
  "ski_version": "0.5",
  "dataset_id": "learning-001",
  "dataset_type": "LEARNING",
  "source_llm": "LLM-A",
  "target_llm": "LLM-B",
  "examples": [
    {
      "example_id": "qa-001",
      "question": {"language": "ja", "text": "GPUは何に適していますか？"},
      "answer": {"language": "ja", "text": "GPUは並列計算に適しています。"},
      "assessment": "UNVERIFIED",
      "approval": "PENDING",
      "source_dialogue_refs": ["u1", "u2"]
    }
  ]
}
```
上記は**概念データモデルの例**であり、従来の `ski_nl_exchange.py` にそのまま渡す `ski-natural-language-v0.5` メッセージ配列ではない。標準ワイヤー形式の定義にはメッセージ形式とのマッピングとSchemaの固定が必要。

## 8. Dataset A/Bとの関係
Dataset A/Bは**送信・応答の向き**であり、DIALOGUE/LEARNINGは**用途**である。対話におけるA→BとB→Aの戻りデータはともにDialogue Datasetでよい。Learning Datasetは一方向複製または受信側からの訂正・評価の返送が可能。受領ACKはどちらの場合も独立した制御情報とし、内容生成の成功や学習完了を示さない。

## 9. v0.4 Replica Registry
対話データには `conversation_id`、`utterance_id`、`message_id` により対応付け、学習データには `example_id`、`question_sd_id`、`answer_sd_id` とグローバル系譜ID・各VMローカルIDを保持する。複数転送先で生成されたコピー/派生データの相互識別を継承する。受信ノードの確認済みACKがない複製を `CONFIRMED` にしない。

## 10. 実装・適合試験
1. DIALOGUE / LEARNINGの種別判定、必須フィールドとスキーマ検証。
2. 質問と回答の対応関係、欠落回答、複数回答、複数ターン、再送・順不同を検証。
3. 対話からの学習候補化時の権限・承認・来歴と隔離を検証。
4. Dataset A/BとACK、相関ID、SHA-256およびReplica Registryの既存要件を維持。
5. 受信後のSemantic Memory保存、`/sleep`、モデル変更、訂正撤回の追跡を別々に検証。
6. SEMOS_S2Nのプロトタイプを拡張し、独立した外部Consumerで適合試験する。

**Status:** SKI v0.5追補案。実装・独立適合試験・公式JSON Schemaは未完成。
