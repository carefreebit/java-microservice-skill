# Custom Stack Presets

User-defined stack presets live here. They are read exactly like the built-in presets.

## How to add

1. Copy `../_template.yaml` into this directory.
2. Rename it to `<company>-<stack>.yaml`, for example `acme-sc-boot3.yaml`.
3. Fill in every required field. Keep versions that the project actually uses.
4. Do not put secrets, credentials, absolute paths, or machine addresses in the file.

## Rules

- `custom/` is preserved when the skill is updated or reinstalled.
- A custom preset does not override a built-in preset with the same `id` unless the user explicitly chooses it; report the conflict instead of silently replacing.
- `verified_at` is required. Mark unknown compatibility as `experimental` and explain it under `constraints`.

## Copy to another computer

Copy the entire `java-microservice` skill folder (including this directory) to the target machine's `~/.claude/skills/` or project `.claude/skills/`. Start a new session so the skill rescans the directory.
