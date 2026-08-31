# Supply Chain Message-Drop Audit

## Choice

I ran the built-in `supply_chain` scenario and changed exactly one supported
setting:

```yaml
failures:
  message_drop: 0.1
```

The angle is a growth-funnel reliability audit. In overseas growth work,
campaign performance can depend on staged handoffs across title selection,
creative generation, platform delivery, and retention or lead measurement. I
used `supply_chain` as a simple proxy for that kind of staged workflow: small
handoff failures can turn into missing downstream artifacts because each stage
can only process what it actually receives.

I kept the experiment intentionally small: run the baseline, change one failure
knob, then compare trace events, metrics, and validators. That made the result
easier to attribute than adding a new agent or changing several scenario
settings at once.

## Hypothesis

The baseline should deliver every material because `message_drop` is `0.0`.
With `message_drop` raised to `0.1`, I expected delivery to fall below 100%,
but I also expected the effect to be nonlinear: one early dropped material
would remove the downstream product, shipment, and delivery events for that
item. This mirrors funnel experiments where a loss near the top of the funnel
can change later-stage volume even when the downstream logic is still correct.

## Commands

```bash
uv sync
uv run nest doctor

mkdir -p traces reports
uv run nest run supply_chain -o traces/supply_chain_baseline.jsonl
uv run nest inspect traces/supply_chain_baseline.jsonl
uv run nest report traces/supply_chain_baseline.jsonl -o reports/supply_chain_baseline.html

uv run nest run scenarios/supply_chain_message_drop_audit.yaml
uv run nest inspect traces/supply_chain_message_drop_audit.jsonl
uv run nest report traces/supply_chain_message_drop_audit.jsonl -o reports/supply_chain_message_drop_audit.html
```

Validator check:

```bash
uv run python - <<'PY'
from pathlib import Path
from nest_core.validators import validate_trace

for trace, scenario in [
    ("traces/supply_chain_baseline.jsonl", "supply_chain"),
    ("traces/supply_chain_message_drop_audit.jsonl", "supply_chain"),
]:
    print(trace)
    for result in validate_trace(Path(trace), scenario):
        print(("PASS" if result.passed else "FAIL"), result.name, "-", result.detail)
PY
```

## Evidence

| Run | message_drop | sends | receives | dropped | success_rate | validator result |
| --- | ---: | ---: | ---: | ---: | ---: | --- |
| Baseline | 0.0 | 60 | 60 | 0 | 1.0 | 15/15 materials delivered |
| Drop audit | 0.1 | 45 | 36 | 9 | 0.8 | 7 of 15 materials not delivered |

The modified trace produced these stage counts:

| Event prefix | Count |
| --- | ---: |
| material | 15 |
| product | 12 |
| shipment | 10 |
| delivered | 8 |

Dropped messages by hop:

| Hop | Dropped messages |
| --- | ---: |
| supplier-0 to manufacturer-0 | 3 |
| manufacturer-0 to distributor-0 | 2 |
| distributor-0 to retailer-0 | 2 |
| retailer-0 to supplier-0 | 2 |

Delivered materials by round:

| Round | Materials sent | Deliveries emitted |
| --- | ---: | ---: |
| 1 | 3 | 2 |
| 2 | 3 | 2 |
| 3 | 3 | 2 |
| 4 | 3 | 0 |
| 5 | 3 | 2 |

## Investigation

The surprising part was that a 10% message-drop setting produced 7 missing
deliveries out of 15 materials, while the aggregate `success_rate` was 0.8. I
checked the trace rather than only reading the metric. The simulator only
creates downstream sends after an upstream receive, so early drops reduce the
number of later opportunities. I also checked the validators and confirmed that
`supply_chain_pipeline` still passed for delivered items, while
`supply_chain_no_lost` failed because not every material resulted in a delivery.

## Next idea

I would build an acknowledgement and retry agent for staged workflows. It would
track item IDs across stages, request retransmission when a downstream handoff
is missing, and compare the reliability gain against extra message volume.
