# Where OLX problem files live

OLX is the format produced when you export an Open edX course to a tarball, and the format Studio reads when you import one. Authors who edit OLX outside Studio need to know how the pieces fit together.

## Directory layout of a course tarball

```
course.tar.gz
└── course/
    ├── course.xml                 # one-line pointer: <course url_name="<run>" org="…" course="…"/>
    ├── course/<run>.xml           # actual course settings + ordered list of chapters
    ├── chapter/<url_name>.xml     # sections (top-level units in the outline)
    ├── sequential/<url_name>.xml  # subsections (one click to enter)
    ├── vertical/<url_name>.xml    # units (the page learners actually see)
    ├── problem/<url_name>.xml     # one file per problem component
    ├── html/<url_name>.xml        # one file per HTML component
    │   <url_name>.html            # the actual HTML; xml file is metadata pointer
    ├── video/<url_name>.xml       # one per video component
    ├── static/                    # uploaded files: images, PDFs, attachments
    ├── policies/<run>/
    │   ├── policy.json            # grading policy, advanced module list, run metadata
    │   └── grading_policy.json
    └── assets/                    # legacy asset metadata
```

The hierarchy is **course → chapter → sequential → vertical → component**. Each level references its children by `url_name`.

## How a problem gets referenced

Inside a unit (vertical), a problem is referenced like this:

```xml
<!-- vertical/intro_unit.xml -->
<vertical display_name="Lesson 1: Introduction" url_name="intro_unit">
  <html url_name="intro_text"/>
  <problem url_name="warmup_question"/>
  <video url_name="welcome_video"/>
</vertical>
```

That `<problem url_name="warmup_question"/>` reference is a **pointer** — the actual problem definition lives in `problem/warmup_question.xml`:

```xml
<!-- problem/warmup_question.xml -->
<problem display_name="Warmup">
  <multiplechoiceresponse>
    <!-- … -->
  </multiplechoiceresponse>
</problem>
```

### Pointer syntax vs inline syntax

You can alternatively inline the full problem definition inside the vertical:

```xml
<!-- vertical/intro_unit.xml (inline form) -->
<vertical display_name="Lesson 1: Introduction" url_name="intro_unit">
  <html url_name="intro_text"/>
  <problem display_name="Warmup" url_name="warmup_question">
    <multiplechoiceresponse>
      <!-- … -->
    </multiplechoiceresponse>
  </problem>
</vertical>
```

Both forms are valid for built-in `<problem>` blocks and round-trip through Studio import/export correctly.

**For custom XBlocks** (drag-and-drop-v2, third-party blocks), there's a known Studio import bug in recent releases (Redwood/Sumac/master) where pointer syntax silently drops the block. The safe path: use **pointer syntax for built-in problems** (more readable, one file per problem, easier to diff) and **inline syntax for custom XBlocks** until that bug is fixed upstream.

## `url_name` rules

- Must equal the filename basename: `url_name="foo"` ↔ `problem/foo.xml`.
- Only `[a-zA-Z0-9_-]`. No spaces, no dots, no slashes. Studio will rewrite invalid characters but the rewrite is not always reversible.
- Must be unique within its block type (no two problems can share a `url_name`).
- The full opaque key the platform uses internally is `block-v1:<org>+<course>+<run>+type@<block_type>+block@<url_name>`. Authors don't write these but seeing one in an error message tells you which block had the problem.

## Course key

The whole course is keyed as `course-v1:<org>+<course>+<run>` — for example, `course-v1:HarvardX+CS50+2025_Q1`. The three parts come from `course.xml`:

```xml
<course url_name="2025_Q1" org="HarvardX" course="CS50"/>
```

`<run>` is the `url_name` here, and `course/2025_Q1.xml` is the file the platform actually loads.

## Editing without exporting the tarball

Most authors don't hand-edit OLX tarballs — they use Studio's advanced editor on individual problems. The OLX-tarball workflow comes up when:

- Bulk-generating many problems (script → tarball → import).
- Version-controlling course content (git of the tarball contents).
- Migrating courses between Open edX instances.
- Producing a deliverable for a course platform that imports OLX (some publishers).

For Studio editing of one problem at a time, the author opens the component, clicks **Edit → Advanced Editor**, and pastes the contents of what would otherwise be `problem/<url_name>.xml` — but **without** the surrounding policies, just the `<problem>...</problem>` element.

## `policy.json` (mention)

`policies/<run>/policy.json` holds course-wide settings: grading policy (assignment weights, drop counts, pass mark), advanced module list (which XBlocks are enabled), discussion configuration, certificate criteria. You rarely edit it by hand — Studio's grading UI writes it. If an author asks "where does my grading policy live", the answer is here.

Per-component metadata (display name, max attempts, weight, etc.) can also live in `policy.json` keyed by the block's full opaque key, but for problems it's standard to put it as attributes on the `<problem>` element instead. Studio export prefers attribute placement.
