# BugHound Mini Model Card (Reflection)

## 1) What is this system?

**Name:** BugHound  
**Purpose:** Analyze a Python snippet, report likely issues, propose a rewrite, and estimate whether the rewrite is safe enough to auto-apply.

**Intended users:** Students learning agentic workflows and AI reliability. It is a small teaching prototype, not a production code reviewer or a substitute for tests and human review.

## 2) How does it work?

BugHound records a five-stage loop:

1. **PLAN:** Log that it will scan and propose a fix.
2. **ANALYZE:** With `client=None`, run simple pattern checks for `print(`, bare `except:`, and `TODO`. With a Gemini client, ask the model for a JSON array of issues, each with a non-empty `type`, a `severity` of `Low`, `Medium`, or `High`, and a non-empty `msg`. Invalid output falls back to heuristics.
3. **ACT:** If there are findings, ask Gemini for a rewritten file when a Gemini client is available; otherwise use limited regex/string-based edits. Empty Gemini output falls back to those heuristic edits. With no findings, return the input unchanged.
4. **TEST:** The risk assessor scores issue severities and a few structural changes. It now also penalizes unchanged output when issues were reported.
5. **REFLECT:** Report whether the risk policy permits auto-apply or recommends human review.

There is an integration caveat: the Streamlit app currently assigns `MockClient` in “Heuristic only” mode. Because the agent treats any object with a `complete` method as an LLM client, that mode can enter the LLM branches and fall back after mock responses. Direct `client=None` runs exercise the true heuristic path and produce the expected offline trace.

## 3) Inputs and outputs

Inputs were short Python functions and a comments-only snippet:

- **`sample_code/cleanish.py`:** A small `add` function using logging. No issues; code stayed unchanged; score 100, Low, auto-fix allowed (a no-op decision).
- **`sample_code/mixed_issues.py`:** A function with a TODO, `print`, division, and bare `except:`. Heuristics found Code Quality/Low, Reliability/High, and Maintainability/Medium. The proposed heuristic rewrite added logging and changed the exception clause, leaving the TODO. Score 30, High, auto-fix denied.
- **`sample_code/flaky_try_except.py`:** A file-reading function with a bare `except:`. Heuristics found one High Reliability issue and narrowed the clause to `except Exception as e:` with a placeholder comment. Score 55, Medium, auto-fix denied.
- **Comments-only TODO:** Heuristics found a Medium Maintainability issue but had no applicable edit. Before the no-op guardrail, that unchanged result scored 80, Low. After the guardrail it scores 60, Medium, with auto-fix denied.

Two live Gemini-mode runs were attempted for `mixed_issues.py` and `flaky_try_except.py`. The analyzer output failed the required issue schema in both; the fixer returned empty output in both. The runs therefore used heuristic analysis and fixes. Their reports matched the corresponding heuristic findings above (scores 30 and 55). No distinct Gemini-only finding or valid Gemini-generated fix was observed.

## 4) Reliability and safety rules

1. **Severity deductions:** High issues subtract 40 points, Medium 20, and Low 5. This makes severe findings more likely to require review. A false positive can occur if an analyzer overstates severity and blocks a safe edit; a false negative can occur if a real issue is assigned a lower severity or an unrecognized severity string, which currently earns no deduction.
2. **Structural checks:** Removing all `return` statements when the original contains `return` subtracts 30 points; changing a bare `except:` subtracts 5. These catch broad behavior changes and encourage review of exception handling. A false positive can occur when a return is intentionally removed or a bare handler is safely narrowed; a false negative can occur when the return value changes but a `return` remains, or when a broad handler is rewritten as `except Exception` and still swallows errors.
3. **Unchanged output with findings:** If issues exist but the proposed code is unchanged, subtract 20 and report that no change was proposed. This raises the comments-only TODO example from Low/80 to Medium/60. A false positive is possible when the correct response to an ambiguous finding is to leave code untouched; a false negative is possible when the code changes but does not actually resolve the reported issue.
4. **Auto-fix threshold:** The score must be at least 90 to permit auto-apply. This is more cautious than treating every Low risk score (75+) as eligible. It can send safe changes to human review; it can still miss risks the scoring rules do not recognize.

These rules are simple signals, not semantic validation or execution-based tests of a proposed patch.

## 5) Observed failure modes

1. **Gemini output did not meet the analyzer contract.** On both live sample runs, the analyzer result failed schema validation; the fixer returned an empty response. BugHound fell back to pattern-based findings and edits. This kept the workflow running, but it means those runs did not establish that Gemini analysis or fixing worked, and heuristic fallback can miss issues outside its few patterns.
2. **The Streamlit heuristic mode can misroute through `MockClient`.** The mock has a `complete` method, so the agent logs LLM paths. Its analyzer prose is rejected and falls back, while its non-empty fixer placeholder can be accepted as rewritten code. This can produce an invalid “fix” and a severe risk score for the wrong reason. The issue is in the app’s client selection/agent capability check.
3. **A finding could receive no edit but still look Low risk.** The comments-only TODO was detected, but the heuristic fixer left it unchanged. Before the no-op penalty, the risk score was 80 (Low). The new guardrail and test now raise that case to Medium and keep auto-fix off.

## 6) Heuristic vs Gemini comparison

In direct offline runs, heuristics consistently found the tested `print(`, bare `except:`, and `TODO` patterns. Their fixes are narrow string/regex edits, not general Python repairs. The live Gemini calls did not yield valid analyzer findings or a non-empty fix in the runs observed, so no reliable comparison of Gemini’s detection quality or unique findings is available. The observed distinction was operational: Gemini mode attempted model calls and then fell back; direct heuristic mode needed no network and produced deterministic pattern-based results. Risk review rejected the mixed and file-reading examples, while cleanish code stayed unchanged and received a Low score.

## 7) Human-in-the-loop decision

If BugHound reports an issue but returns code identical to the input, it should require human review because the finding was not addressed. The trigger belongs in `reliability/risk_assessor.py`, where unchanged output with findings is penalized and auto-fix is denied under the current score policy. The UI should show the risk reason and a clear message such as: **“BugHound found issues but did not change the code. Review the findings and decide what to do next.”**

## 8) Improvement idea

Fix the Streamlit heuristic-mode wiring: use `client=None` for heuristic-only mode, or make the mock explicitly identify itself as non-LLM. Add an offline workflow test that selects the heuristic path and asserts the trace says “Using heuristic analyzer/fixer (offline mode)” and never “Using LLM”. This is a small change with a measurable result and prevents the UI from presenting a mock response as a real fix.
