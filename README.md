# Sourcing-Agents

A multi-agent sourcing system for regulated procurement, built on **LangGraph** with **Claude on Vertex AI**.
Given a buyer's request ("40 t of MCC PH-102 for Raleigh by March"), it finds candidate suppliers,
runs compliance and feasibility checks on each one in parallel, ranks them, and drafts a recommendation
in which every sentence cites its source. It then decides whether a human has to sign off, and who.
Every step is written to a hash-chained audit log.

The thesis: **in regulated workflows the model should explain decisions, not make them.** Code applies the
hard constraints (restricted parties, GMP status, warning letters, capacity, lead time). The model turns
the evidence into a memo, a verifier checks that memo claim by claim, and a policy layer decides how much
autonomy this particular decision gets.

> All suppliers, certificates, list entries and people in `data/` are fictional.

## Quickstart

```bash
pip install -e ".[dev]"
pytest                                   # 31 tests, no credentials needed
python evals/run_evals.py                # golden scenarios + fault injection + injection ablation
python -m sourcing_agents.cli "Source 40 t of MCC PH-102 for the Raleigh plant, need by 2027-03-31."

# live model
export GOOGLE_CLOUD_PROJECT=... VERTEX_REGION=us-east5 VERTEX_CLAUDE_MODEL=<vertex model id>
python evals/run_evals.py --backend vertex
```

## Graph

```
intake ──missing fields──> clarify (interrupt) ──> intake
  │
retrieve ──Send() x N──> due_diligence      parallel map: compliance + feasibility per supplier
                              │
                            rank             reduce: deterministic scoring and selection   <───────┐
                              │                                                                    │
                            draft (Claude) <── verifier feedback, max 3 ── verify                  │
                                                                          │                        │
                                                                       route (policy)              │
                                       ┌──────── requires human ─────────┘                         │
                                 human_review (interrupt) ── exclude supplier ────────────────────┘
                                       │   approve per role, bound to recommendation digest
                                    finalize ── decision_record.json + audit.jsonl
```

| Module | Responsibility |
|---|---|
| `corpus.py` | Dossier chunking, BM25 retrieval, fact extraction with provenance, prompt-injection quarantine, fuzzy restricted-party screening |
| `rules.py` | Versioned compliance and feasibility rules, landed cost (FX × trade lane), transparent weighted scoring |
| `llm.py` | Backend interface; `ClaudeVertexLLM` (forced tool use, Pydantic validation, schema-repair retry, prompt caching); `OfflineLLM`; `FaultyLLM` |
| `grounding.py` | Claim-level verifier: citation exists, not quarantined, every number and date present in cited text, entity match, decision consistency, optional LLM judge |
| `policy.py` | Risk-tiered human-in-the-loop routing with named reasons and approver roles |
| `graph.py` | LangGraph wiring, interrupts, checkpointing with an allow-listed serializer, decision record |
| `audit.py` | SHA-256 hash-chained JSONL log and verifier |

## Design decisions

- **The LLM explains; code decides.** Selection is computed before the drafter runs. The drafter gets the
  decision as input, and if it returns a different `selected_supplier_id`, verification fails.
- **Absence of evidence is not compliance.** A missing certificate or audit becomes a `CONDITION` that a
  named role has to clear, never a silent pass.
- **Conditional suppliers never outrank clear ones.** In the standard scenario Rheinland scores highest
  (0.975) but has no executed quality agreement, so Cascadia (0.965) is selected and Rheinland is
  explained as "not yet clear".
- **Supplier text is untrusted input.** Chunks that address the AI system or try to issue instructions are
  quarantined before fact extraction. The ablation shows why this matters. With quarantine off, Pacific
  Rim's injected lead time of 7 days is still ignored, because quote fields are only taken together from a
  valid quote. Its injected "Minor findings: 0" is not: it wins the latest-document rule, lifts Pacific Rim's
  quality score from 0.70 to 1.00, and flips the award. One structural defence held and one didn't, which
  is why quarantine runs before extraction.
- **Approvals are bound to a recommendation digest.** If a reviewer excludes a supplier, the graph re-ranks
  and earlier approvals stop counting. Separation of duties is enforced: one person can't approve as two roles.
- **Clarify rather than assume.** "ASAP" is not a date, and an unknown material is not mapped to its
  nearest match. The graph interrupts and asks.
- **Checkpoints rehydrate only known types** (`pickle_fallback=False` plus a msgpack allow-list), so a
  tampered checkpoint can't instantiate arbitrary classes.

## Evaluation (offline backend)

| Suite | Result |
|---|---|
| Golden scenarios (expectations written by hand from policy) | 11/11 |
| Supplier verdict accuracy (18 labelled supplier × scenario pairs) | 100% |
| Blocked supplier ever selected | never |
| Fault injection: 6 hallucination classes × transient/persistent | 12/12 (transient faults recovered by revision; persistent ones escalated to a human) |
| Prompt injection changes the award: quarantine on / off | no / **yes** |
| Audit chains verified | all |

Full output: `evals/results/results-offline.json`. A sample decision record and audit log are in `examples/`.

## Limitations (stated plainly)

- The numbers above come from the **deterministic offline backend**, which is grounded by construction. They
  prove that the graph, the rules, the routing and the verifier work. They don't measure Claude's drafting
  quality. The Vertex path is implemented and unit-tested against a mocked client; running
  `--backend vertex` gives the real numbers.
- Fact extraction parses structured `Label: value` lines. Real dossiers (scanned PDFs, CoAs) need an LLM
  extraction step with its own eval set. That step would plug in at `corpus.extract_facts`.
- The injection screen is pattern-based. It is one layer: the others are quarantine, deterministic
  decisions and the verifier. A trained classifier would also be needed in production.
- Quarantine works at the chunk level, so legitimate data inside a poisoned document is lost too
  (Shandong's quote). That is the conservative choice, and the finding says so.
- The corpus is 9 suppliers. BM25 is enough at that size; a large corpus would need hybrid retrieval
  (Vertex AI Search or pgvector) plus a reranker.

## Production path on GCP

Postgres checkpointer on Cloud SQL, so runs can wait at a human gate for days. Cloud Run or Vertex AI Agent
Engine for the graph. Audit records streamed to BigQuery, with the chain head anchored in a
retention-locked GCS bucket. Review packets delivered to the procurement tool, with resumes posted back as
`Command(resume=...)`. Evals run in CI on every prompt, rule or model version change, with the
`--backend vertex` suite as a release gate.
