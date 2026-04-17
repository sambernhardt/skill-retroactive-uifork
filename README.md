# skill-retroactive-uifork

Agent skill: turn **git commit history** for a UI component into a **browsable [UIFork](https://github.com/sambernhardt/uifork) timeline** (retroactive versioning—review past milestones in-context without checking out each SHA).

## Layout

- `skills/retroactive-uifork/SKILL.md` — skill definition

## Use in a project

**Option A — skills CLI** (if your environment supports installing from a path or git URL):

```bash
npx skills add /path/to/skill-retroactive-uifork
# or after publishing:
# npx skills add <your-github-org>/skill-retroactive-uifork
```

**Option B — copy** the `skills/retroactive-uifork/` folder into your repo’s agent skills directory.

## Companion

Install the upstream UIFork skill for CLI and wiring details:

```bash
npx skills add sambernhardt/uifork
```

## License

MIT (match your preference when publishing).
