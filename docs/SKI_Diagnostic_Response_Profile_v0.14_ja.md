# SKI Diagnostic Response Profile v0.14（編集者草案）

**位置付け：** 2026年11月11日付の実験的相互運用プロファイル。既存のSKI v0.5自然言語Dataset Bに追加する任意のメタデータであり、SKI v0.5の正式な共通ワイヤ形式を変更するものではない。標準化団体の承認を受けた仕様でもない。

## 1. 目的と現状

LLM_CHAからLLM_TMPへの問い合わせは、SKI Dataset A、ACK、Dataset Bを通じて対応関係を維持する。LLM_TMPのRelationGraphによる探索結果は、実験的Evidence Responseを経由してLLM_SKI_BRIDGEがDataset Bとして構築する。従来はDataset Bの自然言語テキストと`assessments.evidence_status`に探索状態を保持し、LLM_CHAでは受信後に独立したTruth Boundary診断を生成していた。

v0.14では**診断情報そのものをDataset Bの構造化メタデータとして伝送**する。LLM_CHA側ではその内容を検査した後、独立したTruth Boundaryを適用する。診断結果を受け取っただけで事実として認定することはない。

## 2. データ形式と互換性

Dataset A、Dataset B、ACK、SKI v0.5の`format`、`message_id`、`source_llm`、`target_llm`、`source_sd_id`、`parent_sd_id`、`text`、`text_sha256`、`derivation`は従来どおり維持する。新しいDataset Cや通信方式は定義しない。

Dataset Bメッセージに、任意フィールド`diagnostic`を追加する。既存のv0.13受信処理は`diagnostic`なしのDataset Bを引き続き受け付ける。v0.14専用の検証処理はそのフィールドを必須とする。

### Dataset Bの`diagnostic`例

```json
{
  "profile": "ski-diagnostic-v0.14",
  "response_to_message_id": "<Dataset A message_id>",
  "evidence_request_id": "<Evidence Request message_id>",
  "evidence_response_id": "<Evidence Response message_id>",
  "source_sd": "CLAIM",
  "target_sd": "EVIDENCE",
  "diagnostic_status": "FOUND",
  "evidence_path_hops": 1,
  "truth_state": "UNVERIFIED",
  "approval_state": "UNAUTHENTICATED",
  "verified_fact": false,
  "human_approved": false,
  "allow_model_context": false,
  "allow_answer_from_evidence": false,
  "allow_learning": false,
  "allow_truth_promotion": false,
  "display_message": "承認済み関係の経路を検出。事実の真偽は未検証です。",
  "diagnostic_sha256": "<SHA-256 hex>"
}
```

`diagnostic_status`の許容値は`FOUND`、`NOT_FOUND`、`UNKNOWN_NODE`、`LIMIT_REACHED`、`DEPTH_LIMIT_REACHED`とする。`NOT_FOUND`は関係の不存在を証明するものではない。`evidence_path_hops`は`FOUND`の場合のみ非負整数（上限5）、それ以外は`null`とする。

`diagnostic_sha256`は、このフィールド自体を除いた診断オブジェクトを、UTF-8、キー順ソート、空白なしのJSONとして直列化したSHA-256とする。これは**整合性確認用であり、送信元認証、署名、利用者承認の代替にはならない**。

診断には未検証のEvidence本文、出典文章、探索経路のエッジ本文、ベクトルやテンソルを格納しない。探索経路は別の実験的Evidence Responseで構造検査される。

## 3. 受信側の検証規則

1. Dataset A、Dataset B、ACK、およびDataset A/Bの方向・応答対応関係を検査する。
2. 実験的Evidence Request/ResponseのID、SD ID、探索制限、状態、経路の構造を検査する。
3. Dataset Bの`diagnostic`を元のデータから再構成し、ID、状態、ホップ数、表示用メッセージ、ハッシュおよび安全フラグを完全一致で確認する。
4. 安全フラグはJSONの真偽値型で検査し、`0`を`false`の代替として認めない。ホップ数にも整数型検査を行う。
5. 不一致の場合はFail-Closedとし、受信した根拠を自動的にモデル入力、回答、学習または真偽状態へ昇格させない。

## 4. Trust Boundary

Semantic関係の`APPROVED`は、そのローカル関係記録が持つ状態を示すのみである。署名済みの人手承認、根拠の真正性、事実の正確性は保証しない。このプロファイルでは`truth_state=UNVERIFIED`、`approval_state=UNAUTHENTICATED`、`verified_fact=false`、`human_approved=false`、すべての`allow_*`を`false`に固定する。

真正な人手承認や検証済み事実を転送する場合には、別途認証・監査・承認の仕様とワークフローを定義する必要がある。

## 5. 参照実装と評価範囲

- `digiponta/LLM_SKI_BRIDGE`：`feature/ski-diagnostic-v0.14`。 `ski/diagnostic_v014.py`で構築・照合し、`adapters/llm_cha_client_v05.py`でDataset Bに格納、`adapters/cha_truth_boundary_v07.py`で検証する。
- `digiponta/LLM_CHA`：`feature/ski-diagnostic-v0.14`。受信したDataset Bの構造化診断を必須条件として検査した後、診断結果を表示する。
- LLM_TMPのSDRegistry/RelationGraphは現行のAPIを利用し、変更しない。

検証対象は合成Semantic RegistryとローカルPythonモジュール間の交換である。実運用Semanticデータの独立評価、ネットワーク経由の分散通信、送信元署名、根拠を利用するLLM回答生成は未対象とする。

**本書はプロジェクト内の実験的提案であり、HASKI_DOC v0.5の正式な改訂仕様ではない。**
