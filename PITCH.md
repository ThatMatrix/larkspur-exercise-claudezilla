# PITCH.md

Six lines and a lever. Your words. The last two are scored.

Built: Multi-turn disruption agent connected to Larkspur's booking, flight-status, and policy tools, plus a next-available-day lookup served over a separate MCP server.
Does: Looks up the booking, reads live flight status, applies the disruption policy, and tells the stranded customer exactly what they are owed and what their options are — no human in the loop for the five in-scope shapes.
Number: 19 turns to resolve 5 disruption shapes (3–4 tool calls each, 61,406 tokens in total across all five); schema token tax 2,088 per turn on the 9-tool baseline, 2,880 per turn after MCP additions, both counted on the wire.
Guardrail: Out-of-scope cases (group of 12, refunds, minors) route to a human queue and never attempt a resolution — proved by G2HL9V escalating in 2 tool calls with no rebooking action taken.
Next: Add the tone intelligence lane so abusive messages are handled differently; add prompt caching to reduce the per-turn token cost on the schema context; expand the eval suite beyond the five Stage 1 shapes.
Still broken: An abusive or threatening message still receives a calm, normal disruption response. There is no tone gate on the way in. Build 4 is where that closes.
Lever: <cost | speed | intelligence>

## Priya asked

Costs:
Wrong:
Runs it:
Left out:
