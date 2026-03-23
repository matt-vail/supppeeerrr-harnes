# /generate — Code Generation

Generate code that matches the conventions of the current project.

## Steps

1. **Detect project context** before writing a single line of code.
   - Identify the primary language and runtime (check `package.json`, `go.mod`, `pyproject.toml`, `Cargo.toml`, etc.).
   - Identify the framework in use (Express, FastAPI, Gin, React, etc.).
   - Scan 2–3 representative existing files to infer naming conventions (camelCase vs snake_case, file-per-class vs co-location, import style, export style, error-handling patterns).
   - Note the test framework if tests exist.

2. **Clarify the request** if the user prompt is ambiguous.
   - Ask at most two targeted questions — do not over-question.
   - If enough context exists, proceed directly.

3. **Generate the code**, strictly following detected conventions.
   - Insert `// TODO:` or `# TODO:` comments wherever a value, behaviour, or integration point needs the developer's attention.
   - Keep each generated unit focused; do not add unrequested features.

4. **Summarise what was created.**
   - List every new file or significant change in a short bullet list.
   - Call out any TODOs that require developer action before the code is production-ready.

## Usage

```
/generate <description of what to generate>
```

### Examples

```
/generate REST endpoint for creating a user
/generate React hook that debounces a search input
/generate pytest fixtures for the database session
```

## Notes

- Delegate to the `code-generator` agent for multi-turn refinement sessions.
- If the project has a linter or formatter config, remind the user to run it after generation.
