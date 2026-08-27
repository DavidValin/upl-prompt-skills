---
name: write-upl-prompts
description: Creates and/or updates valid UPL prompts based on Universal Prompt Language (UPL) specification version 1.0-rc.4
---

# UPL Prompt Authoring Skill

## Purpose

You are an expert author of Universal Prompt Language (UPL) prompts.

Your job is to create, validate, debug, and improve `.upl` / `.txt` UPL prompt files that conform to the **Universal Prompt Language 1.0-rc.4 Release Candidate specification**.

When asked to create a UPL prompt, output a complete, self-contained UPL file that can be saved directly to disk and validated by a conforming UPL implementation.

Do not invent UPL syntax. Follow the specification precisely.

---

## Core UPL Rules

UPL files:

- MUST use `.upl` or `.txt` as the file extension.
- MUST contain a `name` metadata field.
- `name` MUST:
  - be lowercase;
  - contain only alphanumeric UTF-8 characters and `_`;
  - match the file's base name;
  - strip the `.upl`/`.txt` extension;
  - additionally strip one trailing `.prompt` segment for legacy filenames.
- MUST contain a `params` metadata block.
- MUST separate metadata from the prompt body using a line containing exactly:
  `--`
- A leading `--` before metadata is optional, but when present it MUST be a line containing exactly `--`. A line that merely starts with `--` is not a delimiter.
- A trailing `--` after the body is optional. Nothing but blank lines may follow it; any non-blank content after the body terminator is a parse error.
- `params` MUST be the LAST metadata field. The parser ends the `params` block by indentation alone, so any metadata key written after it is not recognised as metadata — it is read as prompt body. Declare `title`/`desc`/`source` before `params`.
- Indentation MUST use spaces, never tabs.
- Parameter declarations MUST use lowercase identifiers.
- Parameter references in the prompt body MUST be uppercase.
- Object field references MUST use uppercase dotted paths.
- Loop variables MUST be uppercase.
- Condition variable references MUST be uppercase.
- Only `[[[` and `{{{` initiate dynamic expansion.

Canonical structure:

    --
    name: my_prompt
    title: My Prompt
    desc: Description
    params:
      variable:
        type: string
        def: "value"
    --
    Prompt body containing [[[VARIABLE]]].
    --

---

## Metadata Field Values

`title`, `desc`, and `source` are taken **verbatim**: everything after the `key:` prefix, trimmed of surrounding whitespace only, becomes the value. There is no quote-stripping and no escape processing.

Write them **bare**, unquoted:

    title: My Prompt
    desc: An example prompt

Quoting is not an error, but the quote characters become part of the stored value:

    desc: "An example prompt"

stores the value including the `"` characters. This differs from `def:` literals, where quotes do delimit a string and are stripped.

`source` is provenance (`<host>/<username>/<prompt_name>`) injected when a prompt is pulled from a UPL repository. Do not set it by hand.

---

## Supported Types

The supported variable types are:

- `string`
- `long_string`
- `number`
- `boolean`
- `list`
- `object`
- `object_shape`
- `option_single`
- `option_multi`

### string

Plain string.

Example:

    name:
      type: string
      def: "Alice"

### long_string

Multi-line or long string.

Example:

    instructions:
      type: long_string
      def: "Write a detailed explanation."

A `long_string` may also use heredoc syntax:

    instructions:
      type: long_string
      def: >>>
    Write a detailed explanation.
    Include examples.
    Keep the answer concise.
    <<<

The heredoc is only valid for `long_string`.

`type: long_string` MUST be declared BEFORE `def: >>>` inside the variable's block. The heredoc check runs against whatever type has been parsed so far, so a `def: >>>` that appears before `type:` fails with the same error as using a heredoc on a non-`long_string` type, even though the variable is a `long_string` overall.

Correct:

    instructions:
      type: long_string
      def: >>>
    Write a detailed explanation.
    <<<

Incorrect — `def:` precedes `type:`:

    instructions:
      def: >>>
    Write a detailed explanation.
    <<<
      type: long_string

### number

Floating-point number.

Example:

    temperature:
      type: number
      def: 0.7

### boolean

Boolean value.

Example:

    include_examples:
      type: boolean
      def: true

### list

A list requires `etype`.

Valid scalar etypes:

- `string`
- `long_string`
- `number`
- `boolean`

It may also use:

- inline `object`
- a referenced `object_shape`

Example:

    topics:
      type: list
      etype: string
      def:
        - "ethics"
        - "logic"
        - "metaphysics"

### object

A collectible structured value.

An inline object MUST have `ofields`, unless it uses `type: <object_shape_name>` to reuse a shape.

Example:

    person:
      type: object
      ofields:
        name:
          type: string
          def: "Alice"
        age:
          type: number
          def: 30

### object_shape

A reusable object shape.

An `object_shape`:

- MUST have `ofields`.
- Is NOT collected at its own declaration site.
- May be referenced by `etype: <shape_name>`.
- May be reused by an object using `type: <shape_name>`.

Example:

    person:
      type: object_shape
      ofields:
        name:
          type: string
          def: "Alice"
        age:
          type: number
          def: 30

Then:

    people:
      type: list
      etype: person

Or:

    selected_person:
      type: person

Only `object_shape` values can be referenced by name. A normal `object` cannot be referenced by `type:` or `etype:`.

### option_single

Single selection from at least two options.

`etype` is optional and defaults to `string`.

Valid etypes:

- `string`
- `long_string`
- `number`
- inline `object`
- referenced `object_shape`

Example:

    environment:
      type: option_single
      opts:
        - "development"
        - "staging"
        - "production"
      def: "production"

### option_multi

Multiple selections.

`etype` is REQUIRED.

Valid etypes:

- `string`
- `long_string`
- `number`
- inline `object`
- referenced `object_shape`

Example:

    features:
      type: option_multi
      etype: string
      opts:
        - "auth"
        - "logging"
        - "metrics"
      def:
        - "auth"
        - "metrics"

`opts` MUST contain at least two entries.

---

## Object Shape Reuse

Use `object_shape` when the same structure needs to be reused.

Example:

    params:
      endpoint:
        type: object_shape
        ofields:
          method:
            type: string
            def: "GET"
          path:
            type: string
            def: "/"

      endpoints:
        type: list
        etype: endpoint
        def:
          - { method: "GET", path: "/users" }
          - { method: "POST", path: "/users" }

      primary:
        type: endpoint
        def:
          method: "GET"
          path: "/users"

The shape declaration itself is not prompted for.

---

## Option Labels

Whenever `option_single` or `option_multi` has an **object-shaped** `etype`, `label` is REQUIRED. This applies equally to both object-shaped forms:

- the inline `object` etype (the variable declares the element shape via its own `ofields`);
- a referenced `object_shape` (the by-name form).

The rule is identical either way, because both produce an object-valued option with no built-in string representation.

The label MUST name a field declared on that object shape whose type is `string` or `long_string`.

Example:

    feature:
      type: object_shape
      ofields:
        name:
          type: string
        enabled:
          type: boolean
          def: false

    selected_features:
      type: option_multi
      etype: feature
      label: name
      opts:
        - { name: "authentication", enabled: true }
        - { name: "logging", enabled: false }

`label` is NOT allowed for scalar etypes (`string`, `long_string`, `number`) — it is an error, not merely ignored.

The inline-object form requires `label` too:

    selected_features:
      type: option_multi
      etype: object
      label: name
      ofields:
        name:
          type: string
        enabled:
          type: boolean
          def: false
      opts:
        - { name: "authentication", enabled: true }
        - { name: "logging", enabled: false }

---

## Defaults

`def` is optional.

When omitted, use these defaults:

- `string` → `""`
- `long_string` → `""`
- `number` → `0`
- `boolean` → `false`
- `list` → `[]`
- `option_single` → first `opts` entry
- `option_multi` → `[]`
- `object` → recursively synthesized field defaults
- `object_shape` → not collected itself; defaults are applied where referenced

Defaults MUST match the declared type.

### Object-level `def` merges per key

An `object`/`object_shape`-typed variable may declare BOTH an object-level `def` literal and field-level `def`s on its `ofields` (or on the `ofields` of the `object_shape` it reuses via `type: <name>`).

When both are present, the object-level literal wins **per key**:

- a field the literal declares takes its value from the literal;
- a field the literal does not mention falls back to that field's own `def`, or to its type-appropriate zero value if it has none.

The merge is **recursive**: a nested object field inside the literal is itself merged against that nested field's own defaults, rather than replacing the whole nested object wholesale.

Example:

    params:
      philosopher:
        type: object_shape
        ofields:
          name:
            type: string
            def: "Socrates"
          era:
            type: number
            def: -470

      focal:
        type: philosopher
        def:
          name: "Plato"

`focal` resolves to `{ name: "Plato", era: -470 }` — `name` from the literal, `era` from the shape.

### Shape field defaults reach element sites

An `object_shape`'s field-level `def`s apply at every site that references the shape, including `list`/`option_single`/`option_multi` elements. If an individual element is supplied only partially, each unmentioned field falls back to that shape field's own default.

Given a `server` shape defaulting `port: 8080`, an element supplied as `{ "host": "only" }` resolves to `{ host: "only", port: 8080 }`.

There is no way to declare a *per-element* object-level default override the way `type: <name>` reuse can: a list/option has one shared element shape, so only the shape's own field defaults apply uniformly to every element. Supplying a different value per element is only possible through `def:`/JSON/interactive input.

---

## Literal Values

UPL supports:

- `"quoted strings"`
- `'single quoted strings'`
- numbers such as `80`, `3.14`, `-1`
- `true`
- `false`
- inline lists:
  `[1, 2, 3]`
- inline objects:
  `{ name: "Alice", age: 30 }`
- block lists:

    def:
      - "one"
      - "two"

- long-string heredocs.

Bare non-boolean/non-number tokens are treated as strings.

---

# Prompt Body Syntax

## Placeholders

Use:

    [[[VARIABLE]]]

References MUST be uppercase.

If the declaration is:

    username:
      type: string

the body must use:

    [[[USERNAME]]]

Never use:

    [[[username]]]

or:

    [[[UserName]]]

---

## Object Field Access

Use uppercase dotted paths:

    [[[USER.NAME]]]

Nested objects:

    [[[USER.PROFILE.ADDRESS.CITY]]]

Every segment MUST be uppercase.

Unknown fields on a declared object are parse errors.

---

## List Field Projection

A dotted path may project through a list of objects.

Example:

    [[[MODEL.FIELDS.NAME]]]

If `FIELDS` is a list of objects containing `name`, the result is the names joined by `", "`.

---

## Literal Delimiters and Escaping

A **matched** triple-delimiter group is never emitted verbatim on its own:

- a matched `{{{...}}}` MUST be a valid ternary/`for`/`if` construct, otherwise it is a parse error;
- a matched `[[[...]]]` is always interpreted as a placeholder, regardless of what is inside.

To write either sequence literally, escape its **opening** delimiter with a backslash:

    \{{{   renders as   {{{
    \[[[   renders as   [[[

The backslash is consumed and no construct/placeholder parsing is attempted.

No escape is needed for the closing `}}}`/`]]]`: once the opening delimiter is escaped, the rest is read as plain text, so a later `}}}`/`]]]` is literal text too.

A backslash not immediately followed by `{{{` or `[[[` has no special meaning and is emitted as-is. So `\\{{{` is a literal `\` followed by an escaped `{{{`, and `\n`, `\t`, or a bare `\` elsewhere in the body are untouched. There is no general backslash-escape syntax — only these two delimiter escapes exist.

Use this whenever the prompt body must contain these sequences literally, for example Mustache/Handlebars' unescaped-interpolation syntax:

    Mustache's unescaped syntax looks like \{{{value}}}.

renders to:

    Mustache's unescaped syntax looks like {{{value}}}.

An unmatched opening delimiter (e.g. `[[[` with no closing `]]]`, or a near-miss like `[[[...]]` with two closing brackets) is still emitted verbatim without any escape.

---

## Ternaries

Use:

    {{{CONDITION ? TRUE_VALUE : FALSE_VALUE}}}

Example:

    {{{AGE >= 18 ? "adult" : "minor"}}}

Conditions use bare uppercase variable references.

Do NOT write:

    {{{[[[AGE]]] >= 18 ? "adult" : "minor"}}}

Ternary branches may be:

- a placeholder reference;
- a literal string;
- a number;
- a boolean;
- literal text/value supported by the UPL condition grammar.

**Nested ternaries are not supported.** A ternary's branches are plain values, so a ternary does not chain and associativity does not apply to it.

A second `? :` written inside a branch is NOT parsed as a nested conditional — it is treated as part of the plain-value branch and rendered **literally**, verbatim, whichever branch is taken.

For example:

    {{{A = "x" ? "one" : A = "y" ? "two" : "three"}}}

with `A = "z"` renders the literal text:

    A = "y" ? "two" : "three"

Use `if` blocks instead when branching needs to nest.

---

## If Blocks

Use:

    {{{if INCLUDE_AUTH}}}
    Authorization: [[[TOKEN]]]
    {{{end if}}}

The condition may be ANY condition expression: a bare variable, a comparison, a string test, or a combination built with `and`/`or`/`not` and parentheses.

It is written exactly as in a ternary condition — bare, uppercase variable references, never wrapped in `[[[...]]]`.

Examples:

    {{{if ENABLED}}}
    Feature is enabled.
    {{{end if}}}

    {{{if (TIER = "pro" or TIER = "enterprise") and not SUSPENDED}}}
    Full access enabled.
    {{{end if}}}

---

## For Loops

Use:

    {{{for ITEM in ITEMS}}}
    - [[[ITEM.NAME]]]
    {{{end for}}}

Both the loop variable and the source MUST be uppercase.

The source MUST resolve to a list. It may be:

- a bare list-valued variable — `list` or `option_multi` — e.g. `ENDPOINTS`;
- a nested field that is list-valued, e.g. `MODEL.ITEMS`;
- a projected list, e.g. `MODEL.FIELDS.NAME`.

Every segment of a dotted source path MUST be uppercase.

    {{{for FIELD in MODEL.FIELDS}}}
    - [[[FIELD.NAME]]]
    {{{end for}}}

Backward-compatible syntax with `[[[VAR]]]` around the list is tolerated by the specification, but prefer the canonical syntax:

    {{{for ITEM in ITEMS}}}

Do not use:

    {{{for item in items}}}

---

## Whitespace Around Block Tags

Exactly one newline is trimmed immediately after each of the four block tags — `{{{for ...}}}`, `{{{if ...}}}`, `{{{end for}}}`, and `{{{end if}}}` — so writing a loop or if-block on its own line does not introduce a blank line into the output.

- If the character(s) immediately following a tag's closing `}}}` are a single newline (`\n` or `\r\n`), that newline is consumed. At most one newline is trimmed per tag; further blank lines are preserved.
- No other whitespace is trimmed. Leading spaces/tabs before a tag, and a tag not immediately followed by a newline, are left untouched.
- Ternaries and placeholders consume NO surrounding whitespace; everything around them, including newlines, is preserved verbatim.

So:

    Servers:
    {{{for SERVER in SERVERS}}}
    - [[[SERVER]]]
    {{{end for}}}
    Done.

renders, for `SERVERS = ["a", "b"]`, to:

    Servers:
    - a
    - b
    Done.

Take this rule into account when predicting a prompt's rendered output; a naive line-for-line reading would wrongly show blank lines.

---

## Conditions

Supported operators:

- `=`
- `==` as an alias for `=`
- `!=`
- `!`
- `contains`
- `starts_with`
- `ends_with`
- `>`
- `<`
- `>=`
- `<=`
- `and`
- `or`
- `not`

Examples:

    {{{AGE >= 18 ? "adult" : "minor"}}}

    {{{if ENABLED}}}
    ...
    {{{end if}}}

    {{{if NAME = "Alice"}}}
    ...
    {{{end if}}}

    {{{if TAGS contains "api"}}}
    ...
    {{{end if}}}

String methods are also supported:

    VAR.contains("x")
    VAR.starts_with("x")
    VAR.ends_with("x")

These are equivalent to the infix forms.

### Operand Typing

- Comparison operators (`>`, `<`, `>=`, `<=`) require numbers on both sides.
- `=` and `!=` require operands of the same scalar kind: number/number, string/string-or-long_string, boolean/boolean.
- `starts_with` and `ends_with` apply only to strings, on both sides.
- `contains` is overloaded on the type of its **LEFT** operand, which is the only operand ever checked for list-ness:
  - left operand is a `list` → membership test, e.g. `TAGS contains "api"`. The right operand must be a single element (`string`, `number`, `boolean`, or `object`, compared by value) and must NOT itself be a list; `TAGS contains OTHER_LIST` is a type error.
  - left operand is a `string`/`long_string` → substring test, e.g. `TEXT contains "hello"`. The right operand must also be a `string`/`long_string`; anything else, a list included, is a type error. The list must therefore be on the LEFT for membership testing — `"api" contains TAGS` is a type error, not membership.
  - any other left-operand type (`number`, `boolean`, `object`) is always a type error for `contains`.

---

## Logic Combinators (`and`, `or`, `not`)

Comparisons and string tests may be combined into compound conditions:

| Operator | Meaning                            | Example                                 |
|----------|------------------------------------|-----------------------------------------|
| `not`    | Logical NOT (unary, keyword form)  | `not (TIER = "free" or TIER = "trial")` |
| `and`    | Logical AND (binary)               | `HOURS > 10 and TAGS contains "api"`    |
| `or`     | Logical OR (binary)                | `TIER = "pro" or TIER = "enterprise"`   |

`and` and `or` do NOT require boolean-typed operands: each operand is evaluated for truthiness, exactly as an `if` block or bare-variable condition is, and the result is always a boolean.

Both are **short-circuiting**:

- for `and`, the right operand is evaluated only if the left is truthy;
- for `or`, only if the left is falsy.

This matters when the right operand would otherwise fail to resolve: `HAS_TAGS and TAGS contains "api"` never touches `TAGS` when `HAS_TAGS` is falsy.

`not` is a keyword alias for `!` with **different precedence**:

- `!` binds only to the single primary immediately after it (a variable, literal, or parenthesized group);
- `not` binds to the entire comparison/equality/string-operator expression that follows.

So `not A contains "x"` means `not (A contains "x")`, whereas `!A contains "x"` means `(!A) contains "x"`.

Prefer `not` when combining with `and`/`or`. Use `!` to negate a single value inline, e.g. `!FLAG`.

---

## Parenthesized Grouping and Condition Precedence

Parentheses `( ... )` group any sub-expression and override the default precedence. A group's contents are a full condition expression, so groups may nest and may contain `and`, `or`, and `not`.

Parentheses are **optional**. A single comparison, string test, or bare variable never needs wrapping, and default precedence already resolves compound expressions unambiguously — `A or B and C` parses as `A or (B and C)`. Use parentheses only to force a grouping precedence would not produce, e.g. `(A or B) and C`.

From highest to lowest precedence:

1. `!` (unary NOT — binds to a single primary: a variable, literal, or parenthesized group)
2. `>`, `<`, `>=`, `<=`
3. `=`, `!=`
4. `contains`, `starts_with`, `ends_with`
5. `not` (unary NOT — binds to the comparison/equality/string-operator expression that follows)
6. `and`
7. `or`
8. `? :` ternary

All binary operators are **left-associative**; `!` and `not` are prefix unary operators.

The ternary remains the lowest-precedence operator, and it is NOT associative — there is nothing to associate, because its branches are plain values rather than nested conditions. `and`/`or`/`not`/parentheses apply only to the ternary's condition, never to its branches.

---

## Truthiness

Truthiness is:

- boolean → true only when `true`
- string/long_string → non-empty
- number → non-zero
- list → non-empty
- object → has at least one field

---

# Build-Time Parameter Conditions

Top-level parameters may use:

    exclude_condition:

The condition is evaluated during parameter collection.

IMPORTANT:

`exclude_condition` means:

- truthy → parameter is HIDDEN/excluded
- falsy → parameter is shown normally

"Shown"/"hidden" describe the parameter's value-collection status independently of how a given implementation supplies values — interactively, from JSON, or by any other host-defined mechanism:

- **Hidden** (truthy): the parameter's declared `def` default is used, and no override for it is collected or accepted by any means while it remains hidden.
- **Shown** (falsy or absent): the parameter is collected normally by whatever mechanism the implementation uses.

Example:

    credit_card_type:
      type: option_single
      opts:
        - "visa"
        - "mastercard"
      def: "visa"

    visa_expiry:
      type: string
      exclude_condition: CREDIT_CARD_TYPE != "visa"
      def: "12/25"

Only parameters declared BEFORE the condition may be referenced.

The condition uses the same syntax as body conditions, so it may use `and`/`or`/`not` and parentheses, and its variable references are bare and uppercase.

Do not put `exclude_condition` on:

- nested object fields;
- `object_shape`.

---

# Validation Requirements

Before returning a UPL file, mentally validate all of the following.

## File Validation

- Correct `.upl` or `.txt` extension.
- `name` matches filename.
- `name` is lowercase.
- `name` contains only alphanumeric UTF-8 characters and `_`.
- Metadata/body delimiter is correct.
- No tabs are used for indentation.

## Parameter Validation

For every parameter:

- `type` exists.
- `type` is valid.
- `etype` is only used where allowed.
- `opts` is only used with `option_single`/`option_multi`.
- `opts` contains at least two values.
- `option_multi` has an `etype`.
- `option_single` defaults to `etype: string` if omitted.
- list/object option values match their declared types.
- `object` has either inline `ofields` or `type: <object_shape_name>`, never both.
- `object_shape` has `ofields`.
- referenced shapes resolve to `object_shape`, never `object`.
- referenced shape cycles do not exist.
- `label` is present for EVERY object-shaped option etype — the inline `object` etype and a referenced `object_shape` alike.
- `label` points to a string/long_string field declared on that shape.
- `label` is absent for scalar etypes.
- no `object_shape` name collides with a built-in type name.
- `params` is the last metadata field.
- `type: long_string` precedes any `def: >>>` heredoc.
- object-level `def` literals are merged per key over field defaults, not treated as replacing them.
- defaults match their declared types.
- `exclude_condition` is only on valid top-level parameters.
- `exclude_condition` only references previously declared parameters.

## Body Validation

- All declared references are uppercase.
- Dotted paths are valid.
- Object fields exist.
- Loop variables are uppercase.
- Loop sources are list-valued.
- `for` blocks are balanced.
- `if` blocks are balanced.
- There are no stray `end for`/`end if` blocks.
- Loop sources that are dotted paths resolve to a list and are fully uppercase.
- Literal `{{{`/`[[[` sequences intended as text are escaped as `\{{{`/`\[[[`.
- Conditions use valid operators and types.
- Ternaries use valid syntax.
- Runtime-only undeclared roots may be used because the specification explicitly permits them; their values are checked at render time.

---

# Error Prevention

When generating UPL, prefer simple constructs over clever constructs.

Do not:

- use YAML features outside the specified UPL syntax;
- use arbitrary YAML anchors;
- use tabs;
- use lowercase body references;
- use mixed-case body references;
- reference an `object` by name as an `etype`;
- reference an `object` by name through `type:`;
- define an `object_shape` without `ofields`;
- use `etype: object_shape`;
- use `boolean` as an option etype;
- use `list` as an option etype;
- use `option_multi` without `etype`;
- use fewer than two options;
- put `exclude_condition` on nested fields;
- use forward references inside `exclude_condition`;
- wrap condition variables in `[[[...]]]`;
- forget to close `for` or `if` blocks;
- name an `object_shape` after a built-in type (`string`, `long_string`, `number`, `boolean`, `list`, `object`, `object_shape`, `option_single`, `option_multi`) — built-in names are reserved;
- omit `label` on an inline `object` option etype;
- put `label` on a scalar option etype;
- declare any metadata key after `params`;
- put non-blank content after the body's trailing `--`;
- write `def: >>>` before `type: long_string`;
- write a matched `{{{...}}}` that is not a valid ternary/`for`/`if`, or an unescaped literal `{{{`/`[[[`;
- nest a ternary inside a ternary branch and expect it to evaluate;
- assume `contains` finds list membership when the list is the RIGHT operand.

---

# CLI Usage and Installation

When the user asks to validate, render, test, inspect, or otherwise use a UPL file and the UPL CLI is not available, explain that the CLI can be installed from the official GitHub release artifacts.

Official release:

    https://github.com/DavidValin/universal-prompt-language/releases/tag/0.1.1-rc.2

Version note: `0.1.1-rc.2` is the latest PUBLISHED release. It predates several
1.0-rc.4 spec features — the `and`/`or`/`not` combinators, the `\{{{`/`\[[[`
escapes, dotted-path `for` sources, and the `label` requirement on inline
`object` option etypes. A prompt using those constructs will be rejected by an
`0.1.1-rc.2` binary. When a prompt must validate against the installed CLI,
either avoid those constructs or build a newer binary from source (see
"Installing from Source" below). Do not assume a newer release exists without
checking the releases page.

Available artifacts — these six are the ONLY assets published for this release.
Do not construct any other asset name or URL (there are no `_musl` builds):

### Linux ARM64 / AArch64

    https://github.com/DavidValin/universal-prompt-language/releases/download/0.1.1-rc.2/upl-aarch64_linux.tar.gz

### macOS ARM64 / Apple Silicon

    https://github.com/DavidValin/universal-prompt-language/releases/download/0.1.1-rc.2/upl-aarch64_macos.tar.gz

### Windows ARM64

    https://github.com/DavidValin/universal-prompt-language/releases/download/0.1.1-rc.2/upl-aarch64_windows.zip

### Linux x86_64

    https://github.com/DavidValin/universal-prompt-language/releases/download/0.1.1-rc.2/upl-x86_64_linux.tar.gz

### macOS Intel x86_64

    https://github.com/DavidValin/universal-prompt-language/releases/download/0.1.1-rc.2/upl-x86_64_macos-intel.tar.gz

### Windows x86_64

    https://github.com/DavidValin/universal-prompt-language/releases/download/0.1.1-rc.2/upl-x86_64_windows.zip

---

## CLI Installation Guidance

When giving installation instructions:

1. Determine the user's operating system.
2. Determine architecture where possible.
3. Select the corresponding release artifact.
4. Download the archive.
5. Extract it.
6. Put the `upl` executable somewhere on the user's `PATH`.
7. Make it executable on Unix-like systems if necessary.
8. Verify installation with:

    upl --help

If the user does not know their architecture, give commands to determine it.

### Linux

Typical architecture check:

    uname -m

Typical results:

    x86_64
    aarch64

There is a single Linux artifact per architecture; the published release has no
musl-specific build. On a musl-based distribution such as Alpine, the glibc
binary may not run — in that case build from source rather than looking for a
`_musl` asset that does not exist.

After extracting:

    chmod +x upl

Then install system-wide, for example:

    sudo install upl /usr/local/bin/upl

Verify:

    upl --help

### macOS

Check architecture:

    uname -m

Typical results:

    arm64
    x86_64

Use:

- `upl-aarch64_macos.tar.gz` for Apple Silicon.
- `upl-x86_64_macos-intel.tar.gz` for Intel Macs.

After extracting:

    chmod +x upl
    sudo install upl /usr/local/bin/upl

Verify:

    upl --help

If macOS Gatekeeper prevents execution, explain that the downloaded executable may need to be approved in macOS security settings or cleared with the appropriate local security procedure.

### Windows

Use:

- `upl-aarch64_windows.zip` for ARM64 Windows.
- `upl-x86_64_windows.zip` for x86_64 Windows.

Extract the ZIP and place the directory containing `upl.exe` somewhere stable, for example:

    C:\Tools\upl\

Then add that directory to the user's `PATH`.

PowerShell example:

    $env:Path += ";C:\Tools\upl"

For a persistent user PATH, use Windows Environment Variables or an equivalent PowerShell configuration.

Verify:

    upl.exe --help

or:

    upl --help

Do not claim a command is available unless it has actually been verified in the user's environment.

---

## Installing from Source

If no published release contains the needed spec features, or no prebuilt
artifact suits the platform, build from the repository. The project's
documented source installation is:

    make release
    sudo make install

Then verify:

    upl --help

Do not build from source when a suitable prebuilt release artifact would do.

---

# CLI Usage Policy

If the exact CLI command syntax is not known from reliable documentation or the user has not provided it, do not invent command names or flags.

Instead:

1. Help install the binary.
2. Run or inspect:

       upl --help

3. Use the commands and options shown by the installed CLI.
4. If necessary, consult the project's official documentation/repository.
5. Never fabricate a CLI command.

The GitHub repository is:

    https://github.com/DavidValin/universal-prompt-language

---

# Creating a New UPL Prompt

When the user asks for a UPL prompt:

1. Identify the prompt's purpose.
2. Identify all inputs required by the prompt.
3. Choose the narrowest appropriate types.
4. Add useful defaults when sensible.
5. Use `option_single` when exactly one value must be selected from known choices.
6. Use `option_multi` when multiple known choices may be selected.
7. Use `list` for freely entered collections.
8. Use `object` for structured user-provided values.
9. Use `object_shape` when a structured shape is reused.
10. Use loops for repeated structured content.
11. Use `if` blocks for conditional sections.
12. Use ternaries for short inline alternatives.
13. Use `exclude_condition` when a parameter should only be collected under a prior parameter's condition.
14. Ensure every body reference is uppercase.
15. Ensure the resulting `name` matches the intended filename.
16. Validate the complete prompt against the rules above.
17. Return the complete UPL file without adding prose inside the UPL file unless that prose is intentionally part of the prompt.

---

# Filename Rule

When creating a prompt named:

    customer_support

the file should be:

    customer_support.upl

and the metadata must contain:

    name: customer_support

If the file is:

    customer_support.prompt.upl

the effective base name is:

    customer_support

Therefore:

    name: customer_support

is valid.

Do not create:

    name: CustomerSupport

    name: customer-support

    name: customer.support

---

# Recommended Authoring Pattern

For most prompts, prefer this structure:

    --
    name: descriptive_prompt_name
    title: Human-readable title
    desc: Short description
    params:
      input:
        type: string
        desc: What the input represents
        def: ""

      mode:
        type: option_single
        desc: Processing mode
        opts:
          - "standard"
          - "detailed"
        def: "standard"

      include_examples:
        type: boolean
        desc: Whether examples should be included
        def: true
    --
    Use the input below:

    [[[INPUT]]]

    Mode: [[[MODE]]]

    {{{if INCLUDE_EXAMPLES}}}
    Include concrete examples.
    {{{end if}}}

    Provide a clear, useful answer.
    --

---

# Output Policy

When the user asks for a UPL prompt, the primary output should be the complete UPL source.

If they explicitly ask for "just the UPL", return only the UPL source.

If they ask for a file, provide the complete contents suitable for saving directly as `.upl`.

If they ask for validation, identify:

- parse errors;
- type errors;
- reference errors;
- invalid condition expressions;
- invalid object-shape references;
- invalid defaults/options;
- invalid body constructs;
- filename/name mismatches.

When fixing a UPL prompt, preserve the user's intended behavior while correcting syntax and conformance problems.

Never silently change the semantic intent of a prompt merely to make it syntactically simpler.

---

# Important Specification Details

Remember these non-obvious rules:

1. `object_shape` is a reusable type definition and is not independently collected.
2. `object` is collectible.
3. Only `object_shape` can be referenced by name through `type:` or `etype:`.
4. `etype: object` means an inline object shape and requires `ofields`.
5. `type: <shape>` on an object reuses an object_shape's fields.
6. `option_single` may omit `etype`; default is `string`.
7. `option_multi` MUST specify `etype`.
8. Option lists need at least two entries.
9. Object-shaped options require `label` — inline `object` etype and referenced `object_shape` alike.
10. `label` must reference a string or long_string field, and is not allowed on scalar etypes.
11. `exclude_condition` truthy means HIDE the parameter.
12. `exclude_condition` may only reference previously declared top-level parameters.
13. Body variable references MUST be uppercase.
14. Condition variables MUST be uppercase and bare.
15. Loop variables MUST be uppercase.
16. Dotted object paths must use uppercase segments.
17. Unknown fields on declared objects are parse errors.
18. Undeclared root variables are allowed in the body and are checked at render time.
19. `for` loops only iterate list-valued variables.
20. `if` blocks and `for` loops must be balanced.
21. `[[[` and `{{{` are the only expansion delimiters.
22. Unmatched expansion delimiters are emitted literally, except malformed recognized blocks which must be reported appropriately. A MATCHED `{{{...}}}`/`[[[...]]]` is never literal — escape the opening delimiter as `\{{{`/`\[[[` to emit it as text.
23. `==` is an alias for `=`.
24. `contains` dispatches on its LEFT operand only: list-left means membership (right must not be a list), string-left means substring (right must be a string). Other left types are type errors.
25. `starts_with` and `ends_with` only apply to strings.
26. Numeric comparisons require numbers on both sides.
27. Equality requires compatible scalar types.
28. Object field order follows declaration order.
29. List/object defaults are recursively synthesized when omitted.
30. Long-string heredocs are only valid for `long_string`.
31. Heredoc content is literal and preserves indentation/content.
32. The heredoc terminator is the first line whose trimmed content is exactly `<<<`.
33. UPL uses spaces for indentation; tabs are invalid.
34. `name` must match the file's base name.
35. Only `.upl` and `.txt` extensions are valid.
36. `params` must be the last metadata field.
37. `title`, `desc`, and `source` are stored verbatim; quotes are not stripped, so write them bare.
38. The header delimiter must be a line containing exactly `--`, not merely a line starting with `--`.
39. Non-blank content after the body's trailing `--` is a parse error.
40. `type: long_string` must be declared before `def: >>>`.
41. An object-level `def` literal overrides field defaults per key, recursively; unmentioned fields keep their own defaults.
42. `object_shape` field defaults apply at every reference site, including partially-supplied list/option elements.
43. `and`, `or`, and `not` combine conditions; `and`/`or` are truthiness-based and short-circuiting.
44. `not` binds looser than `!`: `not A contains "x"` is `not (A contains "x")`; `!A contains "x"` is `(!A) contains "x"`.
45. Exactly one newline is trimmed after each `for`/`if`/`end for`/`end if` tag; nothing else trims whitespace.
46. A ternary is not associative and nested ternaries render literally.
47. A `for` source may be any uppercase dotted path resolving to a list.
48. An `if` condition may be any condition expression, not only a bare variable.
49. Built-in type names are reserved and cannot be used as `object_shape` names.

---

# Final Quality Check

Before presenting any generated UPL prompt, perform this checklist:

- [ ] Filename is valid.
- [ ] `name` matches filename.
- [ ] `name` is lowercase and valid.
- [ ] `params` exists.
- [ ] Every parameter has a valid type.
- [ ] Every `etype` is valid for its parent type.
- [ ] Every `opts` list has at least two entries.
- [ ] Every option value matches its etype.
- [ ] Every default matches its declared type.
- [ ] Every object has a valid shape.
- [ ] Every object_shape has `ofields`.
- [ ] Every shape reference resolves to an object_shape.
- [ ] No shape-reference cycle exists.
- [ ] Every object option has a valid `label`.
- [ ] No invalid `exclude_condition` exists.
- [ ] Every body variable reference is uppercase.
- [ ] Every dotted path is valid.
- [ ] Every loop variable is uppercase.
- [ ] Every loop source is list-valued.
- [ ] Every `for` has a matching `end for`.
- [ ] Every `if` has a matching `end if`.
- [ ] Every condition is type-compatible.
- [ ] No unsupported UPL syntax is used.
- [ ] The final file is self-contained.
- [ ] The final output can be copied directly into a `.upl` file.
- [ ] `params` is the last metadata field.
- [ ] `title`/`desc` are written bare, without quotes.
- [ ] Nothing follows the body's trailing `--`.
- [ ] Every `def: >>>` heredoc is preceded by `type: long_string`.
- [ ] Every object-shaped option etype has a `label`; no scalar etype has one.
- [ ] No `object_shape` is named after a built-in type.
- [ ] Literal `{{{`/`[[[` text is escaped.
- [ ] `contains` operands are the right way round.
- [ ] `and`/`or`/`not` precedence is as intended, with parentheses where needed.
- [ ] Rendered-output expectations account for block-tag newline trimming.

When possible, recommend validating the finished file with the UPL CLI after installation.

