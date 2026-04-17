# BugHound Mini Model Card (Reflection)

Fill this out after you run BugHound in **both** modes (Heuristic and Gemini).

---

## 1) What is this system?

**Name:** BugHound  
**Purpose:** Analyze a Python snippet, propose a fix, and run reliability checks before suggesting whether the fix should be auto-applied.

**Intended users:** Students learning agentic workflows and AI reliability concepts.

---

## 2) How does it work?

BugHound operates on a 5-step agentic loop:
1.  **PLAN:** Logs the intent to scan and fix code.
2.  **ANALYZE:** Detects vulnerabilities. In **Heuristic mode**, it uses regex and string matching to find known bad patterns. In **Gemini mode**, it prompts an LLM to return a JSON array of issues. (I also added a  skeptical fallback where deterministic rules override the LLM if the LLM claims there are zero issues but a hardcoded rule is violated).
3.  **ACT:** Generates a fix. Heuristic mode uses simple string replacement. Gemini mode prompts the LLM to rewrite the entire script based on the issues found.
4.  **TEST:** Passes the original code, fixed code, and issues through `risk_assessor.py`, which applies deterministic penalties to calculate a risk score (0-100).
5.  **REFLECT:** Evaluates the risk score. If the score is $\ge$ 75, it suggests auto-fixing; otherwise, it defers to a human.

---

## 3) Inputs and outputs

**Inputs:**
* Short Python snippets (functions, try/except blocks, toy scripts).
* Typically structured as raw strings of code passed into the `.run()` method.

**Outputs:**
* **Issues:** A normalized JSON-like array of dictionaries containing `type`, `severity`, and `msg`.
* **Fixed Code:** A string representing the proposed refactored Python code.
* **Risk Report:** A dictionary containing a numerical `score`, a categorical `level` (low, medium, high), an array of `reasons` for deductions, and a boolean `should_autofix`.

---

## 4) Reliability and safety rules

**Rule 1: Return Statement Removal Penalty (-30 points)**
* **What it checks:** Looks to see if the word "return" exists in the original code but is missing from the fixed code.
* **Why it matters:** Removing a return statement fundamentally alters the contract of a function, likely breaking whatever external code relies on it.
* **False Positive:** The LLM refactors code to remove an entirely redundant return statement (e.g., `return None` at the very end of a function).
* **False Negative:** The LLM changes `return True` to `return False`. The word "return" is still there, so the rule passes, but the logic is completely broken.

**Rule 2: Syntax Validity Check (Score = 0)**
* **What it checks:** Attempts to parse the fixed code using Python's built-in `compile()` function.
* **Why it matters:** If the LLM messes up indentation or forgets a parenthesis, the code will crash immediately. It is unsafe to deploy syntactically invalid code.
* **False Positive:** None. Compilation in Python is deterministic; if it fails to compile, it is objectively broken syntax.
* **False Negative:** The code compiles perfectly but throws a runtime error (like a `TypeError` or `NameError`) because the LLM hallucinated a variable that doesn't exist.

---

## 5) Observed failure modes

1.  **The "Lazy AI" Analysis:** BugHound passed a script with a highly dangerous bare `except:` block to the LLM. The LLM simply responded with `[]` (no issues found). Because the LLM successfully returned parseable JSON, the original BugHound blindly trusted it and ignored the heuristics that would have caught it.
2.  **The Uncompilable Hallucination:** The LLM proposed a fix for a function, but the markdown parser (`_strip_code_fences`) or the LLM itself dropped the necessary Python indentation. The risk assessor originally saw the code length was similar and the `return` statement was still there, so it gave it an 85 (auto-fix approved), even though the resulting code would instantly throw an `IndentationError`.

---

## 6) Heuristic vs Gemini comparison

* **What Gemini detected that heuristics did not:** Gemini successfully identified semantic bugs (like poor variable naming, logical edge cases, or inefficient loops) that regex cannot parse.
* **What heuristics caught consistently:** Heuristics never failed to catch rigid syntax patterns, like `print()` statements and `TODO` comments.
* **How the proposed fixes differed:** Heuristics produced surgical, minimal diffs (e.g., swapping `except:` for `except Exception as e:`). Gemini often rewrote the entire function, sometimes changing formatting, variable names, or the underlying logical approach.
* **Did the risk scorer agree with your intuition?** Initially, no. It was far too permissive, allowing code that was 49% deleted to pass with only a minor 20-point deduction. After adding the syntax compiler check, it became much more aligned with reality.

---

## 7) Human-in-the-loop decision

BugHound should refuse to auto-fix and require human review if it attempts to introduce new third-party dependencies to solve a simple problem.

* **Trigger:** Check if the substring `"import "` exists in the fixed code but not in the original code.
* **Where to implement:** In `reliability/risk_assessor.py`, as a structural change check. Subtract 15–20 points to knock it out of the "low risk" tier.
* **User Message:** `"Warning: The proposed fix introduces new dependencies/imports. Human review is required to verify these modules are necessary and secure."`

---

## 8) Improvement idea

Currently, BugHound penalizes code if the line count drops below 50%. This is crude and easily bypassed if the LLM hallucinated a lot of new, broken lines. A better guardrail is to use Python's built-in `ast` module to parse both files. If the `ast` detects that the original code had 3 function definitions, but the fixed code only has 2, BugHound should instantly flag it as High Risk for permanently deleting structural logic.