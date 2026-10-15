# Team JSON Contracts (v1.0)
---

## 0. The 5 rules

1. Every message is plain JSON (a Python `dict`).
2. Field names are `snake_case`.
3. Missing value = `null` (never leave a field out).
4. Confidence is a number from `0.0` to `1.0`.
5. Locators are always written as `kind=value`: `id=loginBtn`, `css=.btn`, `xpath=//a[1]`, `name=email`.

---

## 1. Steps  (Script Generator → Engine)

A test is just a list of steps.

```json
[
  {"action": "navigate", "url": "http://localhost:8000/v1/login.html"},
  {"action": "type",  "locator": "id=username", "value": "testuser"},
  {"action": "click", "locator": "id=loginBtn"},
  {"action": "assert_title_contains", "value": "Dashboard"}
]
```

| action | needs |
|---|---|
| `navigate` | `url` |
| `type` | `locator`, `value` |
| `click` | `locator` |
| `assert_title_contains` | `value` |
| `assert_text_present` | `value` |

Python signature: `generate_steps(instruction: str, url: str, page_html: str) -> list[dict]`

---

## 2. Healing request  (Engine → Self-Healing)

"This locator broke. Here is what it looked like before, and everything on the page now."

```json
{
  "test_name": "login_test",
  "page_url": "http://localhost:8000/v2/login.html",
  "broken_locator": "id=loginBtn",
  "old_element": {
    "tag": "button", "id": "loginBtn", "name": null, "classes": "",
    "text": "Log In", "placeholder": null, "type": null,
    "x": 8, "y": 120, "w": 60, "h": 22,
    "parent_tag": "body", "parent_id": null
  },
  "candidates": [ { "...same fields as old_element..." } ]
}
```

## 3. Healing response  (Self-Healing → Engine)

```json
{
  "healed_locator": "id=submit-auth-btn",
  "confidence": 0.94,
  "reasons": ["same tag", "text similarity 0.62", "position within 3 px"]
}
```

- Not sure? Return `"healed_locator": null` and a low confidence. **Never guess.**
- The engine only accepts a fix when `confidence >= 0.80` (configurable).

Python signature: `heal(request: dict) -> dict`

---

## 4. Visual check  (Engine → Visual)

```json
{ "baseline_path": "runs/a/base.png", "current_path": "runs/b/now.png",
  "ignore_regions": [ {"x": 0, "y": 0, "w": 200, "h": 40} ] }
```

Returns:

```json
{ "changed": true, "score": 0.81, "diff_image_path": "runs/b/diff.png",
  "regions": [ {"x": 8, "y": 118, "w": 70, "h": 26, "severity": "high"} ] }
```

`severity` is one of `low`, `medium`, `high`.
Python signature: `visual_check(request: dict) -> dict`

---

## 5. Flaky score  (Engine → Flaky)

```json
{ "test_name": "login_test",
  "history": [ {"status": "FAILED", "duration_s": 9.8, "error_type": "timeout"},
               {"status": "PASSED", "duration_s": 3.1, "error_type": null} ] }
```

Returns:

```json
{ "flaky_score": 0.78, "label": "FLAKY",
  "reason": "6 of 20 runs timed out; passes on retry" }
```

`label` is one of `FLAKY`, `STABLE`, `INSUFFICIENT_DATA` (fewer than 5 runs).
Python signature: `flaky_score(request: dict) -> dict`

---

## 6. Root cause  (Engine → Root-Cause)

```json
{ "test_name": "login_test",
  "error_message": "Element not found: id=loginBtn",
  "failed_step": {"action": "click", "locator": "id=loginBtn"},
  "healing": {"attempted": true, "healed": true, "confidence": 0.94},
  "flaky_score": 0.05,
  "visual_changed": false }
```

Returns:

```json
{ "category": "HEALED_LOCATOR_CHANGE", "confidence": 0.96,
  "explanation": "The button id changed; self-healing found the new button." }
```

`category` is one of:

| category | meaning |
|---|---|
| `HEALED_LOCATOR_CHANGE` | UI changed, self-healing fixed it |
| `HEALING_FAILED` | UI changed, no confident fix |
| `REAL_BUG` | elements found, but the result is wrong |
| `FLAKY_TIMING` | random timing or loading problem |
| `ENVIRONMENT_ISSUE` | server down, network, browser problem |

Python signature: `classify_failure(request: dict) -> dict`

---

## 7. Run result  (Engine → Backend)  `POST /api/runs`

```json
{ "run_id": "20261105-101500-ab12cd", "schema_version": 1,
  "test_name": "login_test",
  "status": "HEALED",
  "duration_s": 12.4,
  "steps": [ {"index": 0, "step": {}, "status": "PASSED", "duration_s": 0.4,
              "error": null, "screenshot": null} ],
  "healing_events": [ {"step_index": 3, "broken_locator": "id=loginBtn",
                       "suggested_locator": "id=submit-auth-btn", "confidence": 0.94,
                       "healed": true, "screenshot": "runs/.../step3.png"} ],
  "visual": null, "flaky": null, "root_cause": null }
```

`status` is one of `PASSED`, `HEALED`, `FAILED`. Step `status` also allows `HEALED`.
`visual`, `flaky`, `root_cause` hold the outputs of sections 4, 5, 6 (or `null` if not run yet).

---

## 8. Backend endpoints

| Method | Path | What it does |
|---|---|---|
| POST | `/api/runs` | save a run |
| GET | `/api/runs` | list runs |
| GET | `/api/runs/{id}` | one run with steps and screenshots |
| GET | `/api/stats` | heal rate, wrong-heal rate, counts |
| POST | `/api/jobs` | user starts a test `{"url": "...", "instruction": "..."}` |
| GET | `/api/jobs/next` | the worker asks "anything to run?" |
| POST | `/api/visual-diffs/{id}/review` | `{"decision": "accept" or "reject", "comment": "..."}` |
| GET | `/api/health` | returns `{"status": "ok"}` |


