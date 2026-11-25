<!--
claude-setup template. Replace every add_here_* value and delete what doesn't apply.
Block-level HTML comments like this one are stripped before Claude reads the file.
Keep the file short (well under 200 lines). Anything that only matters for one part of
the codebase belongs in a skill or a path-scoped rule in .claude/rules/.
-->

# add_here_project_name

add_here_project_summary: what the project is, who it's for and the current milestone, in two or three sentences.

## Stack
- add_here_stack: framework and version, language, package manager, database, hosting.
- Read-only folders: add_here_read_only_paths (for example a legacy app kept for reference). Read them for behaviour. Never edit or run them.

## Commands
- Install: `add_here_install_command`
- Dev server: `add_here_dev_command`
- Tests: `add_here_test_command`
- Check: `add_here_check_command` (lint, types, build and tests). It must pass before every commit.

## Project docs
All docs go in `.claude/docs/`, including the ones the skills write, which keep their own file names (for example `.claude/docs/SPEC.md` and `.claude/docs/tasks/plan.md`).

Of the project docs, the rules and the memory load in every session:
- Rules: @.claude/docs/RULES.md
- Memory: @.claude/docs/MEMORY.md

Read the others when the task needs them:
- `.claude/docs/PRD.md`: what the product is, who it's for, the scope and what's out of scope. Read it before planning a feature.
- `.claude/docs/ARCHITECTURE.md`: how the code is organised, the data flow, the integrations and the decisions behind them. Read it before changing structure, data models or dependencies.
- `.claude/docs/DESIGN.md`: the design system (tokens, type, components, UI rules). Read it before any UI work.
- `.claude/docs/PLAN.md`: the project plan, with the phases, milestones and what comes next. Read it before planning work or picking the next task.
- `.claude/docs/TASKS.md`: the record of delivered work and hours. Log each finished task there (see Log finished work below).

`.claude/references/` holds the material the user supplies: designs, mockups, screenshots, client briefs, API notes and other explanations. Before UI work, look for the matching design there along with `DESIGN.md`. When a task, doc or the user points to a reference, read it before you start. When a reference and a doc disagree, ask which one is right. Treat the folder as read-only: don't edit, rename or delete files in it.

When a change makes a doc wrong, fix the doc in the same change. The docs describe how things are now and git keeps the history. `TASKS.md` is the exception: it's a running log.

Add to `MEMORY.md` when you learn something the code doesn't show and the next session will need: a gotcha, a command quirk, a decision and its reason, a correction from the user. Keep each entry to a line or two, grouped by topic, and remove entries that stop being true. This file is shared through git. Claude Code's auto memory isn't.

## How to work
- Think before coding. State your assumptions. If the request can be read more than one way, or something is unclear, ask instead of guessing. If a simpler approach exists, say so.
- Keep it simple. Write the least code that solves the problem, with no features, abstractions, options or error handling that nobody asked for.
- Make surgical changes. Touch only what the task needs and match the existing style. Mention unrelated problems instead of fixing them, and remove only the code your own change left unused.
- Work towards a goal you can check. Turn the task into success criteria (a failing test, a command that must pass) and keep going until they're met. For multi-step work, write a short plan with a check for each step.
- Don't invent facts. Names, prices, copy, IDs, URLs and credentials come from the user or the docs. When one is missing, put `add_here_<name>` where it's needed and say so.
- Don't create sample config files such as `.env.example`. Put `add_here_*` placeholders in the real file instead.
- Don't add local services (databases, queues, containers) the project doesn't already use unless the task asks for them.

## Skills
The skills in `.claude/skills/` are recorded in `skills-lock.json`.
- Start with `using-agent-skills`. It maps a task to its workflow (interview, spec, plan, build, test, review, ship) and to the skill for each step. Follow that skill's steps, including its checks and the points where the user reviews.
- The stack skills apply only when the project uses that stack: `prisma-*` for Prisma, the Better Auth set (`better-auth-*`, `create-auth`, `email-and-password-best-practices`, `organization-best-practices`, `two-factor-authentication-best-practices`), `shopify` and `ucp` for Shopify, `php-pro` for PHP and `vercel-react-best-practices` for React and Next.js.
- When a skill's advice conflicts with this file or `RULES.md`, this file and `RULES.md` win.
- Add skills at project level, so they land here and in the lock file: `npx skills add <owner/repo> --skill <name> -a claude-code`. Don't use `-g`, which installs for your user account instead of the repo. `npx skills update -p` updates them and `npx skills remove <name> -y` removes one.

## Git
- Commit only when the user asks. Push, open or merge a pull request only when asked.
- Work on a short-lived branch from `main`: `feature/<name>`, `fix/<name>`, `chore/<name>` or `refactor/<name>`.
- Write commit messages as `<type>: <short description>`, where the type is `feat`, `fix`, `refactor`, `test`, `docs` or `chore`, then a body that says why. One concern per commit.
- No `Co-Authored-By` trailer.
- Run the check command first. Read `git status`, and stage files by name rather than `git add -A`.
- Never commit secrets: `.env*` files, `.claude/settings.local.json`, `CLAUDE.local.md`, or keys and passwords in the diff.
- Never force push, rewrite history or delete branches unless asked.

## Writing
This covers docs, commit messages, pull request descriptions, code comments and messages for the user or a client.
- Be plain and direct. Skip the intro, the recap of what you just said and the offer of more help.
- No em dashes, emojis, hype words (seamless, robust, powerful, leverage, crucial, delve) or chatbot phrases ("Great question", "I hope this helps").
- Use sentence case for headings and bold only where it's needed.
- Code comments say why, not what.

## Done means
- `add_here_check_command` passes, along with any tests the task added.
- The change was run, tested or looked at, not only written. Say what you couldn't check.
- The task is logged in `TASKS.md`, and the other docs still match the code.

## Log finished work in TASKS.md
`TASKS.md` is the project's record of work and hours. When you finish a piece of project work, log it in the same turn, before you report back.

Only project work goes in: features, pages, design, fixes, backend, integrations, tests, research and code reviews. Never log documentation work, such as writing `CLAUDE.md`, the docs, rules, plans or notes. When a task included some documentation, log the project part only. A question answered in chat isn't a task.

1. Append the task at the end as `## N. <what was delivered>`, where N is the last number plus 1. Never renumber, reorder or reword earlier tasks, and never change their hours unless the user asks.
2. Under it, write `Category: <category>, Hours: <hours>`. Reuse a category already in the file when one fits (for example Planning, Setup, UI, Pages, Design, Backend, Payments, Integrations, Testing, Refactor, Fixes, Research). Hours go in half-hour steps, written `0.5 hours`, `1 hour`, `1.5 hours`.
3. Update `**Total: X hours**` at the foot, and end the date range on line 3 at today's date.
4. Check the total with a command, not by hand: `grep -oE "Hours: [0-9.]+" .claude/docs/TASKS.md | awk '{s+=$2} END {print s}'`
5. Tell the user the task title, its hours and the new total, so they can correct the estimate.

Keep the title to one short sentence in plain language that says what was delivered, with no file paths, component names or code terms. Use one task per deliverable. A fix to earlier work is its own task ("Checkout fix: ..."), and several small changes made together can be one task. Estimate from the size and risk of the work, not from how long the session took. When it's unclear whether something should be billed, ask.

## Where things are
- `.claude/docs/`: all docs, the project docs above and the skills' docs.
- `.claude/references/`: the user's reference files (designs and explanations). Read-only.
- `.claude/skills/` and `skills-lock.json`: the skills.
- `.claude/plans/`: plan-mode plans (`plansDirectory` in `.claude/settings.json`).
- add_here_key_paths: the few places a new developer needs to know (entry point, routes, components), one line each.
