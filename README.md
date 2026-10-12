# Email Permutation Engine

Turns any contact list with **first name, last name, company domain** into verified work emails, at zero marginal data cost. The engine behind [The Revenue Inbox](https://therevenueinbox.com)'s list building, open source.

- **Zero per-lead cost.** You pay for verification requests, not per contact. Proven across **487,977 processed rows**: **120,582 verified addresses from 564,482 requests**.
- **One probe per domain, not per person.** An email pattern is a property of the domain: crack it once, apply it to everyone.
- **Never greylisted.** Stage-major verification order, so consecutive probes always hit different mail servers.
- **Resumable by design.** Checkpoints, a retry queue, and transient errors treated as retries, never as failures.
- **Ships with the whole system.** The engine, the logic as an installable AI skill, the runbook, the prompts, the run history, and a self-hosted web app.

### The one rule

**Never send to `catchall_guess`.** Those domains accept every address, so nothing can be confirmed. About 40 to 47 percent of the guesses are right and the rest land nowhere; they do not bounce, so the damage is a slow invisible drag on sender reputation instead of an obvious spike. `verified_send_ready.csv` is pre-filtered. Use it.

## Quickstart (Python standard library only)

```bash
git clone https://github.com/pandush-blip/email-permutation-engine
cd email-permutation-engine
cp .env.example .env      # paste your MailTester.Ninja key

python email_engine.py --input clients/acme/input.csv --output-dir clients/acme/output --dry-run
python email_engine.py --input clients/acme/input.csv --output-dir clients/acme/output --live-sample --domain-limit 50
python email_engine.py --input clients/acme/input.csv --output-dir clients/acme/output --full-run --workers 5
python email_engine.py --output-dir clients/acme/output --status
```

## The stages

1. **Normalize.** Transliterate to ASCII, strip suffixes and punctuation, expand hyphenated and multi-token last names into variants. Skipping this quietly destroys match rate.
2. **Catch-all pre-check.** One request per domain to a random nonexistent mailbox. Catch-all domains get quarantined.
3. **Pattern detection.** One probe contact per domain climbs the ranked pattern ladder. The first Accepted wins the pattern for the whole domain.
4. **Propagation.** Apply the solved pattern to every remaining contact at the domain.
5. **Retry queue + output.** Transient errors retried with backoff; every row preserved, only labelled.

## Outputs

| File | Contents |
|---|---|
| `verified_send_ready.csv` | Verified only. This goes to the sending tool. |
| `enriched_all.csv` | Every input row, all original columns, plus email and status. |
| `results.jsonl` | Append-only audit log. Enables resume. Never delete mid-run. |

Row count of `enriched_all.csv` always equals the input. Nothing is deleted, only labelled.

## Status values

| Status | Sendable | Meaning |
|---|---|---|
| `verified` | Yes | Mail server accepted it. |
| `catchall_guess` | **No** | Domain accepts everything. Pattern is a guess. |
| `check_failed` | Not yet | Probe blocked or timed out. Retriable. |
| `person_invalid` | No | Domain pattern known, this person rejected. Usually left the company. |
| `pattern_unknown` | No | No pattern cracked. |
| `dead_domain` | No | No MX record. |
| `duplicate_contact` | No | Same address as an earlier row. Mailed once. |

## Expected yield

Latest comparable profile: a small B2B software list, 32,615 raw rows, 30,263 unique usable contacts.

| Status | Share of raw rows | Share of unique usable contacts |
|---|---:|---:|
| `verified` | 31.30% | 33.74% |
| `catchall_guess` | 37.11% | 40.00% |
| `check_failed` | 4.94% | 5.33% |
| other | 26.65% | 20.93% |

For similarly sourced lists, plan about **3.19 raw rows** or **2.96 unique usable contacts** per verified email. Recalculate from your latest comparable run before quoting a forecast.

## What's inside

```text
email_engine.py             the engine: stages, checkpoints, retry queue, resume
verify_harvested_emails.py  verification-only mode
SKILL-Permutation.md        the entire logic as an installable AI skill
RUNBOOK.md / MANUAL.md / PROMPTS.md    setup, walkthrough, copy-paste run sequence
RUN_HISTORY.md              aggregate benchmarks from every completed run
ENGINE-PATCHES.md           engine improvements and next experiments
skills/lead-enrichment/     a second skill: guided intake to enriched outputs
webapp/                     self-hosted Harvester app: upload, map, watch, download
clients/                    your lists live here (gitignored)
```

## The web app (needs Flask)

```bash
pip install -r webapp/requirements.txt
python webapp/worker.py    # background worker
python webapp/app.py       # serves on 127.0.0.1:8080
```

## Client data never enters this repo

`clients/` and `.env` are gitignored. Lead lists are client property, and a key committed once is committed permanently. Delete client folders when an engagement ends.

---

Built by [The Revenue Inbox](https://therevenueinbox.com). Steal it, self-host it, change it.
