# Pre-return checklist

Run through this before handing OLX back to the author. Most of these are silent failures — the problem will render in Studio, but it will grade wrong, or it will look fine to you and lock the author out of the visual editor without warning.

## Structural

- [ ] The output is wrapped in `<problem>…</problem>` — not just a bare response element. Studio's advanced editor expects the full document.
- [ ] `display_name` is set.
- [ ] If the author is editing a course export, `url_name` is set and matches the file basename, using only `[a-zA-Z0-9_-]`.
- [ ] Every response element has a `<label>` child. (Required for accessibility — screen readers use it as the input prompt. Omitted labels also produce a Studio warning.)
- [ ] At least one `<solution>` exists per problem (even a one-line explanation). The wrapper must be `<solution><div class="detailed-solution">…</div></solution>` — without the `div.detailed-solution` the Show Answer styling breaks.

## Numerical input

- [ ] `<responseparam type="tolerance" default="…">` is present unless the answer is an exact integer. Without tolerance, `3.14` does not equal `3.140000001` and learners get zero.
- [ ] Tolerance is a percent (`5%`) for values that span many magnitudes; absolute (`0.01`) when the scale is fixed.
- [ ] The input element is `<formulaequationinput/>` (renders MathJax live). `<textline math="1"/>` is deprecated.
- [ ] If using `trailing_text="m/s"` for units display, the author understands the unit is **not** graded — only the number is checked.

## Multiple choice (`<multiplechoiceresponse>`)

- [ ] Exactly one `<choice correct="true">` — or `correct="partial"` with `point_value` set and `partial_credit="points"` on the response element.
- [ ] `shuffle="true"` is on `<choicegroup>`, **not** on `<multiplechoiceresponse>`. (Common typo: shuffle on the wrong tag silently does nothing.)
- [ ] `<choicehint>` inside `<choice>` does **not** have a `selected` attribute. (That attribute belongs to `<choiceresponse>`.)
- [ ] If using `fixed="true"` to pin "None of the above"-style choices, they're at the end of the list.

## Multi-select (`<choiceresponse>`)

- [ ] Every `<choicehint>` has `selected="true"` or `selected="false"`. (Required here, forbidden in `<multiplechoiceresponse>`.)
- [ ] Compound hints use letters in alphabetical order: `<compoundhint value="A B">` (not `"B A"`, not `"1 2"`).
- [ ] If choices were reordered after compound hints were written, recheck every `value=` — letters are positional, so reordering silently breaks them.
- [ ] `partial_credit` is `EDC` (Every Decision Counts), `halves`, or omitted. Other values are silently ignored.

## Text input (`<stringresponse>`)

- [ ] Special characters in `answer=` are escaped: `&` → `&amp;`, `<` → `&lt;`, `>` → `&gt;`.
- [ ] If `type="regexp"`, regexes are anchored with `^…$` unless partial matching is intentional.
- [ ] If `type` is omitted, the default is `ci` (case-insensitive).

## Formula (`<formularesponse>`)

- [ ] `samples` ranges avoid singularities in the answer (don't sample at 0 for `1/x` or `log(x)`).
- [ ] A `<responseparam type="tolerance">` is set — formula grading samples-and-compares numerically, so tolerance still applies.
- [ ] The number of sample points (`#N` in `samples`) is at least 5; 10 is the typical default.

## Custom Python (`<customresponse>`)

- [ ] The `def` line of the check function starts at **column 0** of the `<script>` block. Indenting it (even by one space) means the function won't be found.
- [ ] `cfn=` on `<customresponse>` matches the function name exactly (case-sensitive).
- [ ] The script only imports from the CodeJail allowlist: `math`, `random`, `numpy`, `sympy`, sometimes `scipy`, and LON-CAPA helpers. **Not** `matplotlib`, `pandas`, `os`, `subprocess`, `requests`, network or filesystem.
- [ ] If the problem uses per-student randomization, `random.seed(anonymous_student_id)` is at the top of the script, with derived values stored in module-level variables that the check function reads. The check function does **not** reference `anonymous_student_id` directly — it's out of scope at grading time.
- [ ] Returns a dict with `'ok'` and `'msg'` (not just `True`/`False`) so the learner sees feedback.
- [ ] Returning `None` is treated as `False` — every code path through the check function returns something explicit.
- [ ] If using `f"…"` strings, all quotes are balanced and there's no stray backslash that would break the XML parse of the surrounding `<script>`. When in doubt, wrap the whole script in `<![CDATA[…]]>`.

## Image-mapped (`<imageresponse>`)

- [ ] `width` and `height` match the displayed image size (where coordinates were measured) — not the original file dimensions.
- [ ] `alt` text describes the image for screen readers.
- [ ] The skill warned the author that image-click is not keyboard-accessible and offered drag-and-drop v2 as an accessible alternative.

## Studio one-way doors

The Studio simple/visual editor cannot re-open a problem that uses any of these features. The "Switch to advanced editor" button is one-way; the moment you save with any of these, the visual editor is gone for that problem:

- `<script>` blocks of any kind
- `partial_credit` attribute
- `targeted-feedback` attribute
- `<additional_answer>` (more than one correct answer)
- `<stringequalhint>` (targeted wrong-answer feedback)
- `regexp` in a string response `type`
- `<formularesponse>`, `<customresponse>`, `<imageresponse>` (these types have no visual editor at all)

If the author is going from a simple-editor problem to one of these features, mention this trade-off once per session so they know they're committing to raw OLX for that problem going forward.

## XML hygiene

- [ ] No unescaped `&`, `<`, `>` outside of intentional tags.
- [ ] Attribute values are quoted (preferably double-quotes).
- [ ] If a value needs to contain double-quotes, switch the attribute's outer quotes to single-quotes rather than escaping inside.
- [ ] No stray Windows line endings inside `<script>` (rarely an issue but can break the Python parse).
