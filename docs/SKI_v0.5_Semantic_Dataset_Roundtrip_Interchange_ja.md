# Semantic Knowledge Interchange (SKI) v0.5
## Natural-Language Dataset Roundtrip, Correlation & Heterogeneous LLM Interoperability
**Independent Editor’s Draft — 日本語版 — 2026-10-21**

**基本原則:** 「意味を交換し、ベクトルは交換しない」。本仕様はSKI v0.1–v0.4に基づく検討版であり、国際標準ではない。データ転送の整合性と意味的妥当性は別の検証項目である。

## 1. 背景と根拠
[SEMOS_S2N](https://github.com/digiponta/SEMOS_S2N) の2026-10-21時点の `ski_nl_exchange.py`、`ski_dataset_roundtrip.py`、`SKI_DATASET_ROUNDTRIP.md` を参照する。同実装は `ski-natural-language-v0.5` メッセージのJSON配列をDataset A/Bとして扱い、単一メッセージの整合性、データセットACK、応答の相関をローカル検証する。

**観測済み:** ローカルの2件サンプルで `local-protocol-smoke-pass`、A/B各2件、同一フォーマット、相関検証成功（ユーザー実行ログ）。**未実証:** 遠隔通信、実LLM推論、意味の正確な再構成、実装間の独立適合性。既存SEMOS_S2Nのメッセージ形式は本稿が正式に規範化する以前の**実験形式**であり、過去の完全準拠を遡及的に主張しない。

## 2. 範囲・非目標・規範語
**対象:** 自然言語命題と属性情報からなるDataset Aを外部LLMへ転送し、転送先で生成された独立のSemantic DataをDataset Bとして返すプロトコル、送達確認、データの来歴・対応付け、複数転送先の管理。

**対象外:** モデル間のEmbedding空間整合、Embedding自体の伝送、知識の真偽保証、モデル重み同一化、学習済み知識の完全消去、任意の日本語命題の確実な意味解析。

**MUST / MUST NOT** は必須、**SHOULD** は逸脱理由を記録する推奨、**MAY** は任意とする。

## 3. v0.4からの継承
v0.4の `globalSemanticId`、`origin`、`localAssertionId`、`replicas[]`、`replicaMapVersion`、訂正/撤回イベントを維持する。Global IDは**来歴グループ/系譜**を識別し、内容が等価であることを保証しない。コピー、翻訳、回答、訂正を区別する。

## 4. Semantic Data交換モデル
- Dataset A：送信側が保有する自然言語命題・属性情報のメッセージ集合。
- Dataset B：受信側が**自身のLLM/処理系から新たに生成**して返すSemantic Dataメッセージ集合。Aの単純なコピーを意味しない。
- 両者は同一の**SKIメッセージ形式**を使う（MUST）。JSON配列は基本コンテナ（MUST）。配列外に別の専用S2Nメッセージ形式を導入しない（MUST NOT）。
- 配列内には異なるClaim、条件、来歴を持つ複数メッセージを含めてよい。空配列の扱いはプロファイルで宣言する（基準プロファイルでは禁止）。
- v0.5の基本プロファイルは1命題=1メッセージ。1要求に0・1・複数の応答を認める拡張プロファイルは別途定める。SEMOS_S2N現行の**1対1制約はデモアダプタ固有**であり、SKI全体の原則ではない。
- 言語は `language` で宣言し、`text` と構造化 `claim` の対応を検証する。現行デモが日本語のみを処理することは全SKIへの言語制限を意味しない。

## 5. メッセージフィールド
| 必須度 | フィールド | 内容 |
|---|---|---|
| MUST | `format` | `ski-natural-language-v0.5` |
| MUST | `message_id` | 交換メッセージの安定ID（同一配列内で一意） |
| MUST | `source_llm`, `target_llm` | 発信/宛先処理系の識別子 |
| MUST | `source_sd_id` | 当該発信側Semantic DataのID |
| MUST | `language`, `text` | 言語・自然言語による意味内容 |
| MUST | `text_sha256` | UTF-8テキストのSHA-256 |
| SHOULD | `parent_sd_id` | 直前の親Semantic Dataへの参照 |
| SHOULD | `derivation` | 応答/翻訳/複製などの系譜・相関情報 |
| SHOULD | `claim`, `context`, `conditions` | 構造化命題・条件・文脈 |
| SHOULD | `globalSemanticId`, `origin`, `replicas` | v0.4互換の分散識別情報 |
| SHOULD | `provenance`, `assessments`, `lifecycle` | 出典・評価・承認/撤回状態 |
| MAY | `local_model_metadata` | 再生成モデルの追跡用（送信側ベクトル自体は不可） |

**注意:** この表はSEMOS_S2N現行の最小メッセージに対してv0.4互換要素を追加した仕様提案。現行プログラムが全フィールドを生成・検査するとは限らない。公開JSON Schemaが実装されるまでは草案である。

## 6. LLM識別規則
Semantic Dataは生成元LLMの識別情報を持つ。ローカルベクトルを再利用するのは**同一ベクトル空間/モデルID・版数が検証できる場合のみ**。異なるモデル間では元の埋め込みを解釈可能と仮定しない（MUST NOT）。

異なるLLMに渡す場合は、元のSemantic Dataを自然言語化し、受信側で新規Semantic Dataとして再構成する（SHOULD）。この処理は自動的な意味同値の証明ではなく、原点・親・派生関係を保全し、必要なら専門分野/条件の合致確認および人間のレビューを求める。元のベクトルを共通座標に投影して同一知識とみなさない。

## 7. Dataset A→ACK→Dataset Bの処理
1. Aの生成：送信者がSKIメッセージ配列に `source_llm`、`target_llm`、`source_sd_id`、`message_id`、`text_sha256` を付与。
2. 送信前：構文・重複ID・宛先・本文ダイジェストを確認。
3. 受信：宛先受信器が構文・宛先LLM識別子・テキストSHA-256・メッセージID・認可を検証し、ローカル処理対象として受理/隔離。
4. ACK：Dataset A全体のダイジェストとメッセージID集合を含む**別個の受領記録**を返す。ACKは配送/受理処理段階の状態であり、解釈・正しさ・知識承認を証明しない。
5. 受信側の再Semantic化：現行試作では提供済みの返信テキストを新たなSemantic Dataにしており、実LLMの意味理解は呼び出していない。実装時は明示的な再解釈と評価ステップを別に記録する。
6. Dataset B：Aと**同じSKIメッセージ形式**で新たな `source_sd_id`、`message_id`、`text`、`text_sha256` を生成。
7. 対応付け：Bの `derivation.response_to_message_id` をAのIDへ、`derivation.source_sd_id` をAの `source_sd_id` へ対応付け、送受信側のLLM識別子の逆転を検証。
8. Replica Registry：応答に応じてローカルID・origin・派生関係を更新。確認済みACKがない対象をCONFIRMEDとしてはならない。

## 8. 受領ACKと応答の分離
ACKは `protocol`、`receiver_llm`、`message_ids[]`、`dataset_sha256` を持つ別の制御レコード。データセットのダイジェストは**正準化したJSON**に対しSHA-256で計算する。基準のデモでは `json.dumps(..., sort_keys=True, separators=(',', ':'), ensure_ascii=False)` とUTF-8を用いる。別実装は同一の正準化方式を固定しなければならない。

**ACK受領 ≠ Dataset B生成 ≠ 解釈成功 ≠ 承認 ≠ /sleep内部化。** 状態を混同しない（MUST NOT）。

## 9. データセット例（説明用）
Dataset A:
```json
[
  {
    "format": "ski-natural-language-v0.5",
    "message_id": "ski-EXAMPLE-A-001",
    "source_llm": "LLM-A",
    "target_llm": "LLM-B",
    "source_sd_id": "A-101",
    "parent_sd_id": null,
    "language": "ja",
    "text": "猫は動物である。",
    "text_sha256": "<SHA-256 of UTF-8 text>",
    "derivation": null
  }
]
```
Dataset B:
```json
[
  {
    "format": "ski-natural-language-v0.5",
    "message_id": "ski-EXAMPLE-B-001",
    "source_llm": "LLM-B",
    "target_llm": "LLM-A",
    "source_sd_id": "B-205",
    "parent_sd_id": "A-101",
    "language": "ja",
    "text": "猫は動物に分類される。",
    "text_sha256": "<SHA-256 of UTF-8 text>",
    "derivation": {
      "response_to_message_id": "ski-EXAMPLE-A-001",
      "source_sd_id": "A-101"
    }
  }
]
```
**重要:** 上記IDとダイジェストは**説明用プレースホルダ**であり、実装の `ski_nl_exchange.py` にそのまま入力して適合する有効データではない。正常系試験には同実装が自動生成した実ID/ダイジェストを使用する。

## 10. 相関・完全性の検証
- Dataset A/Bは同じ `format` を要求する。
- 各データセット内 `message_id` は一意。
- `text_sha256` を再計算して検証し、`message_id` について宣言された生成規則を満たすか確認。
- `source_llm` / `target_llm` の方向、受信器ID、`source_sd_id`、応答相関、応答の重複を検証。
- v0.4互換の `origin`、`globalSemanticId`、`replicas[]` がある場合、派生関係と登録状態を整合確認。
- 相関一致・SHA一致は**意味同値を保証しない**。
- `text_sha256` は破損発見に用いるもので、送信元の認証や改ざん防止には認証済みTLS・署名等を併用する（SHOULD）。

## 11. Meaning / Truth / Unknown の別プロファイル
`SEMOS_S2N`のv0.9.2評価器は限定した日本語命題について、ローカルのcosine値とsubject/relation/polarity照合を組み合わせ、KNOWN / UNKNOWN / REVIEWを返す**限定ルール実験**である。汎用構文解析、命題の真偽検証ではない。

- `KNOWN`：限定した評価方法で登録命題との整合が得られた状態。科学的真実の証明ではない。
- `UNKNOWN`：対象命題と条件などが一致しないと判断した状態。FALSEと同義ではない。
- `REVIEW`：低信頼、曖昧、解析対象外として判断保留。UNKNOWNと混同しない。

v0.9.3では別途作成した12例の開発用holdoutを使用するが、独立無作為抽出された外部コーパスではない。結果を一般化性能として主張しない。SKI-Core適合性試験と意味品質ベンチマークは別管理する。

## 12. 3VM / 複数転送先
送信AがB/Cに同一の原命題を送っても、それぞれ異なるローカルSD ID・評価・Embeddingを持つ。v0.4のReplica Registryを保全し、A→B、A→C、A→B→Cを別イベントとして履歴化。受信先の応答は `COPY_OF`、`DERIVED_FROM`、`TRANSLATION_OF`、`RESPONSE_TO` などの関係を区別する。相互情報を全ノードに確実に通知できない場合は未確認状態を表示する。

## 13. セキュリティ・ガバナンス
リプレイ、メッセージID衝突、ダイジェスト不整合、偽送信者、権限外宛先、過大データセット、外部URI取得、機密情報露出、プロンプトインジェクション、返信内容の不審な命令に対処する。異種LLM間で自動的に知識承認/学習させない。外部の自然言語内容は**データ**であって命令ではない。

## 14. 最低限の適合試験と実装計画
### Protocol conformance
1. A/B同形式、非空JSON配列、各メッセージに必須属性。
2. 宛先/発信者ID不一致、改ざんテキスト、重複ID、欠落相関、不正ハッシュを拒否。
3. ACKの件数・message ID集合・dataset hashの一致。
4. コピー/派生/応答を区別して、originと転送経路の追跡ができる。
5. 複数転送先へのレジストリ配布・訂正・撤回・ネットワーク分断と再接続。
### Meaning-quality evaluation (separate)
1. 原文命題と受信側再構成命題のsubject/predicate/object、条件、否定、数量、時制、来歴保持を人手ラベルを用いて評価。
2. 厳密一致、意味保持、矛盾率、Unknown/REVIEW、coverage/selective accuracyを測定。
3. 開発用・検証用・未見の最終テストを分離する。

### 実装ロードマップ
- **v0.5仕様確定:** メッセージ形式、Dataset schema、ACK正準化、データモデルと相関ルールの統一。
- **実装:** SEMOS_S2Nの現行prototypeを公式SKI schemaへ統合、実LLM adapterと認証済み通信を追加。
- **検証:** 1:N応答、複数転送先、障害、再送、意味保持、Independent consumer適合試験。
- **将来:** 標準語彙IRI/JSON-LD context/SHACL shapesと公開ガバナンス整備。

## 参考実装・関連資料
- https://github.com/digiponta/SEMOS_S2N/blob/main/SKI_DATASET_ROUNDTRIP.md
- https://github.com/digiponta/SEMOS_S2N/blob/main/ski_nl_exchange.py
- https://github.com/digiponta/SEMOS_S2N/blob/main/ski_dataset_roundtrip.py
- https://github.com/digiponta/SEMOS_S2N/blob/main/SEMOS_TRUTH_GATE_V092.md
- https://github.com/digiponta/SEMOS_S2N/blob/main/SEMOS_HOLDOUT_V093.md
