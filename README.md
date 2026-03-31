# LogicShield

**Deterministic validation firewall for LLM outputs.**

LogicShield validates AI-generated proposals against ground-truth state using deterministic rules. Before any AI output can be acted upon, it must pass every rule you define. If it fails, you know exactly why.

**Zero dependencies. Pure Python. Works with any LLM, any domain.**

> LogicShield does not care what LLM you use, what domain you operate in, or what your data looks like. You define the rules. LogicShield enforces them.

---

## Why This Exists

LLMs are probabilistic. They hallucinate. In high-stakes domains (healthcare dosing, financial trading, industrial control, aerospace), a single hallucinated value can be catastrophic.

LogicShield solves this by ensuring that no LLM output reaches production without passing deterministic, mathematically verifiable validation. The LLM proposes. LogicShield verifies. Nothing executes until truth is confirmed.

---

## Install

```bash
pip install logicshield
```

No additional dependencies required.

---

## Quick Start

```python
from logicshield import LogicShield, Rule

# Define your validation rules
rules = [
    Rule("value_within_limit",
         lambda proposal, state: proposal["value"] <= state["max_value"],
         error="Value {proposal[value]} exceeds maximum {state[max_value]}"),
    Rule("value_positive",
         lambda proposal, state: proposal["value"] > 0,
         error="Value must be positive"),
    Rule.required("reason"),
    Rule.one_of("action", ["approve", "reject", "escalate"]),
]

shield = LogicShield(rules=rules)

# Ground-truth state (the Input Anchor)
state = {
    "max_value": 1000,
    "min_value": 1,
    "category": "standard",
}

# Validate a proposal from any source (LLM, user input, API, etc.)
result = shield.validate(
    proposal={"value": 500, "action": "approve", "reason": "Within normal range"},
    state=state,
)

print(result.valid)       # True
print(result.errors)      # []
print(result.state_hash)  # SHA-256 of the frozen state
```

**When validation fails:**

```python
result = shield.validate(
    proposal={"value": 5000, "action": "approve", "reason": "Override"},
    state=state,
)

print(result.valid)   # False
print(result.errors)  # ["Value 5000 exceeds maximum 1000"]

# Feedback Error Vector -- feed this back to your LLM for correction
print(result.feedback_vector)
# [SYSTEM FEEDBACK] Your proposal was REJECTED. Fix these errors:
#   1. Value 5000 exceeds maximum 1000
```

---

## Built-in Rule Helpers

| Helper | Description | Example |
|---|---|---|
| `Rule.required(key)` | Key must exist and be non-empty | `Rule.required("reason")` |
| `Rule.type_check(key, type)` | Value must be of type | `Rule.type_check("dose", float)` |
| `Rule.range(key, min, max)` | Value must be in range | `Rule.range("temp", 36.0, 42.0)` |
| `Rule.equals(key, state_key)` | Must equal state value | `Rule.equals("mode", "safety_mode")` |
| `Rule.less_than(key, state_key)` | Must be less than state value | `Rule.less_than("stop_loss", "price")` |
| `Rule.greater_than(key, state_key)` | Must be greater than state value | `Rule.greater_than("dose", "min_dose")` |
| `Rule.one_of(key, allowed)` | Must be one of allowed values | `Rule.one_of("action", ["BUY", "SELL"])` |
| `Rule.regex(key, pattern)` | Must match regex | `Rule.regex("code", r"^[A-Z]{3}$")` |

**Custom rules:**

```python
Rule("custom_check",
     lambda proposal, state: your_logic_here(proposal, state),
     error="Your error message with {proposal} and {state} interpolation")
```

---

## Immutable State

LogicShield freezes the ground-truth state at validation time. The state cannot be modified during or after validation, preventing tampering between input anchoring and output verification.

```python
from logicshield import ImmutableState

state = ImmutableState({"max_dose": 100})
state["max_dose"] = 999   # TypeError: ImmutableState cannot be modified.
state.max_dose = 999      # TypeError: ImmutableState cannot be modified.
del state["max_dose"]     # TypeError: ImmutableState cannot be modified.
```

---

## Verification Ledger

Every validation produces a SHA-256 state hash. Use `compute_signature()` to generate a cryptographic proof that a specific proposal was validated against a specific state.

```python
from logicshield import compute_signature

result = shield.validate(proposal, state)
if result.valid:
    signature = compute_signature(result.state_hash, proposal)
    # signature = SHA-256(state_hash + proposal_json)
    # Any third party can re-run validation to verify this.
```

---

## Feedback Error Vector

When validation fails, `result.feedback_vector` gives you a formatted error string ready to inject back into your LLM prompt. This enables a correction loop in your own code:

```python
prompt = base_prompt
for attempt in range(max_retries):
    response = your_llm_call(prompt)
    proposal = json.loads(response)
    result = shield.validate(proposal, state)
    if result.valid:
        break
    prompt = f"{base_prompt}\n\n{result.feedback_vector}"
```

LogicShield handles the validation. You handle the LLM. Clean separation.

---

## JSON Repair

LLMs often produce broken JSON. LogicShield includes a standalone repair utility:

```python
from logicshield import repair_json

data = repair_json('```json\n{"temp": 72.5,}\n```')
# Strips markdown fences, fixes trailing commas, handles single quotes
```

---

## Industry Examples

### Healthcare: Medication Dosage Verification

```python
shield = LogicShield(rules=[
    Rule("dose_within_max",
         lambda p, s: p["dose_mg"] <= s["max_dose_mg"],
         error="Dose {proposal[dose_mg]}mg exceeds patient max {state[max_dose_mg]}mg"),
    Rule("dose_above_min",
         lambda p, s: p["dose_mg"] >= s["min_effective_mg"],
         error="Dose {proposal[dose_mg]}mg below therapeutic minimum {state[min_effective_mg]}mg"),
    Rule("no_contraindication",
         lambda p, s: p["medication"] not in s["contraindications"],
         error="'{proposal[medication]}' is contraindicated for this patient"),
    Rule.one_of("route", ["oral", "iv", "subcutaneous", "intramuscular"]),
    Rule.required("clinical_reasoning"),
])

patient = {
    "max_dose_mg": 100,
    "min_effective_mg": 25,
    "contraindications": ["penicillin", "sulfonamides"],
}

# Valid prescription passes
result = shield.validate({
    "medication": "amoxicillin",
    "dose_mg": 50,
    "route": "oral",
    "clinical_reasoning": "Standard adult dose for mild infection",
}, patient)
assert result.valid

# Contraindicated medication is blocked
result = shield.validate({
    "medication": "penicillin",
    "dose_mg": 50,
    "route": "iv",
    "clinical_reasoning": "Broad spectrum coverage",
}, patient)
assert not result.valid  # "penicillin is contraindicated for this patient"
```

### Finance: Trading Signal Validation

```python
shield = LogicShield(rules=[
    Rule.one_of("action", ["BUY", "SELL", "HOLD"]),
    Rule("stop_loss_valid",
         lambda p, s: p["action"] != "BUY" or p["stop_loss"] < s["current_price"],
         error="Stop-loss must be below current price for BUY"),
    Rule("position_within_limit",
         lambda p, s: p["action"] == "HOLD" or p["position_pct"] <= s["max_position_pct"],
         error="Position {proposal[position_pct]}% exceeds max {state[max_position_pct]}%"),
    Rule("risk_reward_ratio",
         lambda p, s: p["action"] == "HOLD" or (
             abs(p["take_profit"] - s["current_price"]) /
             max(abs(s["current_price"] - p["stop_loss"]), 0.01) >= 2.0
         ),
         error="Risk-reward ratio below 2:1 minimum"),
    Rule.regex("ticker", r"^[A-Z]{1,5}$"),
])

market = {
    "current_price": 65000.0,
    "max_position_pct": 5.0,
}

# Stop-loss above entry price is blocked
result = shield.validate({
    "action": "BUY",
    "ticker": "BTC",
    "stop_loss": 66000.0,   # Above current price
    "take_profit": 70000.0,
    "position_pct": 2.0,
}, market)
assert not result.valid  # "Stop-loss must be below current price for BUY"
```

### Industrial Control: Reactor Safety

```python
shield = LogicShield(rules=[
    Rule.range("set_temp_c", min_val=-20, max_val=200),
    Rule("pressure_safe",
         lambda p, s: p["set_pressure_bar"] <= s["vessel_max_bar"],
         error="Pressure {proposal[set_pressure_bar]}bar exceeds vessel rating {state[vessel_max_bar]}bar"),
    Rule("flow_positive",
         lambda p, s: p["flow_rate_lpm"] > 0,
         error="Flow rate must be positive"),
    Rule("temp_step_limit",
         lambda p, s: abs(p["set_temp_c"] - s["current_temp_c"]) <= s["current_temp_c"] * 0.10,
         error="Temperature change exceeds 10% step limit"),
    Rule.one_of("mode", ["heating", "cooling", "standby"]),
])

reactor = {
    "current_temp_c": 150.0,
    "vessel_max_bar": 12.0,
}

# Overpressure is blocked
result = shield.validate({
    "set_temp_c": 155.0,
    "set_pressure_bar": 15.0,  # Exceeds 12 bar vessel rating
    "flow_rate_lpm": 50.0,
    "mode": "heating",
}, reactor)
assert not result.valid  # "Pressure 15.0bar exceeds vessel rating 12.0bar"
```

### Autonomous Agents: Action Permissions

```python
shield = LogicShield(rules=[
    Rule("action_permitted",
         lambda p, s: p["action"] in s["allowed_actions"],
         error="Action '{proposal[action]}' not in allowed set"),
    Rule("target_not_restricted",
         lambda p, s: not any(p.get("target", "").startswith(r) for r in s["restricted_paths"]),
         error="Target '{proposal[target]}' is in a restricted path"),
    Rule("within_budget",
         lambda p, s: p.get("estimated_cost_usd", 0) <= s["remaining_budget_usd"],
         error="Estimated cost exceeds budget"),
    Rule("confidence_sufficient",
         lambda p, s: p.get("confidence", 0) >= s["min_confidence"],
         error="Confidence {proposal[confidence]} below minimum {state[min_confidence]}"),
])

context = {
    "allowed_actions": ["read_file", "write_file", "search", "analyze"],
    "restricted_paths": ["/etc/", "/root/", "C:\\Windows\\"],
    "remaining_budget_usd": 5.00,
    "min_confidence": 0.7,
}

# Forbidden action is blocked
result = shield.validate({
    "action": "execute_shell",
    "target": "rm -rf /",
    "estimated_cost_usd": 0,
    "confidence": 0.99,
}, context)
assert not result.valid  # "Action 'execute_shell' not in allowed set"
```

### Content Moderation: Editorial Policy

```python
shield = LogicShield(rules=[
    Rule.required("title"),
    Rule.required("content"),
    Rule("content_length",
         lambda p, s: s["min_words"] <= len(p["content"].split()) <= s["max_words"],
         error="Content must be between {state[min_words]} and {state[max_words]} words"),
    Rule("no_banned_words",
         lambda p, s: not any(w in p["content"].lower() for w in s["banned_words"]),
         error="Content contains banned words"),
    Rule.one_of("category", ["news", "opinion", "tutorial", "review"]),
    Rule("news_has_sources",
         lambda p, s: p["category"] != "news" or len(p.get("sources", [])) >= 1,
         error="News articles must include at least one source"),
])

policy = {
    "min_words": 50,
    "max_words": 5000,
    "banned_words": ["hack", "exploit", "crack", "keygen"],
}

# News without sources is blocked
result = shield.validate({
    "title": "Breaking News",
    "content": " ".join(["Major technology announcement today."] * 15),
    "category": "news",
    "sources": [],
}, policy)
assert not result.valid  # "News articles must include at least one source"
```

---

## Ecosystem

| Product | What It Does |
|---|---|
| **[LogicShield](https://github.com/mattijsmoens/LogicShield)** | Deterministic validation firewall for LLM outputs |
| **[SovereignShield](https://github.com/mattijsmoens/Sovereign-Shield)** | 4-layer AI defense (input filter + action audit + ethical engine + LLM veto) |
| **[IntentShield](https://github.com/mattijsmoens/IntentShield)** | Lightweight action-gate for AI agents |

---

## License

Business Source License 1.1. See [LICENSE](LICENSE) for details.

Copyright (c) 2026 Mattijs Moens. All rights reserved.
