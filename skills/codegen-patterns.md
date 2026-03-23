---
name: codegen-patterns
description: >
  Convention-matching and structured output rules for greenfield code generation.
  Use when scaffolding a new project, generating boilerplate, creating a new
  module from scratch, or producing starter code that does not yet exist in the
  codebase. Enforces convention detection before writing and structured output
  with TODO markers for all unresolved decision points.
---

# Skill: codegen-patterns

When this skill is active, apply the following rules to every code generation task in the conversation.

## Convention detection (do this first, always)

Before writing any code:

1. Locate and read 2–3 representative source files of the same type being generated.
2. Extract and note:
   - **Naming style**: camelCase / snake_case / PascalCase per identifier type.
   - **File naming**: how files are named for this type of module.
   - **Import style**: named vs default exports, barrel files, path aliases.
   - **Error handling**: exceptions vs result types vs error callbacks.
   - **Async style**: async/await, Promises, callbacks, goroutines, etc.
   - **Logging/instrumentation**: what logger is used and how it is called.

## Generation rules

- Match every detected convention exactly. Do not introduce a new pattern because you prefer it.
- If two conventions conflict, use the one found in the file closest in type to what is being generated.
- Mark every placeholder or decision point with a `TODO` comment using the project's comment syntax.
- Do not add abstraction layers, utility helpers, or generalisations unless explicitly requested.
- Do not add logging, metrics, or tracing beyond what the surrounding code already does.

## Output structure

1. Generated file(s) — verbatim, ready to paste or write.
2. A concise bullet list of what was created.
3. A separate bullet list of every TODO that requires developer attention.

## Quality gate

Before presenting output, mentally check:
- [ ] Naming matches detected conventions.
- [ ] No unrequested features added.
- [ ] All decision points have TODOs.
- [ ] Imports follow the detected style.
