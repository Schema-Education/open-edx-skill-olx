# OLX problem-type reference

One section per response tag. Each section has: minimal skeleton, key attributes, feedback options, partial credit (if supported), and the gotchas authors hit most.

## `<problem>` wrapper attributes

Every problem starts with `<problem ...>`. Attributes apply to the whole problem.

| Attribute | Values | What it does |
|---|---|---|
| `display_name` | string | Title shown to learners and in the gradebook. Always set this. |
| `max_attempts` | integer | Cap on Submit clicks. Omit for unlimited (ungraded practice). |
| `weight` | float | Total point value. Defaults to the number of response fields, which is rarely what you want for multi-field problems — set explicitly. |
| `showanswer` | `finished`, `past_due`, `closed`, `attempted`, `correct_or_past_due`, `after_all_attempts`, `after_all_attempts_or_correct`, `never`, `always` | When the Show Answer button is enabled. `finished` is sensible for practice, `past_due` for exams. |
| `rerandomize` | `per_student`, `never`, `always`, `on_reset` | When script-driven randomness reseeds. `per_student` is the safe default. The platform caps available seeds at 20 — for true infinite variants use a `library_content` randomized block instead. |
| `show_reset_button` | `true`, `false` | Whether learners can wipe their attempt. |
| `submission_wait_seconds` | integer | Throttle between Submit clicks. |
| `attempts_before_showanswer_button` | integer | Forces N attempts before Show Answer becomes available. |

Inside `<problem>`, you can use HTML (`<p>`, `<ul>`, `<img>`, `<pre>`, `<code>`) freely for prompt text. Wrap LaTeX in `\(…\)` (inline) or `\[…\]` (display).

---

## Single-select multiple choice — `<multiplechoiceresponse>`

```xml
<problem display_name="Capital of France" max_attempts="2" showanswer="finished">
  <multiplechoiceresponse>
    <label>What is the capital of France?</label>
    <description>Choose the best answer.</description>
    <choicegroup type="MultipleChoice" shuffle="true">
      <choice correct="false">London
        <choicehint>London is the capital of the United Kingdom.</choicehint>
      </choice>
      <choice correct="true">Paris
        <choicehint>Correct — Paris has been France's capital since 987.</choicehint>
      </choice>
      <choice correct="false" fixed="true">None of the above
        <choicehint>One of the listed cities is correct.</choicehint>
      </choice>
    </choicegroup>
    <solution>
      <div class="detailed-solution">
        <p>Explanation</p>
        <p>Paris has been France's capital since the 10th century.</p>
      </div>
    </solution>
  </multiplechoiceresponse>
  <demandhint>
    <hint>Think of a city on the river Seine.</hint>
  </demandhint>
</problem>
```

- Exactly one `<choice correct="true">` (or use `correct="partial"` with `point_value="0.5"` and `partial_credit="points"` on `<multiplechoiceresponse>` for partial credit).
- Per-choice feedback: `<choicehint>` **without** a `selected` attribute. (Putting `selected="…"` here is wrong — that attribute belongs to `<choiceresponse>`.)
- Randomize order: `shuffle="true"` on `<choicegroup>` (not on the response element).
- Pin a choice to the bottom: `fixed="true"` on that `<choice>`. Use this for "All of the above" / "None of the above".
- Show only N of M choices each visit: `answer-pool="N"` on `<choicegroup>`.
- Demand hints (progressive on-demand): `<demandhint><hint>…</hint></demandhint>` as a sibling of `<multiplechoiceresponse>`, **inside** the `<problem>` wrapper.

---

## Multi-select / checkboxes — `<choiceresponse>`

```xml
<problem display_name="Mammals" weight="2.0">
  <choiceresponse partial_credit="EDC">
    <label>Which of the following are mammals?</label>
    <description>Select all that apply.</description>
    <checkboxgroup>
      <choice correct="true">Whale
        <choicehint selected="true">Yes — mammals can be aquatic.</choicehint>
        <choicehint selected="false">Whales are warm-blooded and nurse young.</choicehint>
      </choice>
      <choice correct="true">Bat</choice>
      <choice correct="false">Shark
        <choicehint selected="true">Sharks are fish, not mammals.</choicehint>
      </choice>
      <choice correct="false">Iguana</choice>
    </checkboxgroup>
    <compoundhint value="A B">Good — you picked both mammals.</compoundhint>
    <solution>
      <div class="detailed-solution"><p>Whales and bats are mammals.</p></div>
    </solution>
  </choiceresponse>
</problem>
```

- `<choicehint selected="true">` fires when the learner ticks the box; `selected="false"` fires when they leave it unticked. **The `selected` attribute is required here** — it's the difference from `<multiplechoiceresponse>`.
- `<compoundhint value="A B">` shows when the learner selects exactly that combination. The value is **space-separated uppercase letters in alphabetical order**, indexed from the choice ordering — `A` is the first `<choice>`, `B` the second. Reorder choices and these silently break. Don't reorder without rechecking compound hints.
- Partial credit modes:
  - `partial_credit="EDC"` — "Every Decision Counts". +1 per correct selected, −1 per incorrect selected, floored at 0.
  - `partial_credit="halves"` — full credit, half credit with one mistake, none with two or more.
  - Omit for all-or-nothing.

---

## Dropdown — `<optionresponse>`

```xml
<problem display_name="Phase of matter">
  <optionresponse>
    <label>At room temperature, water is a:</label>
    <optioninput>
      <option correct="False">solid<optionhint>Below 0°C.</optionhint></option>
      <option correct="True">liquid</option>
      <option correct="False">gas<optionhint>Above 100°C at 1 atm.</optionhint></option>
    </optioninput>
  </optionresponse>
</problem>
```

- Use the child-element form (one `<option>` per choice) — the inline tuple form (`<optioninput options="('a','b')" correct="b"/>`) breaks if any option contains a comma or apostrophe.
- Exactly one `correct="True"`. Both `True` and `true` are tolerated.
- Per-option feedback: `<optionhint>` as a child of `<option>`.

---

## Numerical input — `<numericalresponse>`

```xml
<problem display_name="Speed of light">
  <numericalresponse answer="3.0e8" partial_credit="close">
    <label>What is the speed of light in m/s?</label>
    <responseparam type="tolerance" default="5%" partial_range="3"/>
    <formulaequationinput size="20" trailing_text="m/s"/>
    <additional_answer answer="2.998e8">
      <correcthint>Excellent — the more precise value.</correcthint>
    </additional_answer>
    <correcthint>Correct.</correcthint>
    <solution>
      <div class="detailed-solution"><p>c ≈ 3×10⁸ m/s.</p></div>
    </solution>
  </numericalresponse>
</problem>
```

- **Always include `<responseparam type="tolerance" default="…">`** unless the answer is an exact integer. Without tolerance, `3.14` ≠ `3.140000001` and learners get zero. Use percent (`5%`) for values across many magnitudes; use absolute (`0.01`) when the scale is fixed.
- Range answers: `answer="[3.0e8, 3.1e8]"` (inclusive) or `(3.0e8, 3.1e8)` (exclusive). Mixed brackets work.
- Script-defined answer: put a `<script type="loncapa/python">` block before the response and write `answer="$varname"`.
- Use `<formulaequationinput/>` as the input element (renders MathJax live). The old `<textline math="1"/>` is deprecated.
- Units: there's no first-class unit grading. Use `trailing_text="m/s"` to display the unit *outside* the input; learners enter only the number. For real unit checking, use `<formularesponse>` or `<customresponse>`.
- Partial credit:
  - `partial_credit="close"` + `partial_range="N"` on `<responseparam>` — 50% credit within N×tolerance.
  - `partial_credit="list"` + `partial_answers="2.99e8, 3.1e8"` on `<responseparam>` — listed near-misses get 50%.
  - `partial_credit="close,list"` combines both.
- Additional correct answers: `<additional_answer answer="…">` with optional `<correcthint>`.

---

## Text input — `<stringresponse>`

```xml
<problem display_name="Author">
  <stringresponse answer="Mark Twain" type="ci">
    <label>Who wrote <em>Huckleberry Finn</em>?</label>
    <additional_answer answer="Samuel Clemens">
      <correcthint>That's his real name.</correcthint>
    </additional_answer>
    <additional_answer answer="(?i)^samuel\s+l(\.|anghorne)?\s+clemens$"/>
    <stringequalhint answer="Hemingway">Different author entirely.</stringequalhint>
    <textline size="30"/>
    <correcthint>Correct.</correcthint>
  </stringresponse>
</problem>
```

- `type` is a space-separated combination: `ci` (case-insensitive, default), `cs` (case-sensitive), `regexp` (regex matching). E.g., `type="ci regexp"`.
- Multiple correct answers: repeat `<additional_answer answer="…">`, optionally with its own `<correcthint>`.
- Regex: when `type="regexp"` is set, both `answer` and `additional_answer` values are treated as Python regexes. **Anchor with `^…$`** unless you want partial matches.
- Targeted wrong-answer feedback: `<stringequalhint answer="…">message</stringequalhint>`.
- Escape `&`, `<`, `>` in `answer=` values (`&amp;`, `&lt;`, `&gt;`). If the regex contains single-quotes, surround the attribute with double-quotes.

---

## Formula / math expression — `<formularesponse>`

```xml
<problem display_name="Voltage divider">
  <formularesponse type="ci"
                   samples="R_1,R_2,V_in@1,1,1:10,10,10#10"
                   answer="V_in*R_2/(R_1+R_2)">
    <label>Express V_out in terms of V_in, R_1, R_2.</label>
    <responseparam type="tolerance" default="0.00001"/>
    <formulaequationinput size="40"/>
  </formularesponse>
</problem>
```

- How grading works: the platform samples the learner's expression and the author's `answer` at N random points within the variable ranges and compares numerically within tolerance. This catches algebraic equivalents the author never enumerated.
- `samples` syntax: `"VARS@LOW_BOUNDS:HIGH_BOUNDS#N"`. Variables and bounds are comma-separated and positional. N is the number of sample points (10 is typical).
- Choose sample ranges that **avoid singularities** in the answer — don't sample at 0 for `1/x`. Use small positive bounds when in doubt.
- Variables can have subscripts (`m_1`), super+subscripts (`G_{ij}^{12}`), primes (`f'`). The same parser drives MathJax preview, so what you write is what learners see typeset.
- Difference from `<numericalresponse>`: `<numericalresponse>` allows only numeric expressions with no free variables; `<formularesponse>` requires free variables declared in `samples`.

For algebraic-equivalence grading without sampling, see `<symbolicresponse>` (legacy; requires `<textline math="1">`). Most uses are covered by `<formularesponse>`.

---

## Custom Python-graded — `<customresponse>`

See `customresponse-python.md` for the full pattern, allowed libraries, and per-student randomization. Minimal skeleton:

```xml
<problem display_name="Even number">
  <script type="loncapa/python">
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
</problem>
```

- `cfn` names a function defined in the `<script type="loncapa/python">` block.
- The `def` line **must start at column 0**. Indenting (even by one space) makes the function invisible to the grader.
- Return value: `True`/`False`/`'Partial'`, or a dict `{'ok': ..., 'msg': '<xhtml>...</xhtml>'}`, or for multi-input `{'overall_message': str, 'input_list': [{'ok': ..., 'msg': ..., 'grade_decimal': float}, ...]}`.
- Returning `None` is treated as `False`.

---

## Image-mapped input — `<imageresponse>`

```xml
<problem display_name="Locate Egypt">
  <imageresponse>
    <p>Click on Egypt.</p>
    <imageinput src="/static/africa.png"
                width="600" height="638"
                rectangle="(338,98)-(412,168)"
                alt="Map of Africa"/>
  </imageresponse>
</problem>
```

- Required attributes: `src`, `width`, `height`, `alt`, and either `rectangle` or `regions`.
- `rectangle="(x1,y1)-(x2,y2)"` upper-left to lower-right. Multiple rectangles separated by `;`.
- `regions="[[x1,y1],[x2,y2],[x3,y3],...]"` for polygons (≥3 vertices).
- Coordinates are pixels measured from top-left of the displayed image — if you change `width`/`height` from the original, coordinates won't scale. Record coordinates against the final displayed size.
- **Accessibility:** image-click has no keyboard equivalent. For accessibility-sensitive courses, recommend drag-and-drop v2 (an XBlock, configured via the visual editor) instead.

---

## Drag-and-drop v2 (XBlock, brief)

This isn't an OLX problem tag — it's a separate XBlock (`drag-and-drop-v2`) with its own JSON-configured visual editor. In OLX it appears as `<drag-and-drop-v2 url_name="..." data='{...}'/>` where `data` holds the JSON the editor produces. Hand-authoring the JSON is painful and error-prone — instead, tell the author to configure it in Studio's visual editor and export the JSON if they need version control.

One gotcha worth flagging: drag-and-drop v2 problems **cannot be placed inside content libraries** (the randomized-pool feature). If the author wants randomized drag-and-drop, this isn't possible today; suggest converting to multiple checkbox problems instead.

---

## Less-common types (mention only)

- `<symbolicresponse>` — symbolic algebraic equivalence; requires legacy `<textline math="1">`. Most cases now use `<formularesponse>`.
- `<matlabresponse>` — connects to a MathWorks MATLAB server (needs API key). Niche.
- `<chemicalequationinput>` — typesets chemical formulas live; usually graded with `<customresponse>` + the `chemcalc` helper.
- `<schematicresponse>` — circuit-drawing input graded against SPICE-like netlists. Niche.

If the author asks for one of these, point them to the upstream MIT example courses rather than templating from scratch — these tags are version-fragile.
