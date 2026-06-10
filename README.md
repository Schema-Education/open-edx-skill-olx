# open-edx-skill-olx

A Claude Code skill for authoring Open edX OLX problems — multiple choice, checkboxes, dropdown, numerical input, text input, formula/math, custom Python-graded, image-mapped, and the surrounding course-XML structure.

OLX (Open Learning XML) is the format used by every problem and component in an Open edX course. Studio has a visual editor for the simple cases, but anything serious — partial credit, scripted graders, regex matching, formula equivalence, targeted feedback — requires dropping to raw OLX. This skill teaches Claude the patterns, attributes, and gotchas needed to produce OLX that imports correctly the first time.

## What the skill does

When Claude detects an OLX-related prompt (the description triggers on keywords like "OLX", "Open edX problem", "advanced editor", `<multiplechoiceresponse>`, `<numericalresponse>`, `<customresponse>`, "Studio import", "LON-CAPA", "course tarball", `course.xml`, `url_name`), it loads the skill and follows the workflow in [`SKILL.md`](./SKILL.md):

1. Identify the problem type and consult the relevant section of [`references/problem-types.md`](./references/problem-types.md).
2. Adapt the matching template from [`templates/`](./templates/) (rather than composing OLX from memory).
3. Run the pre-return checklist in [`references/gotchas.md`](./references/gotchas.md) — tolerance defaults, the `<choicehint selected>` vs bare `<choicehint>` distinction, `<script>` indentation, escape rules, etc.
4. Return the complete `<problem>` element plus a one-paragraph rationale for any non-obvious choice.

## Layout

```
SKILL.md                              top-level workflow + decision tree
references/
  problem-types.md                    per-response-type reference (skeletons, attributes, gotchas)
  customresponse-python.md            CodeJail allowlist, per-student randomness, multi-input
  course-structure.md                 where problem/<url_name>.xml lives in a course export
  gotchas.md                          pre-return checklist
templates/
  multiple-choice.xml                 11 copy-pasteable scaffolds, one per response type
  checkbox.xml                        (with the gotcha-mitigations already baked in)
  dropdown.xml
  numerical.xml
  numerical-partial-credit.xml
  text-input.xml
  text-input-regex.xml
  formula.xml
  customresponse.xml
  customresponse-multi-input.xml
  image-response.xml
evals/
  evals.json                          3 realistic test prompts used to validate the skill
```

## Installing

This skill is consumed by Claude Code via the `skills-lock.json` pattern. From a repo where you want the skill available, add an entry alongside any other skills:

```json
{
  "version": 1,
  "skills": {
    "skill-openedx-OLX": {
      "source": "Schema-Education/open-edx-skill-olx",
      "sourceType": "github",
      "skillPath": "SKILL.md"
    }
  }
}
```

Then start a Claude Code session in that directory and ask any OLX-related question — Claude will pick up the skill automatically.

## Validation

The skill was validated against three realistic authoring prompts (chemistry multiple choice, physics numerical with partial credit, customresponse with per-student randomization) using paired with-skill / baseline subagent runs. With-skill scored 100% on the assertion set; baseline scored 82%. The most notable baseline failure was a silent partial-credit bug — `partial_credit="3"` on `<responseparam>` is invalid syntax but produces no error at import time; the skill prevents this.

The eval prompts and assertions live in [`evals/evals.json`](./evals/evals.json).

## About

This skill is one of several Open edX-focused tools Schema Education has published. The catalog of related tools lives at [oex-tools.schema.education](https://oex-tools.schema.education). It originally lived in the [schema-oex-tools](https://github.com/Schema-Education/schema-oex-tools) repository and was extracted here so it can be versioned and consumed independently.

## License

[MIT](./LICENSE)
