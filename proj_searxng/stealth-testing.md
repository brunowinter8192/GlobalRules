# Stealth Testing Methodology

Applies to: `dev/search_pipeline/engines_eval/` — all browser-based engine testing.

## Core Principle

We are testing DETECTION, not PERFORMANCE. Every test measures: "At which query does the engine detect us, and HOW?" Delays, cooldowns, and rate-limit avoidance defeat the purpose of the test.

## Network Context Awareness (BEFORE first stress-run)

BEFORE running stress-tests, benchmarks, or back-to-back loops that repeatedly hit an external service (search engine, API, public endpoint), ask the user about the network context:

- Home / residential IP
- Public library / coffeeshop / coworking WLAN (shared NAT)
- Mobile tethering
- VPN / Tor
- Cloud / VPS / datacenter IP
- Corporate / university network (shared NAT, may have DPI)

Shared-NAT contexts (public library, coworking, university, coffeeshop) carry two distinct costs not visible from the scraping code:

1. **Ethical cost:** the test burns the service's per-IP reputation score for everyone sharing that NAT. Other users at the same location may start getting CAPTCHAs, rate-limits, or blocks caused by the test traffic, without any indication that we are responsible.
2. **Test-validity cost:** baseline measurements are not reproducible. Other concurrent users at the same NAT influence the same IP's score — a "clean baseline" from a library WLAN is not a baseline of our code, it's a baseline of "our code + whatever else was happening on that IP".

When context is shared-NAT: either defer volume-heavy tests to a non-shared context, OR proceed with explicit user acknowledgment that baseline is noisy and other users are affected.

## Single-Variable Testing Protocol

### Per Test Run
1. Change ONE config variable (stealth_config.py or engine_selectors.py)
2. Run `28_stress_test.py --engine <name> --limit 30` — NO delays, NO between_query_delay
3. Note: X/30 success, first failure at query #N
4. **IMMEDIATELY at failure:** take screenshot + check page title + HTTP status
5. Record which detection layer blocked (see `decisions/stealth01_detection_layers.md`)
6. REVERT the config change before next test

### Between Test Runs
- Do NOT wait for IP cooldowns. If the IP is blocked from the previous run, that IS data.
- If a fresh run starts at 0/30 because IP is still blocked: that tells us the previous patch did NOT prevent IP-level blocking. Record it.
- If you need a clean baseline: state "IP was pre-blocked from previous run" in the report. Do NOT silently wait and pretend it's a clean test.

### Failure Diagnosis (MANDATORY at each failure point)
When queries start failing:
1. `27_stealth_test.py <engine> "<query>" --screenshot` — see what the page shows
2. Check page title: "PoW Captcha", "Captcha", HTTP error, empty results?
3. Map to detection layer: Fingerprint? Rate-limit? Session? IP-block?
4. Record in report: "Query N failed → [detection layer] → [evidence: screenshot/title/status]"

### Report Format
After all tests, produce a summary table:

```
| Test | Config Change | X/30 | First Fail | Detection Layer | Evidence |
```

The "Detection Layer" and "Evidence" columns are NON-NEGOTIABLE. Without them, the test is useless — we know it failed but not WHY.

## What NOT To Do

- NO `sleep 300` between test runs
- NO `between_query_delay` in test runs (we WANT to hit limits)
- NO reverting engine config to "safe" defaults after testing (proxy, settle_seconds — leave as the test requires)
- NO claiming "works" based on 5 queries — always 30
- NO running tests without screenshot at failure point
- NO adding rate-limiting infrastructure during stealth testing

## Config Combination Rules

When testing patches in combination:
- NEVER combine a patch that helps (e.g., webgl +7) with one that hurts (e.g., permissions -3). The bad patch dominates.
- First: test each patch individually. Then: combine ONLY the ones that individually helped.
- If combined result < best individual result → one patch interferes with another. Test which one.
