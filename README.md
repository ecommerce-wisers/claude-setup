# claude-setup

The Claude Code setup we start every project with. Copy it into a new repo or merge it into an existing one, and Claude Code gets the same instructions, project docs and skills in every project.

## What's inside

```
claude-setup/
├── CLAUDE.md            instructions Claude Code loads in every session (a template)
├── skills-lock.json     the skills.sh skills, each pinned to a source commit
├── .gitignore           keeps secrets and local settings out of git
└── .claude/
    ├── settings.json    project settings; plan-mode plans go to .claude/plans/
    ├── docs/            all project docs
    ├── references/      designs, mockups and explanations supplied for the project
    └── skills/          51 skills
```

## CLAUDE.md

A template. Project-specific values are `add_here_*` placeholders (name, summary, stack, commands, key paths) that you fill in per project. The rest applies as written.

| Section | Covers |
|---|---|
| Stack, Commands | The stack, read-only folders, and the install, dev, test and check commands. The check must pass before every commit. |
| Project docs | Where docs go, which ones load in every session and when to read the others. How to use `.claude/references/` and what goes in `MEMORY.md`. |
| How to work | The Karpathy guidelines (think before coding, keep it simple, make surgical changes, work to a goal you can check), no invented facts, `add_here_*` placeholders instead of sample config files, and no local services the project doesn't need. |
| Skills | Start with `using-agent-skills`. Stack skills apply only on their stack, and the project rules win over a skill's advice. How to add skills at project level. |
| Git | Commit, push and open pull requests only when asked. Short-lived `feature/`, `fix/`, `chore/` and `refactor/` branches, conventional commit messages, no `Co-Authored-By` trailer, no secrets, no force pushes. |
| Writing | Plain, direct prose for docs, commits, pull requests and client messages, without em dashes, emojis or chatbot phrases. |
| Done means | The checks pass, the change was run or looked at, the task is logged and the docs still match the code. |
| Log finished work | How to record each delivered task and its hours in `TASKS.md`. |

## Project docs

Every doc goes in `.claude/docs/`, including the ones the skills write (`SPEC.md`, `tasks/plan.md` and so on), which keep their own names.

| File | Holds |
|---|---|
| `PRD.md` | What the product is, who it's for, the scope and what's out of scope. |
| `ARCHITECTURE.md` | How the code is organised, the data flow, the integrations and the decisions behind them. |
| `DESIGN.md` | The design system: tokens, type, components and UI rules. |
| `RULES.md` | The project's own rules. Loads in every session. |
| `MEMORY.md` | Gotchas, quirks and decisions the code doesn't show. Loads in every session. |
| `PLAN.md` | The project plan: phases, milestones and what comes next. |
| `TASKS.md` | The record of delivered work and hours. |

They start empty, apart from the header and total in `TASKS.md`. Claude appends one entry per finished task and keeps the total up to date:

```markdown
## 12. Checkout window with card payment, order summary and consent
Category: Payments, Hours: 10 hours
```

## Skills

51 skills. 49 come from [skills.sh](https://skills.sh) and are listed in `skills-lock.json`. The two Splide skills are our own and live only in this repo.

| Source | Skills | Covers | Licence |
|---|---|---|---|
| [addyosmani/agent-skills](https://github.com/addyosmani/agent-skills) | 26 | The engineering workflow: idea, spec, plan, incremental build, TDD, debugging, code review, simplification, security, performance, accessibility, CI/CD, docs and ADRs, shipping | MIT |
| [prisma/skills](https://github.com/prisma/skills) | 11 | Prisma ORM setup, client API, CLI, Prisma Postgres, driver adapters, upgrades to v7 | MIT |
| [better-auth/skills](https://github.com/better-auth/skills) | 6 | Better Auth setup, security, email and password, organisations, two-factor auth | Not stated |
| [Shopify/shopify-ai-toolkit](https://github.com/Shopify/shopify-ai-toolkit) | 2 | `shopify` (Admin API, Liquid, Functions, Polaris, Hydrogen, Flow and the other Shopify surfaces) and `ucp` (shopping agents through the UCP CLI) | MIT |
| [vercel-labs/agent-skills](https://github.com/vercel-labs/agent-skills) | 1 | `vercel-react-best-practices` for React and Next.js performance | MIT |
| [vercel-labs/skills](https://github.com/vercel-labs/skills) | 1 | `find-skills` for finding and installing more skills | MIT |
| [Jeffallan/claude-skills](https://github.com/Jeffallan/claude-skills) | 1 | `php-pro` for modern PHP, Laravel and Symfony | MIT |
| [multica-ai/andrej-karpathy-skills](https://github.com/multica-ai/andrej-karpathy-skills) | 1 | `karpathy-guidelines`, the four coding principles | MIT |
| This repo | 2 | `splide-vanilla-js` (Splide v4 sliders in plain JavaScript from the CDN, for static HTML, PHP, WordPress and Shopify themes, with the Splide files bundled) and `splide-react` (Splide in React and Next.js) | Ours. The bundled Splide files are MIT. |

The Prisma, Better Auth, Shopify, React, PHP and Splide skills only matter on those stacks. Remove the ones a project doesn't use with `npx skills remove <name> -y`.

## Add it to a project

You need Claude Code and Node.js 22.20 or later (for `npx skills`). The same steps work for a new project and an existing one. Run them from the project's root:

```sh
git clone --depth 1 https://github.com/ecommerce-wisers/claude-setup.git /tmp/claude-setup

# copy the setup without overwriting anything the project already has
rsync -a --ignore-existing /tmp/claude-setup/.claude/ .claude/
[ -e CLAUDE.md ] || cp /tmp/claude-setup/CLAUDE.md .
[ -e skills-lock.json ] || cp /tmp/claude-setup/skills-lock.json .

# add the .gitignore lines the project doesn't have yet
touch .gitignore
grep -v '^#' /tmp/claude-setup/.gitignore | grep . | while read -r line; do
  grep -qxF "$line" .gitignore || echo "$line" >> .gitignore
done
```

Then:

1. Fill in the placeholders. `grep -rn add_here_ CLAUDE.md .claude/docs/TASKS.md` lists them, or open Claude Code and ask it to fill them in from the codebase.
2. If the project already had a `CLAUDE.md`, it was left alone. Merge the template's sections into it, for example by asking Claude Code to merge `/tmp/claude-setup/CLAUDE.md` into it.
3. If the project already had a `skills-lock.json`, it was left alone too, so the copied skills aren't in it yet. Record them with the `npx skills add` commands below.
4. Delete `/tmp/claude-setup` when you're done.

## Manage the skills

The skills are committed, so a clone already has them.

```sh
npx skills update -p                                             # update the project's skills to their latest versions
npx skills add <owner/repo> --skill <name> -a claude-code -y     # add a skill to this project
npx skills remove <name> -y                                      # remove a skill and its lock entry
```

Always add skills with `-a claude-code` and without `-g`, so they land in `.claude/skills/` and the lock file rather than in your user account.

`npx skills experimental_install` can restore the set from `skills-lock.json`, but it puts the files in `.agents/skills/` and symlinks them into `.claude/skills/`. To get real copies, the way this repo has them, install the whole set from scratch instead:

```sh
npx skills add addyosmani/agent-skills --skill '*' -a claude-code -y
npx skills add prisma/skills --skill '*' -a claude-code -y
npx skills add better-auth/skills --skill '*' -a claude-code -y
npx skills add Shopify/shopify-ai-toolkit --skill '*' -a claude-code -y
npx skills add vercel-labs/agent-skills --skill vercel-react-best-practices -a claude-code -y
npx skills add vercel-labs/skills --skill find-skills -a claude-code -y
npx skills add jeffallan/claude-skills --skill php-pro -a claude-code -y
npx skills add multica-ai/andrej-karpathy-skills --skill karpathy-guidelines -a claude-code -y
```

These commands don't cover the Splide skills, which aren't on skills.sh. Edit them in `.claude/skills/splide-*` directly.
