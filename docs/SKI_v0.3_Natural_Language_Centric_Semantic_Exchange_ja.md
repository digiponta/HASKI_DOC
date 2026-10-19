# Semantic Knowledge Interchange (SKI) v0.3
## Natural-Language-Centric Semantic Exchange — Core, Lifecycle & Distributed Synchronization

**公開技術仕様草案 / Independent Editor’s Draft — 2026-10-19**

> **設計原則：意味を交換し、ベクトルは交換しない。** このMarkdownは同版DOCX技術仕様草案のリポジトリ向け要約・実装者向け記述である。W3C・ISO/IEC・JISなどの承認規格ではない。

## 1. 目的と範囲
異なるLLM、Semantic Memory、Semantic OS、VMがEmbedding空間を共有せずに知識を交換する。交換対象は**自然言語の意味記述と構造化命題**、文脈、条件、来歴、評価、履歴とする。送信側Embedding、centroid、projection tensor、モデル重みの移転は基本プロファイルの範囲外。真実性や重みからの完全削除は保証しない。

**MUST / MUST NOT** は適合必須、**SHOULD** は推奨（逸脱理由を記録）、**MAY** は任意。

## 2. v0.1・v0.2との関係
- **v0.1**：主張、文脈、来歴、評価状態のモデル非依存交換。
- **v0.2**：SKI-Core、Lifecycle、Sync、Internalization、Transport；各VM独立Semantic Memory。
- **v0.3**：`semanticText`と構造化`claim`の組を交換の中心に定義。Embeddingは受信側で独自生成する。

## 3. 設計原則
1. 意味の正準記録は自然言語と構造化Claimを組み合わせる。
2. 同次元ベクトルであっても異なるモデルの座標空間が一致すると仮定しない。
3. 受信・承認・真偽判定・内部学習を独立した操作とする。
4. 否定、条件、時制、数量、単位、概念識別子を保持する。
5. 言語間翻訳・要約・ベクトル再生成は意味保存を保証しない。変換来歴を残す。

## 4. プロファイル
| Profile | 役割 |
|---|---|
| SKI-Core / SKI-Text | 自然言語、構造化命題、文脈、条件、来歴と評価 |
| SKI-Lifecycle | ASSERT、ASSESS、APPROVE、CORRECT、RETRACT |
| SKI-Sync | イベント同期、冪等性、版数、因果関係、競合と再送 |
| SKI-Internalization（任意） | `/sleep`に伴うローカルモデル内部化履歴 |
| SKI-Transport（任意） | 認証済みTLS通信、必要に応じてRDMA等 |

## 5. SKI-Core / Text データモデル
| 必須度 | 属性 | 定義 |
|---|---|---|
| MUST | `skiVersion` | `"0.3"` |
| MUST | `assertionId` | 安定識別子。テキスト類似度のみで生成しない |
| MUST | `issuer`, `recordedAt` | 発行主体、タイムゾーン付き時刻 |
| MUST | `version`, `lifecycle` | 版数（1以上）、管理状態 |
| MUST | `semanticText[]` | 言語タグ付き自然言語表現 |
| MUST | `claim` | subject / predicate / object と polarity |
| SHOULD | `context[]`, `conditions[]` | 成立条件・適用文脈 |
| SHOULD | `assessments[]`, `provenance` | 誰が・いつ・どう評価したかと出典 |
| SHOULD | `accessPolicy` | 共有条件と保護区分 |
| SHOULD | `representationPolicy` | 基本値 `LOCAL_ONLY` |

構造化命題と自然言語が矛盾する場合は**自動承認せずCONFLICTまたはQUARANTINED**とする（MUST）。表現の修正と意味を変えるCORRECT操作を区別する。

## 6. JSON交換例（非公式語彙）
```json
{
  "skiVersion": "0.3",
  "assertionId": "urn:uuid:550e8400-e29b-41d4-a716-446655440000",
  "issuer": "urn:vm:a",
  "recordedAt": "2026-10-19T13:00:00+09:00",
  "version": 1,
  "lifecycle": "CANDIDATE",
  "semanticText": [
    {"language": "ja", "text": "猫は動物である。", "role": "canonical"}
  ],
  "claim": {
    "subject": {"label": "猫", "conceptId": "urn:concept:cat"},
    "predicate": {"id": "is_a"},
    "object": {"label": "動物", "conceptId": "urn:concept:animal"},
    "polarity": true
  },
  "context": [],
  "conditions": [],
  "assessments": [{
    "assessor": "urn:vm:a",
    "status": "UNVERIFIED",
    "assessedAt": "2026-10-19T13:00:00+09:00"
  }],
  "representationPolicy": "LOCAL_ONLY"
}
```
本例は説明用であり、公開済み公式JSON Schema、JSON-LD context、語彙IRIを意味しない。

## 7. ローカルベクトルの扱い
異種モデル間のEmbeddingの意味空間一致を仮定してはならない（MUST NOT）。受信側は`semanticText`と`claim`から独自のEmbedding、Role Vector、Prototype、検索索引を生成できる（MAY）。生成結果には`localModelId`、`checkpointId`、`encoderVersion`、`dimension`、`createdAt`、`sourceAssertionId/version`、`validationState`を記録することを推奨する。モデル更新時の再生成・失効はローカル責務。

## 8. Lifecycle / Assessment
操作は`ASSERT`、`ASSESS`、`APPROVE`、`CORRECT`、`RETRACT`。Lifecycle（CANDIDATE / APPROVED / SUPERSEDED / RETRACTED）、Epistemic（TRUE / FALSE / UNVERIFIED / CONTESTED / OUTDATED）、Realization（EXTERNAL / CACHED / INTERNALIZED / UNKNOWN）は別々に記録する。外部APPROVEをローカル承認として無条件適用してはならない。

## 9. SKI-Sync：各VMが独立のSemantic Memoryを保持
VM-A、VM-B、VM-C間で共有可能な命題と変更イベントを同期する。イベントは`eventId`、`assertionId`、`issuer`、`originVm`、`operation`、`previousVersion`、`newVersion`、`timestamp`、`payloadDigest`、必要なら`causalParents[]`を持つ。

受信処理：**認証 → スキーマ検証 → 権限確認 → 版数・因果関係照合 → ローカルCommit → ACK**。同じeventIdは冪等処理し、欠落イベントは再送またはsnapshotで回復する。競合する訂正やAssessmentの来歴を保持し、Last-Writer-Winsのみで解決しない。2VM間の受領だけでは分散合意を意味しない。

## 10. SKI-Internalization（/sleep）
同期Commitと学習を分離する。任意の内部化記録には`vmId`、`modelId`、`checkpointId`、版数付き`inputAssertionIds`、`startedAt`、`completedAt`、`trainingStatus`、`validationResult`を残す。撤回後もモデル重みからの完全削除を保証しない。

## 11. Transport、セキュリティ
SKIは転送基盤非依存。TLSによる送信者認証とACKを推奨。RDMAは大きなsnapshotやバッチに任意利用できるが、**RDMA転送完了 ≠ SKI COMMITTED**。入力サイズ・改ざん・リプレイ・権限を検証し、外部文書内の命令を実行しない。外部URI自動取得、SSRF、プロンプトインジェクション、機密、著作権、個人情報を管理する。

## 12. 検証計画
提案PoC：3VM、各100件の知識と異なるEmbeddingモデルを使用。意味保持、誤承認、訂正・撤回・翻訳・否定・条件付き命題、障害復旧、収束、P50/P95遅延、転送サイズを測定する。**実測結果や適合性認証の存在を主張しない**。

## 13. ロードマップ
- v0.3：規範語彙・JSON Schema/JSON-LD context・例・検証器整備。
- 次段階：SKI-Text往復変換、LLM_SEM/LLM_TRY/SEMOS_SIMと3VM同期の実装、独立holdout評価。
- 将来：相互運用・適合性試験、セキュリティレビュー、変更管理。

## 参考
[JSON-LD 1.1](https://www.w3.org/TR/json-ld11/) · [RDF](https://www.w3.org/TR/rdf11-concepts/) · [PROV-O](https://www.w3.org/TR/prov-o/) · [SHACL](https://www.w3.org/TR/shacl/) · [MCP](https://modelcontextprotocol.io/)

旧版：[SKI v0.1](SKI_SPECIFICATION_v0.1_ja.md) / [SKI v0.2](SKI_v0.2_Core_Data_Model_Lifecycle_Distributed_Synchronization_ja.md)。
