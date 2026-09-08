# UPL Prompts skills

A collection of skills for writing and build valid UPL prompts ([Universal Prompt Language](https://github.com/DavidValin/universal-prompt-language)).

Both skills target the `upl` CLI `0.1.1`, which implements the UPL 1.0 specification.

## Skills

### Write UPL Prompts

Creates and/or updates valid UPL prompts based on Universal Prompt Language (UPL) specification version 1.0

- [Write UPL prompts](skills/write-upl-prompts/SKILL.md)

### Run UPL prompts

Discover and run Universal Prompt Language (UPL) prompts using the `upl` CLI (`upl build-from-json`)

- [Run UPL prompts](skills/run-upl-prompts/SKILL.md)

## Install the skills

Install with the [`skills`](https://github.com/vercel-labs/skills) CLI, which
supports Claude Code, OpenCode, Codex, Cursor and many other agents:

```bash
npx skills add DavidValin/upl-prompt-skills
```

That asks which skills to install, which agents to install them for, and whether
to symlink or copy them. Either way the source is recorded in `skills-lock.json`,
so `npx skills update` can refresh them later.

List what the repository offers without installing anything:

```bash
npx skills add DavidValin/upl-prompt-skills --list
```

Install both skills globally (available in every project) for a specific agent,
without prompts:

```bash
npx skills add DavidValin/upl-prompt-skills --skill '*' -g -a claude-code -y
```

Drop `-g` to install into the current project instead, and pass `-a` more than
once (`-a claude-code -a opencode`) to target several agents. A single skill can
be installed on its own:

```bash
npx skills add DavidValin/upl-prompt-skills --skill write-upl-prompts
npx skills add DavidValin/upl-prompt-skills --skill run-upl-prompts
```

Then check the result, and update later, with:

```bash
npx skills list
npx skills update
```

## Install the `upl` CLI

The skills drive the `upl` binary, so install it as well — release `0.1.1`:

```bash
curl -sL -O https://github.com/DavidValin/universal-prompt-language/releases/download/0.1.1/upl-0.1.1-linux-x86_64.tar.gz
tar -xzf upl-0.1.1-linux-x86_64.tar.gz
sudo install -m 0755 upl-0.1.1-linux-x86_64/upl /usr/local/bin/upl
upl --help | head -1   # upl 0.1.1
```

Assets for the other platforms (Linux ARM64 and musl, macOS, Windows, FreeBSD,
NetBSD, OpenBSD) are listed on the
[release page](https://github.com/DavidValin/universal-prompt-language/releases/tag/0.1.1);
both skills also describe how to pick and install the right one.
