# SKI Diagnostic Response Profile v0.14 (Editor's Draft)

**Status:** Experimental interoperability profile, 2026-11-11.
**Compatibility:** Optional metadata field on a SKI v0.5 Natural-Language
Dataset B message. **Not** a new canonical SKI v0.5 wire format and not
an externally ratified standard.

## 1. Motivation and boundary

LLM_CHA issues a SKI Dataset A inquiry to LLM_TMP, and LLM_TMP returns
Dataset B, correlated to Dataset A with
`derivation.response_to_message_id` and `parent_sd_id`.
The existing v0.13 implementation already returns a bounded natural-language
status in Dataset B and includes an `assessments.evidence_status`.
The local LLM_CHA Truth Boundary also creates an independent diagnostic
decision. That decision is not itself an SKI reply.

This profile makes the *source structural diagnosis* explicitly available
**inside SKI Dataset B**. The receiver must validate the incoming field
against Dataset A and the experimental Evidence request/response envelopes,
then independently apply the existing Truth Boundary.

The profile cannot prove sender authentication, factual correctness,
actual human approval or suitability for model answer generation. All
positive graph hits remain unauthenticated claims about stored relations.

## 2. Transport

No additional Dataset C, no new network protocol and no change to ACK.
Keep the SKI v0.5 message array, with the same:
`format`, `message_id`, `source_llm`, `target_llm`,
`source_sd_id`, `parent_sd_id`, `language`, `text`,
`text_sha256`, and `derivation` fields.

Add **optional** metadata `diagnostic` to the Dataset B message.
Dataset A and the existing ACK remain unchanged.

## 3. Dataset B diagnostic fields

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
  "diagnostic_sha256": "<lowercase hex SHA-256>"
}
```

`diagnostic_sha256` is SHA-256 of the UTF-8 JSON object with
`sort_keys=true`, `ensure_ascii=false`, separators `(',', ':')`
**excluding** the hash field. It detects mismatches or mutations but
does **not** provide cryptographic origin authentication. Hashes cannot
replace signatures, identity verification or human review.

`diagnostic_status` belongs to the bounded set:
`FOUND`, `NOT_FOUND`, `UNKNOWN_NODE`, `LIMIT_REACHED`,
`DEPTH_LIMIT_REACHED`. A `NOT_FOUND` result does not prove
nonexistence. `evidence_path_hops` is a bounded nonnegative integer
for `FOUND`, otherwise `null`.

No raw evidence path, untrusted provenance text, retrieved natural
language, tensor, vector or embedding is included in this diagnostic.
The existing experimental Evidence response remains separately
available for structural validation, never for automatic truth promotion.

## 4. Required receiver validation

A compliant v0.14 receiver SHOULD:
1. Validate Dataset A, Dataset B, ACK, and A/B message/SD lineage.
2. Validate experimental Evidence request/response correlation.
3. Validate the optional Dataset B `diagnostic` metadata: exact source
   and target SD IDs, both Evidence message IDs, response-to A message ID,
   status, path hop count and strict safety flags.
4. Reject extra or changed diagnostic fields; do not trust a separately
   recomputed checksum as proof of authority.
5. Apply its own Truth Boundary and never automatically pass received
   Evidence to the generative model or learning system.

Legacy v0.13 Dataset B messages without `diagnostic` are still
accepted by the unchanged base profile. A v0.14-specific validator
requires the diagnostic to be present.

## 5. Trust-state semantics

`APPROVED` on a stored graph relation means only that the local graph
record bears that state. It does not imply authenticated user consent,
evidence authenticity, verified scientific fact, or that a receiving LLM
may use it. Therefore the profile fixes:

- `truth_state=UNVERIFIED`
- `approval_state=UNAUTHENTICATED`
- `verified_fact=false`
- `human_approved=false`
- all `allow_*` fields to `false`

Any future authenticated provenance or human-approved knowledge exchange
requires a separate, explicit authorization and verification workflow.

## 6. Reference implementation and current scope

- `digiponta/LLM_SKI_BRIDGE`, branch
  `feature/ski-diagnostic-v0.14`:
  `ski/diagnostic_v014.py`,
  `adapters/llm_cha_client_v05.py`,
  `adapters/cha_truth_boundary_v07.py`.
- `digiponta/LLM_CHA`, branch
  `feature/ski-diagnostic-v0.14`:
  `ski_runtime_v011.py` requires the profile before displaying a
  diagnostic when the v0.14 Bridge is configured.
- LLM_TMP native `SDRegistry` / `RelationGraph` remains unchanged.
- Verification uses the **synthetic** native SD snapshot and real local
  class implementations. Actual production Semantic Registry semantics,
  real user approvals, cross-machine networking and model-generated
  grounded answers are out of scope.

**Important:** This proposal is a local project profile. It is not an
ISO/W3C/JIS-approved SKI protocol version, nor a revision of the
HASKI_DOC v0.5 specification.
