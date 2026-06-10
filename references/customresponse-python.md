# `<customresponse>` with Python check functions

This is the most powerful and the most footgun-laden OLX response type. It lets you write a Python function that grades arbitrary input. The function runs inside CodeJail, a sandboxed Python interpreter — the sandbox is what makes this safe to run user-graded code on a shared platform, and it's also what trips most authors up.

## The full pattern (single input)

```xml
<problem display_name="Even number">
  <script type="loncapa/python">
import random

def check_even(expect, ans):
    try:
        n = int(ans)
    except ValueError:
        return {'ok': False, 'msg': 'Please enter an integer.'}
    if n % 2 == 0:
        return {'ok': True, 'msg': f'{n} is even.'}
    return {'ok': False, 'msg': f'{n} is odd.'}
  </script>

  <p>Enter any even number:</p>
  <customresponse cfn="check_even" expect="even">
    <textline size="10"/>
  </customresponse>

  <solution>
    <div class="detailed-solution">
      <p>Any even integer works (e.g., 2, 4, -6).</p>
    </div>
  </solution>
</problem>
```

## Function contract

- Signature: `def check_function(expect, answer)`.
  - `expect` is the value of the `expect=` attribute on `<customresponse>` (a string). Use this to pass author-defined data into the grader rather than hardcoding it inside the script.
  - `answer` is a **string** for single-input problems and a **list of strings** for multi-input problems.
- The `def` line **must start at column 0**. The most common reason a check function "isn't called" is that someone indented it by accident inside the `<script>` tag.
- Returning `None` is treated as `False`. Returning nothing (just a bare `return`) likewise gives the learner zero with no message.
- Returning a `bool` is the simplest case but provides no feedback. Returning a dict is almost always better.

## Return value forms

**Boolean / coarse:**
```python
return True            # full credit, no message
return False           # zero, no message
return 'Partial'       # 50% credit, no message
```

**Dict with feedback (single input):**
```python
return {
    'ok': True,                        # True | False | 'Partial'
    'msg': '<p>Nicely done — even number.</p>',  # XHTML, rendered to the learner
    'grade_decimal': 1.0,              # optional; lets you set arbitrary partial credit
}
```

The `msg` is parsed as XHTML — you can include `<p>`, `<code>`, `<em>`, etc., but unclosed tags will cause the grade to silently fail to render. When in doubt, wrap the whole message in `<p>...</p>`.

**Dict with per-input feedback (multi-input):**
```python
return {
    'overall_message': '<p>Checking your answers…</p>',
    'input_list': [
        {'ok': True,  'msg': 'x is correct.',   'grade_decimal': 1.0},
        {'ok': False, 'msg': 'y should be 5.',  'grade_decimal': 0.0},
        {'ok': 'Partial', 'msg': 'z is close.', 'grade_decimal': 0.5},
    ],
}
```

The `input_list` must have one entry per `<textline>` (or other input) inside the `<customresponse>`, in document order.

## CodeJail allowlist (what you can `import`)

Reliably available everywhere:
- `math`
- `random`
- `numpy`
- `sympy`
- `scipy` (sometimes, depending on host config — don't rely on it for portability)
- LON-CAPA helpers: `chemcalc`, `chemtools`, `eia`, `inverter` (rarely needed)

**Reliably not available** (and the reason most "Cannot create LoncapaProblem block" errors happen):
- `matplotlib`, `pandas` — heavyweight; deliberately excluded.
- `os`, `subprocess`, `sys`, `shutil`, `pathlib` — sandboxed away.
- `requests`, `urllib`, `socket`, anything that does network I/O.
- File I/O of any kind. There is no filesystem inside CodeJail.
- `importlib`, dynamic `__import__`.

If the author wants a grader that needs anything off this list, they cannot use `<customresponse>` — they need to either reformulate using `<formularesponse>` (for math) or move the grading off-platform (an LTI tool, an external grader XBlock). Tell them this directly rather than producing something that will silently fail.

## Per-student randomization

The single most common pattern: each learner should see a different problem instance. The mechanism is `anonymous_student_id`, a stable opaque string available **inside the `<script>` block at problem-render time**. It is **not** in scope inside the `cfn` body when the learner clicks Submit — by then, the script has finished. To carry per-student state through, seed `random` at script-load and store derived values in module-level variables:

```xml
<script type="loncapa/python">
import random

# anonymous_student_id is a stable per-learner string. Seeding random with it
# guarantees the same learner sees the same problem variant across page reloads,
# while different learners see different variants. Without this seeding, the
# variant changes every render and Submit grades against whatever was last shown.
random.seed(anonymous_student_id)

# Module-level state — accessible from check_answer below because the script
# block has already executed by the time grading runs.
target_number = random.randint(10, 99)
target_squared = target_number ** 2

def check_answer(expect, ans):
    try:
        value = int(ans)
    except ValueError:
        return {'ok': False, 'msg': 'Please enter an integer.'}
    if value == target_squared:
        return {'ok': True, 'msg': f'Correct — {target_number}² = {target_squared}.'}
    return {'ok': False, 'msg': f'Not quite — try squaring {target_number}.'}
</script>

<p>What is the square of $target_number?</p>
<customresponse cfn="check_answer">
  <textline size="10"/>
</customresponse>
```

A few things to note:

- The `$target_number` template substitution inside the prompt — Python variables from the `<script>` block are interpolated into the problem text using `$name`.
- `random.seed(anonymous_student_id)` is OK even though it's a string — Python's `random.seed` accepts strings and hashes them.
- The platform also accepts `rerandomize="per_student"` on the `<problem>` element, which tells the platform to keep the seed stable across that learner's sessions. Pair the two: seed with `anonymous_student_id`, and set `rerandomize="per_student"` on the wrapping `<problem>`.
- **20-seed cap:** the platform caps available random seeds per problem at 20, which surprises authors who set `rerandomize="always"` expecting infinite variants. For truly unique-per-attempt variants, use a `library_content` randomized pool instead of in-problem randomization.

## Math equivalence with SymPy

When `<formularesponse>` isn't enough (e.g., the answer involves logical conditions or non-numeric values), use SymPy in a customresponse:

```xml
<script type="loncapa/python">
import sympy
from sympy.parsing.sympy_parser import parse_expr

correct = parse_expr("(x+1)*(x-1)")

def check_expr(expect, ans):
    try:
        learner = parse_expr(ans)
    except (SyntaxError, TypeError):
        return {'ok': False, 'msg': 'Could not parse your expression.'}
    if sympy.simplify(learner - correct) == 0:
        return {'ok': True, 'msg': 'Algebraically equivalent to the expected answer.'}
    return {'ok': False, 'msg': 'That expression is not equivalent.'}
</script>

<p>Expand $(x+1)(x-1)$.</p>
<customresponse cfn="check_expr">
  <textline size="40"/>
</customresponse>
```

`parse_expr` from SymPy is the safer option here than `eval` — it parses math syntax without executing arbitrary Python. Never use bare `eval(ans)` in a customresponse; the sandbox blocks the worst of it, but it also blocks useful imports and makes errors hard to diagnose.

## Multi-input pattern

```xml
<script type="loncapa/python">
def check_pair(expect, ans):
    x, y = ans  # ans is a list because there are two textlines below
    x_ok = x == '2'
    if y == '5':
        y_status, y_msg, y_grade = True, 'y is correct.', 1.0
    elif y in ('3', '4', '6', '7'):
        y_status, y_msg, y_grade = 'Partial', 'Close — half credit.', 0.5
    else:
        y_status, y_msg, y_grade = False, 'y should be 5.', 0.0
    return {
        'overall_message': '<p>Both x and y are graded.</p>',
        'input_list': [
            {'ok': x_ok, 'msg': 'x must be 2.' if not x_ok else 'x is correct.',
             'grade_decimal': 1.0 if x_ok else 0.0},
            {'ok': y_status, 'msg': y_msg, 'grade_decimal': y_grade},
        ],
    }
</script>

<customresponse cfn="check_pair">
  <p>Solve: <textline label="x" size="6"/> and <textline label="y" size="6"/></p>
</customresponse>
```

`label="x"` on each `<textline>` lets the screen reader read the input's name. `input_list` entries map to inputs in document order.

## Debugging

- `print()` output from the script is invisible — there's no console. Return debug info inside `msg` instead during development, then remove it.
- If the problem fails to load with "Cannot create LoncapaProblem block", the script is broken (syntax error, banned import, indentation issue). Look at the `<script>` block, not the response element. Common causes: indented `def`, banned import, missing `import` for something used, unbalanced quotes in `f"…"` strings.
- If the function loads but always returns zero, check that `cfn=` matches the function name exactly (case-sensitive) and that the function is at the top level of the script (not nested inside another `def`).
- If `anonymous_student_id`-seeded randomness produces the same variant for everyone, the script is running before the platform injects the variable. Make sure `anonymous_student_id` is referenced inside the script body, not at import time of a helper module.

## When not to use customresponse

Reach for `<customresponse>` when nothing else fits. Don't reach for it as the default — the other response types have better Studio integration, are easier to debug, and don't require sandboxing. In particular:

- Numeric answer with tolerance → `<numericalresponse>`.
- Algebraic-equivalence math answer → `<formularesponse>`.
- One of N strings → `<stringresponse>` with `<additional_answer>`.
- Regex on input → `<stringresponse type="regexp">`.
- "Right answer is one of these N choices" → multiple choice / checkboxes.

Use `<customresponse>` for: per-student randomization, multi-step validation, custom math (matrix problems, code-output validation), grading that depends on multiple inputs together, or anything where you genuinely need to write logic.
