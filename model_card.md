# BugHound Mini Model Card (Reflection)

## 1) What is this system?

**Name:** BugHound

**Purpose:** BugHound takes a Python code snippet, detects common issues, proposes a minimal fix, and decides whether it is safe enough to apply automatically.

**Intended users:** Students learning how agentic AI workflows work and how to build reliability guardrails into AI systems.

## 2) How does it work?

BugHound runs five steps in order.

**Plan:** The agent logs that it is starting a scan and fix workflow.

**Analyze:** The agent tries to detect issues. If Gemini is enabled, it sends the code to the API and asks for a JSON list of issues. If the API returns something that cannot be parsed, or if the mode is set to heuristic only, the agent falls back to three pattern checks: bare `except:` blocks, `print()` calls in executable lines, and `# TODO` comments.

**Act:** The agent tries to produce a fixed version of the code. In Gemini mode, it sends the issues and the original code to the API and asks for a rewritten function. If the response is empty, missing the function definition, or still contains unfenced content, the agent falls back to a heuristic fixer that replaces bare except blocks and print calls using string substitution.

**Test:** The risk assessor scores the fix from 0 to 100. It deducts points for high or medium severity issues, for fixes that are much shorter than the original, and for fixes that remove return statements. It also adds a note if the bare except was modified.

**Reflect:** If the score is 75 or above and no medium or high severity issues were found, the agent marks the fix as safe to auto-apply. Otherwise, it recommends human review.

## 3) Inputs and outputs

**Inputs tested:**

- `cleanish.py`: a short two-line function using `logging.info` and returning a value. No issues expected.
- `flaky_try_except.py`: a file-reading function with a bare `except:` block and a file handle that is not closed safely.
- `mixed_issues.py`: a function with a `# TODO` comment, a `print()` call, and a bare `except:` block.
- A comment-only case: a function where `print(` appeared only inside comment lines, not in executable code.

**Outputs:**

- Issues detected ranged from "Reliability / High" for bare except blocks to "Code Quality / Low" for print statements and "Maintainability / Medium" for TODO comments. In Gemini mode, the agent also detected a resource leak caused by not using a `with` statement.
- Fixes included replacing `except:` with `except Exception as e:`, converting `print()` calls to `logging.info()`, and adding `import logging` at the top.
- Risk reports showed scores between 5 and 100. Auto-fix was approved only for the clean file and for files with no issues above Low severity.

## 4) Reliability and safety rules

**Rule 1: High severity issue detected (minus 40 points)**

This rule deducts 40 points for each High severity issue in the list. It exists because High severity issues like bare except blocks or resource leaks can hide bugs or cause data loss. A false positive could occur if the analyzer incorrectly labels a minor issue as High severity, which would block auto-fix for a safe change. A false negative could occur if a real High severity issue is labeled Medium or Low, allowing a risky fix to be auto-applied.

**Rule 2: Fixed code is much shorter than original (minus 20 points)**

This rule fires when the fixed code has fewer than half the lines of the original. It exists because a very short fix is likely incomplete, for example if the model returned only one line instead of the whole function. A false positive could occur if the fix is legitimately shorter because dead code was removed. A false negative could occur if a bad fix keeps the same number of lines but replaces meaningful logic with placeholder comments.

## 5) Observed failure modes

**Failure 1: The Gemini API call always returned empty output.**

The `generate_content` call was passing `{"role": "system", ...}` as part of the content list. Gemini does not support a system role in that format. The API threw an error every time, which was silently caught and returned as an empty string. The agent fell back to heuristics on every run without logging the real cause.

**Failure 2: The heuristic flagged print() inside a comment line.**

The check `"print(" in code` is a plain string search. When the input contained a comment like `# Old debug line: print("data")`, the heuristic flagged a Code Quality issue even though there was no real print call. The fixer then replaced `print(` everywhere in the file, including inside the comment and inside a docstring, corrupting the text. The risk score was 95 and auto-fix was approved, so the broken fix would have been applied automatically.

## 6) Heuristic vs Gemini comparison

**What Gemini caught that heuristics did not:**

Gemini detected a resource leak in `flaky_try_except.py`. It noted that the file handle `f` might not be closed if an exception occurs between `open()` and `f.close()`, and suggested using a `with` statement. Heuristics only checked for the bare except block and missed the file handling issue entirely.

**What heuristics caught consistently:**

Heuristics reliably found bare except blocks, print statements in executable code, and TODO comments across every run. They did not depend on an API call and produced the same result every time.

**How fixes differed:**

Heuristic fixes were minimal text substitutions. They changed `except:` to `except Exception as e:` and `print(` to `logging.info(`. Gemini fixes were more complete rewrites that restructured the function to use a `with` statement and added a comment about error handling.

**Did the risk scorer match intuition:**

For the two-issue Gemini result on `flaky_try_except.py`, the score was 15 and auto-fix was blocked. That felt right because both issues were High severity. For the heuristic result on the same file, the score was 55 because only one issue was found. That also felt appropriate. The mismatch is that the two modes produced different scores for the same input, which could confuse a user comparing runs.

## 7) Human-in-the-loop decision

**Scenario:** The Gemini fixer returns a rewritten function that is structurally similar to the original in length and contains a return statement, but it removes a line of logic that was not part of the issue.

The current risk scorer does not check whether specific logic was removed, only whether return statements are present and whether the fix is much shorter. A fix that silently drops a line of real code would score well and could be auto-applied.

**Trigger to add:** Check whether any non-whitespace lines from the original are missing in the fix, excluding the lines that were specifically identified as issues.

**Where to implement it:** In `assess_risk` in `reliability/risk_assessor.py`, as an additional structural check.

**Message to show:** "The proposed fix removed lines not related to the detected issues. Human review is required before applying."

## 8) Improvement idea

**Refine the `print(` heuristic to skip comment lines.**

The current check is `"print(" in code`, which is a raw string match. It fires on any line containing those characters, including comments and docstrings. This causes false positives and unsafe auto-fix of corrupted output.

The fix is one change in `_heuristic_analyze`: filter out comment-only lines before running the check.

```python
non_comment_code = "\n".join(
    line for line in code.splitlines()
    if not line.strip().startswith("#")
)
if "print(" in non_comment_code:
    ...
```

This change is small, auditable, and does not affect any other part of the workflow. It eliminates an entire class of false positives without adding complexity.
