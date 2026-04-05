# custom-claude-skills

Custom Claude Code skills.

## Installation

Skills can be installed at two scopes:

**Personal (all your projects)**
Copy a skill directory into `~/.claude/skills/`:
```bash
cp -r interview-briefer ~/.claude/skills/
```

**Project-level (this project only)**
Copy a skill directory into `.claude/skills/` at the root of your project:
```bash
cp -r interview-briefer path/to/your-project/.claude/skills/
```

No configuration is required — Claude Code auto-discovers skills from these locations.

### Usage

Once installed, invoke a skill by typing its name as a slash command:
```
/interview-briefer
/better-auth-nextjs-express
```

Claude will also invoke skills automatically when your prompt matches the skill's description.

## Skills

### `better-auth-nextjs-express`
Setup, configuration, and debugging for **Better Auth** with a Next.js frontend (App Router or Pages Router), Express.js backend, and PostgreSQL database. Covers cross-domain deployments (Vercel + Railway), cookie/session issues, CORS, and 307 redirect problems.

### `interview-briefer`
Researches a target company and generates a personalised interview prep briefing. Takes a company name and optional job description link, reads your background and behavioural story bank, and produces a polished briefing document covering company context, role fit, talking points, and story mapping.

Requires filling in `interview-briefer/references/about_candidate.md` and `interview-briefer/references/behavioural_stories.md` before use. See the `example_*` files in that folder for reference.
