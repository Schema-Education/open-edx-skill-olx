---
name: skill-openedx-OLX
description: Author Open edX OLX problems — multiple choice, checkboxes, dropdown, numerical input, text input, formula/math, custom Python-graded, image-mapped, and the surrounding course-XML structure. Use this skill whenever the user mentions OLX, Open edX, edX Studio problems, the advanced problem editor, LON-CAPA, `<problem>` XML, `<multiplechoiceresponse>`, `<numericalresponse>`, `<customresponse>`, `<choiceresponse>`, `<optionresponse>`, `<formularesponse>`, `<stringresponse>`, `<imageresponse>`, `loncapa/python`, "Studio import", "edx-platform", course tarball exports, `course.xml`, `url_name`, or asks for help writing a graded problem for an Open edX course even if they don't explicitly say "OLX".
---

# Open edX OLX Problem Authoring

You are helping someone author problems for an Open edX course. Open edX courses are written in OLX (Open Learning XML) — an XML dialect with a fixed set of response-grading tags. Studio (the authoring UI) has a "simple/visual" editor and an "advanced" editor; this skill targets the **advanced editor** and the raw `problem/<url_name>.xml` files in a course export, because everything serious (partial credit, scripted graders, regex matching, targeted feedback, formula equivalence) requires dropping out of the simple editor.

Your job is to produce OLX that an author can paste into the advanced editor and have it work on the first try, without round-tripping through Studio to debug it.

## Workflow

1. **Identify the problem type.** Ask which kind of grading the author wants if it's not obvious. The mapping is: single answer from a list → `<multiplechoiceresponse>`; multiple answers from a list → `<choiceresponse>`; pick one from a dropdown → `<optionresponse>`; numeric value → `<numericalresponse>`; short text → `<stringresponse>`; algebraic/math expression → `<formularesponse>`; click on an image → `<imageresponse>`; anything else (per-student randomization, multi-step grading, custom validation) → `<customresponse>` with a `<script type="loncapa/python">` block.

2. **Start from `references/problem-types.md`.** That file has a copy-pasteable skeleton, the required and common-optional attributes, and the gotchas for every response type. Read the relevant subsection before writing OLX from scratch. The `templates/` directory holds the same skeletons as standalone `.xml` files when you need to hand the author one file.

3. **Fill in the content, then run the checklist in `references/gotchas.md` before returning the OLX.** The checklist catches the failures that show up most often in practice: missing tolerance on numerical problems, `<choicehint selected="…">` on the wrong response type, `def` lines not at column 0 inside `<script>`, unescaped `&`/`<`/`>` in attribute values, `shuffle="true"` placed on `<multiplechoiceresponse>` instead of on `<choicegroup>`, and so on.

4. **Tell the author where the OLX goes.** If they're using Studio: open the problem component, click **Edit → Advanced Editor → Settings → Edit**, paste the OLX, save. If they're editing a course export tarball: the file is `course/problem/<url_name>.xml`, and a `<problem url_name="<url_name>"/>` reference must exist in the parent `course/vertical/<unit_url_name>.xml`. The `<url_name>` value must equal the file basename and only contain `[a-zA-Z0-9_-]`. See `references/course-structure.md` for the full tree.

## What you should always include

Every problem you produce should have a `<label>` (the question prompt — required for accessibility and screen readers), at least one form of feedback (either per-choice `<choicehint>`/`<choicehint selected>`, `<correcthint>` on the response, or `<stringequalhint>` for common wrong text answers), and a `<solution>` block with the explanation — even one-line. The solution is what learners see when they click "Show Answer", and authors who skip it almost always regret it later when learners ask for one in the forum.

Set `display_name`, set `max_attempts` to a sensible number (3 is the common default for graded work; omit for ungraded practice), and set `showanswer="finished"` (or `"past_due"` for graded exams) so learners can study correct answers after they're done. If the problem uses scripted randomization, set `rerandomize="per_student"` — the alternatives reseed too often and confuse learners who refresh the page.

## When to stay in OLX vs. recommend the simple editor

If the author only needs a vanilla multiple-choice or numerical problem with no per-choice feedback, no partial credit, no scripting, and no regex — recommend they use Studio's simple editor with its Markdown shorthand. It round-trips cleanly and is friendlier to non-technical co-authors. The moment they want **any** of: `<script>`, partial credit, regex matching, targeted feedback, additional answers, image-response, formula response, or per-choice feedback richer than "right/wrong" — produce raw OLX and tell them to use the advanced editor. The Studio simple editor cannot re-open a problem that has any of those features; once you save with them, the visual editor button is gone. Warn the author about this one-way door the first time it comes up in a session.

## Custom Python graders (the highest-value, highest-risk case)

`<customresponse>` with a Python check function is what authors reach for when nothing else fits — multi-step grading, per-student randomization, math-equivalence checks the formula response can't do. Get this right and the author can express almost any grading logic; get it wrong and they get cryptic CodeJail errors and silent zero-score bugs.

The full pattern is in `references/customresponse-python.md`. The core constraints to remember without re-reading:

- The Python script runs in **CodeJail**, an AppArmor-sandboxed interpreter. Allowed: `math`, `random`, `numpy`, `sympy`, plus the LON-CAPA helpers. **Not** allowed: filesystem, network, `os`, `subprocess`, `requests`, `matplotlib` (will fail with "Cannot create LoncapaProblem block"), and anything else not on the host's CodeJail allowlist.
- The `def` line of the check function **must start at column 0** of the `<script>` block. This is the single most common copy-paste bug.
- The check function signature is `def f(expect, answer)`. `answer` is a string for single-input problems and a list of strings for multi-input. Returning `None` is treated as `False`.
- `anonymous_student_id` is available **inside the `<script>` block at problem-render time**, but **not inside the `cfn` body at grading time**. To carry it through, seed `random` with it at script load and store any derived values in module-level variables, then read those variables from the `cfn` body. The provided template does exactly this.
- For partial credit on a single input, return `{'ok': 'Partial', 'msg': '...', 'grade_decimal': 0.5}`. For per-input grading on multi-input problems, return `{'input_list': [{'ok': ..., 'msg': ..., 'grade_decimal': ...}, ...]}`.

## Reference files

Read these as needed — they exist so this top-level file can stay short:

- `references/problem-types.md` — one section per response tag, with skeleton XML, attributes, feedback patterns, and gotchas. Start here when you know the type.
- `references/customresponse-python.md` — full template, CodeJail allowlist, common patterns (per-student randomness, multi-input scoring, math equivalence via SymPy), and debugging tips.
- `references/course-structure.md` — where `problem/<url_name>.xml` lives in a course export, how it's referenced from verticals, and the difference between pointer-syntax and inline-syntax problems (and why pointer syntax is safer for built-in `<problem>` blocks).
- `references/gotchas.md` — pre-return checklist. Run through it mentally (or literally) before handing OLX back to the author.

## Templates

The `templates/` directory mirrors `references/problem-types.md` but as standalone XML files the author can copy verbatim:

- `multiple-choice.xml`, `checkbox.xml`, `dropdown.xml`
- `numerical.xml`, `numerical-partial-credit.xml`
- `text-input.xml`, `text-input-regex.xml`
- `formula.xml`
- `customresponse.xml`, `customresponse-multi-input.xml`
- `image-response.xml`

Prefer adapting a template over composing OLX from memory — the templates have the gotcha-mitigations (tolerance set, `shuffle` in the right place, `def` at column 0, `<label>` and `<solution>` present) already baked in.

## Output format

When the author asks for a problem, return the complete `<problem>…</problem>` element ready to paste. If they're editing a course export, also include the suggested `url_name` and a one-line reminder of which `vertical/*.xml` needs to reference it. Do not return a partial fragment that omits the wrapping `<problem>` tag — authors paste exactly what you return, and a missing wrapper produces an unhelpful Studio import error.

If the author has given you an existing problem to modify, return the modified `<problem>` in full rather than a diff — Studio's advanced editor takes a full document, not a patch.

Briefly explain (one or two sentences) any non-obvious choice you made: tolerance value picked, regex you wrote, why you chose `<customresponse>` over `<formularesponse>`. The author often wants to tweak these and skipping the rationale forces them back to the documentation.
