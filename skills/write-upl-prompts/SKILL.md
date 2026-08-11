---
name: write-upl-prompts
description: Creates and/or updates valid UPL prompts based on Universal Prompt Language (UPL) specification version 1.0-rc.3
---

# UPL Prompt Authoring Skill

## Purpose

You are an expert author of Universal Prompt Language (UPL) prompts.

Your job is to create, validate, debug, and improve `.upl` / `.txt` UPL prompt files that conform to the **Universal Prompt Language 1.0-rc.3 Official Standard Specification**.

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
- A leading `--` before metadata is optional.
- A trailing `--` after the body is optional.
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

When `option_single` or `option_multi` uses a referenced `object_shape` as its `etype`, `label` is REQUIRED.

The label MUST name a field on the referenced shape whose type is `string` or `long_string`.

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

Do not use `label` for scalar etypes.

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

Nested ternaries are not supported.

---

## If Blocks

Use:

    {{{if INCLUDE_AUTH}}}
    Authorization: [[[TOKEN]]]
    {{{end if}}}

The condition is bare and uppercase.

Do NOT wrap the condition in `[[[...]]]`.

Example:

    {{{if ENABLED}}}
    Feature is enabled.
    {{{end if}}}

---

## For Loops

Use:

    {{{for ITEM in ITEMS}}}
    - [[[ITEM.NAME]]]
    {{{end for}}}

Both the loop variable and source variable MUST be uppercase.

The source MUST be a list-valued variable:

- `list`
- `option_multi`

Backward-compatible syntax with `[[[VAR]]]` around the list is tolerated by the specification, but prefer the canonical syntax:

    {{{for ITEM in ITEMS}}}

Do not use:

    {{{for item in items}}}

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

---

## Condition Precedence

From highest to lowest:

1. `!`
2. `>`, `<`, `>=`, `<=`
3. `=`, `!=`
4. `contains`, `starts_with`, `ends_with`
5. `? :`

Ternary is right-associative.

Parentheses may be used for grouping.

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
- `label` is present for referenced object-shape option types.
- `label` points to a string/long_string field.
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
- forget to close `for` or `if` blocks.

---

# CLI Usage and Installation

When the user asks to validate, render, test, inspect, or otherwise use a UPL file and the UPL CLI is not available, explain that the CLI can be installed from the official GitHub release artifacts.

Official release:

    https://github.com/DavidValin/universal-prompt-language/releases/tag/0.1.1-rc.2

Available artifacts:

### Linux ARM64 / AArch64

glibc:

    https://github.com/DavidValin/universal-prompt-language/releases/download/0.1.1-rc.2/upl-aarch64_linux.tar.gz

musl:

    https://github.com/DavidValin/universal-prompt-language/releases/download/0.1.1-rc.2/upl-aarch64_linux_musl.tar.gz

### macOS ARM64 / Apple Silicon

    https://github.com/DavidValin/universal-prompt-language/releases/download/0.1.1-rc.2/upl-aarch64_macos.tar.gz

### Windows ARM64

    https://github.com/DavidValin/universal-prompt-language/releases/download/0.1.1-rc.2/upl-aarch64_windows.zip

### Linux x86_64

glibc:

    https://github.com/DavidValin/universal-prompt-language/releases/download/0.1.1-rc.2/upl-x86_64_linux.tar.gz

musl:

    https://github.com/DavidValin/universal-prompt-language/releases/download/0.1.1-rc.2/upl-x86_64_linux_musl.tar.gz

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

For glibc Linux, use the normal Linux archive.

For systems using musl, use the `_musl` artifact.

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
9. Object-shaped options require `label`.
10. `label` must reference a string or long_string field.
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
22. Unmatched expansion delimiters are emitted literally, except malformed recognized blocks which must be reported appropriately.
23. `==` is an alias for `=`.
24. `contains` works for strings and list membership.
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

When possible, recommend validating the finished file with the UPL CLI after installation.

