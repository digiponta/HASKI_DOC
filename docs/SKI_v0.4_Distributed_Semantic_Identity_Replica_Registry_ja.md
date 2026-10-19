# SKI v0.4 — Distributed Semantic Identity & Replica Registry
**Independent Editor's Draft / 日本語 — 2026-10-19**

## 1. 目的
転送元のSemantic Dataと、1件以上の転送先で生成される対応Semantic Dataを相互に識別可能とする。各参加VMは複数の転送先の対応情報をReplica Registryに保持し、更新・再転送・訂正・撤回を追跡する。

**原則:** 意味は自然言語と構造化命題で交換し、ベクトルはローカル生成する（SKI v0.3を継承）。相互IDの共有は同一のベクトル空間や同一の意味・真実性を保証しない。

## 2. 識別子
- `globalSemanticId`：原点が発行するグローバルな系譜識別子。再利用しない。意味の等価性を示す識別子ではない。
- `origin`：原点の `vmId`、`assertionId`、`version`、`issuer`。
- `localAssertionId`：各VMが固有に付与する識別子。原点のIDをコピーしてローカルIDと称してはならない。
- `replicaId`：各転送先レプリカの安定識別子。
- `replicaMapVersion`：対応表の版数（単調増加）。
- `lineage`：転送、翻訳、訂正、派生等の関係。修正された内容を単純な同一レプリカとして扱わない。
- `payloadDigest`：正準化規則に基づく内容ダイジェスト。転送時の完全性検査用で、意味同一性の証明ではない。

## 3. Replica Registry
各Semantic Memoryは、対応データの識別情報を保持する。登録には、`globalSemanticId`、`origin`、`localAssertionId`、`replicas[]`、`replicaMapVersion`を使用する。

`replicas[]`の各エントリは`vmId`、`replicaId`、`localAssertionId`、`sourceAssertionId`、`sourceVersion`、`localVersion`、`contentDigest`、`relation`（COPY_OF / DERIVED_FROM / TRANSLATION_OF / CORRECTION_OF）、`state`（PENDING / CONFIRMED / STALE / CONFLICT / RETRACTED / UNREACHABLE）、`lastAcknowledgedAt`を含むことを推奨する。アクセス制御に応じてPeer情報を部分公開してよい。

## 4. 転送と相互登録
1. 送信側Aが`globalSemanticId`と転送要求IDを含むSKIメッセージを送る。
2. 受信側Bが認証・構文/意味構造・版数・権限を検証し、ローカルID `B-205` を生成してCommitする。
3. Bは`REPLICA_REGISTERED`イベントでB-205、G-001、適用版数、digestをAへ返す。
4. AはBの登録応答を検証して対応表に追加し、`REPLICA_MAP_UPDATE`を関係先へ配布する。
5. Cにも転送された場合、C-309を登録し、A/B/Cが受信・検証できた版の対応表を維持する。
6. 登録イベントは`eventId`で冪等化し、欠落は再取得、競合は保留・保全する。ACK未受領ならCONFIRMEDとしない。

**規範:** 受信者は他者が申告したレプリカIDを勝手にローカルCommit済みとして扱ってはならない。各レプリカの存在・版数を確認できるのは、そのVMの認証済み応答があったときだけである。

## 5. JSON例（草案・非公式Schema）
```json
{
  "skiVersion": "0.4",
  "globalSemanticId": "urn:uuid:550e8400-e29b-41d4-a716-446655440000",
  "origin": {"vmId": "VM-A", "assertionId": "A-101", "version": 1},
  "localVmId": "VM-B",
  "localAssertionId": "B-205",
  "replicaMapVersion": 3,
  "semanticText": [{"language": "ja", "text": "猫は動物である。"}],
  "claim": {"subject": "猫", "predicate": "is_a", "object": "動物", "polarity": true},
  "replicas": [
    {"vmId": "VM-A", "localAssertionId": "A-101", "relation": "ORIGIN", "state": "CONFIRMED"},
    {"vmId": "VM-B", "localAssertionId": "B-205", "relation": "COPY_OF", "state": "CONFIRMED"},
    {"vmId": "VM-C", "localAssertionId": "C-309", "relation": "COPY_OF", "state": "CONFIRMED"}
  ]
}
```
この例は全参加VMから確認応答を受けた後のスナップショットを表す。単一メッセージを受信した時点で全レプリカのCONFIRMEDを推定しない。

## 6. 複数転送先・障害・履歴
- A→BとA→Cは別々の転送IDを持つ。同じG-001でも各VMは独立したローカルIDとAssessmentを保持する。
- BがCへ再転送しても、`origin`と直前の`sourceAssertionId`を両方保持し、A→B→Cの経路を追跡する。
- オフラインPeerに関する情報は最終確認状態として残し、到達可能性や最新性を断定しない。
- Cに内容修正がある場合は版数・ダイジェスト・relationを更新し、内容が原本と等しいと自動推定しない。
- CORRECT/RETRACT時は対象関係先へイベントを配送し、相手のACKを追跡する。受信側のローカル評価、承認、/sleepの状態は独立。

## 7. セキュリティと適合試験
Peer情報自体が機密性を持つため、最小公開・認可・暗号化・監査が必要。外部レプリカマップを無条件に信頼しない。偽造登録、重複、遅延ACK、同時版更新、再転送ループ、ネットワーク分断、部分同期、撤回後の再接続を試験する。

## 8. 実装計画
1. JSON Schema、イベント仕様、ステートマシンとReplica Registry永続化。
2. SEMOS_SIMの3VM模擬環境でA→B/C、A→B→C、分断復旧、競合試験。
3. LLM_SEMとの往復変換・ローカルEmbedding再生成・/sleep履歴との連携。
4. 独立した相互運用テストとアクセス制御・プライバシーレビュー。

**Status:** 未実装の提案仕様。分散合意、意味等価性、モデル重みの同期は保証しない。
