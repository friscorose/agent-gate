# agent-gate guide

How a solo developer lets AI agents commit code without ever giving them write access to their own
repositories, on GitHub Free. The `agent-gate` script automates §4–§5; this guide explains what it
does, so you can do it by hand or adapt it.

RFC 2119 keywords (MUST, SHOULD, MAY) mark requirements.

---

## 1 Model

```
  agent (cloud or local)                         you (your machine)
  ──────────────────────                         ──────────────────
  push task branches ──► BOT repo ◄── fetch ──── your clone ──► push main ──► OWNER repo
                         <bot>/<repo>            review, merge                 <owner>/<repo>
                            ▲                          │
                            └──── push main (mirror) ──┘
```

Two GitHub accounts, two private repositories, trust in one direction. You read the bot's
repository; nothing the agent holds can write to yours. The gate is **account separation**.
Branch protection and required reviews, the paid features on private repos, only restrict accounts
that already have write access. Here the agent has none, so they would add nothing.

### 1.1 Two routes

| | Private route (default) | Public route |
|---|---|---|
| Owner repository | private | public |
| Bot repository | independent private repository, **not a fork** | **fork** of the owner repository |
| Changes arrive as | branches you fetch and merge locally | pull requests (or branches you fetch, as before) |
| Bot can read owner repository | no | yes, as can anyone |

Why the private route avoids forks: a fork of a private repository is deleted when access to its
parent is removed, and forking one requires the bot to have access in the first place. Neither
applies to a public repository. There a bot fork plus pull request is the ordinary open-source
flow, and it keeps the same guarantee: the bot writes only to its fork, and only you merge.
`agent-gate repo --public` sets this up; `agent-gate check` recognizes either route.

## 2 Invariants

1. The owner repository has **no collaborators**.
2. The bot account **owns nothing but bot repositories** (its own repositories and forks) and has no
   write access to any owner repository; on the private route, no read access either. Every
   credential issued to it is scoped to disposable copies.
3. Only bot credentials go to agent infrastructure.
4. Agents push **task branches only**, to the bot repository. They MUST NOT push `main`.
5. Only the owner merges. The merge is the review record; `--no-ff` keeps one merge commit per
   accepted branch.
6. The bot repository's `main` is a **mirror**. Only the owner updates it, by pushing `main` after
   each merge, so the next branch starts from current code.

`agent-gate check` verifies 1; the identities behind 3; per route, the visible part of 2 (private:
the bot cannot read the owner repository and its repository is not a fork; public: the bot
repository is a fork of the owner repository); and 6 (the bot's `main` matches yours).

## 3 When to use what

| Work | Where it runs | Path to `main` |
|---|---|---|
| Agent session in the cloud | Sourced from the **bot** repository only | Task branch → bot repository → `review`, `land` |
| Agent with no repository access (chat) | No clone | `git format-patch` series against a named base → `git am -3` onto a local branch → review, merge |
| Agent run locally | A **separate clone whose only remote is the bot repository**, in a shell with the bot's `GH_CONFIG_DIR` | As for cloud |
| Your own edits | Your clone | Commit → `git push origin main` → `git push bot main` |

MUST NOT:
- **Run an agent inside your own clone.** It inherits your SSH agent and `gh` login, and the gate
  is gone.
- **Add the bot as a collaborator on your repository.** On personal repositories every collaborator
  has write access; there is no read-only role.
- **Attach two same-named repositories to one cloud session** (Claude Code). The checkouts collide;
  use one session per repository.

## 4 Setup by hand

What `agent-gate setup` and `agent-gate repo` do. Placeholders: `<owner>`, `<bot>`, `<repo>`, SSH
host alias `<alias>`, key `~/.ssh/<botkey>`.

**4.1 Bot account.** A separate GitHub account with its own email address. It owns only bot
repositories.

**4.2 SSH key and host alias** (`~/.ssh/config`):
```
Host <alias>
    User git
    HostName github.com
    IdentityFile ~/.ssh/<botkey>
    IdentitiesOnly yes
    ForwardX11 no
```
`IdentitiesOnly yes` is REQUIRED. Without it ssh offers your SSH agent's keys first, and if one of
yours signs, the push authenticates **as you**. Add the public key to the bot account. Check:
`ssh -T <alias>` prints `Hi <bot>!`.

`IdentitiesOnly` is not enough on its own: `IdentityFile` **accumulates** across every matching
`Host` block. A `Host *` block with `IdentityFile ~/.ssh/id_ed25519` still offers your key to the
alias, and your agent signs with it, behind a confirmation prompt if you use `ssh-add -c`.
Exclude the alias there:
```
Host * !<alias>
    IdentityFile ~/.ssh/id_ed25519
```
Check the effective settings with `ssh -G <alias> | grep -E '^identit'`: one `identityfile` line,
the bot's key, and `identitiesonly yes`. `agent-gate check` tests exactly this.

**4.3 Separate `gh` login for the bot:**
```sh
GH_CONFIG_DIR=~/.config/gh-<bot> gh auth login -h github.com -p ssh --skip-ssh-key \
    -s workflow,admin:public_key --insecure-storage
alias ghbot='GH_CONFIG_DIR=~/.config/gh-<bot> gh'
```
`--insecure-storage` is REQUIRED. The system keyring is shared between `GH_CONFIG_DIR`s, so a
keyring-stored bot token can answer for your own login: `gh auth status` reports one account while
`gh api user` returns the other. With the flag, the token lives in that config directory, mode 600.
Check: `gh api user --jq .login` → `<owner>`; `ghbot api user --jq .login` → `<bot>`.
`admin:public_key` is needed only to upload the key with `gh ssh-key add`.

**4.4 Repositories.** *Private route:* both private, and the bot repository created fresh, **not
as a fork** (§1.1).
```sh
gh    repo create <owner>/<repo> --private
ghbot repo create <bot>/<repo> --private
git remote add origin git@github.com:<owner>/<repo>.git
git remote add bot    git@<alias>:<bot>/<repo>.git
git push origin main && git push bot main
```
You fetch from the bot repository with the bot's own SSH key, so the bot never needs to add you as
a collaborator either.

*Public route:* the owner repository must have at least one commit before it can be forked.
```sh
gh repo create <owner>/<repo> --public
git remote add origin git@github.com:<owner>/<repo>.git && git push -u origin main
ghbot repo fork <owner>/<repo> --clone=false
git remote add bot git@<alias>:<bot>/<repo>.git
```

**4.5 Agent credential.** Give the agent the bot's credential and nothing else.
- **Claude Code (cloud sessions):** in a shell with the bot's `GH_CONFIG_DIR` set
  (`eval "$(agent-gate env)"`), run `claude` and then `/web-setup`. It sends that shell's `gh`
  token to your Claude account. If a session can't clone, install the Claude GitHub App on **the bot
  repository only**.
- **Other agents:** a fine-grained personal access token on the bot account, limited to the bot
  repositories, or the bot's SSH key. Never one of yours.

Your agent account holds one GitHub connection. While it's the bot's, cloud sessions reach only bot
repositories, which is the point.

## 5 Review and merge

```sh
agent-gate review <branch>          # git fetch bot; git log main..bot/<branch>; diffstat
agent-gate review <branch> -p       # the full diff (through your pager); add -- PATH... to narrow
agent-gate review <branch> --by-commit   # each commit with its own diff, oldest first
agent-gate land <branch> -m "..."   # merge --no-ff --no-commit, AG_TEST_CMD, commit,
                                    # push origin main, push bot main, delete bot/<branch>
```
`land` refuses to run off `main`, with a dirty tree, or with `main` out of step with `origin/main`.
Set `AG_TEST_CMD` in `~/.config/agent-gate/config` to run your tests on the merge result before it
is committed; if they fail, the merge is aborted.

**Public route.** The agent pushes `<branch>` to its fork; a pull request from `<bot>:<branch>` to
`<owner>:main` is then opened, either by the agent where its platform allows it, or by you from
GitHub's compare page (`https://github.com/<owner>/<repo>/compare/main...<bot>:<repo>:<branch>`).
Review and merge it on GitHub, then `git pull` and `git push bot main` to keep the fork's `main`
current. `agent-gate land` works on the public route too, if you prefer merging locally.

**Partial acceptance.** Cherry-pick the commits you want, or revert the ones you don't before
merging. Projects that keep policy or documentation commits apart from code commits, with a fixed
subject prefix, make this a per-commit decision. Use a prefix like `docs:`, not `[docs]`:
`git am` strips bracketed subject prefixes.

**Sending a branch back.** Say what to change; the agent pushes new commits on the same branch.
Agents SHOULD NOT rewrite history you have already fetched.

## 6 Project-side conventions (recommended)

Put the agent's own rules in the repository, where agents read them at session start (for Claude
Code, `CLAUDE.md` at the root):
- one task per branch, named for the task; never push `main`;
- the exact test command, run before every push;
- **push at every verified checkpoint**: cloud containers are reclaimed when idle, and unpushed
  work goes with them;
- a committed hand-off file the session updates as its last commit, so the next session starts from
  reviewed state rather than memory.

## 7 Pitfalls (all observed)

- **Shared keyring** across `gh` config dirs → wrong identity (§4.3).
- **SSH agent keys offered first** → possible push as you (§4.2).
- **`IdentityFile` accumulates** across matching `Host` blocks, so a `Host *` key reaches the alias
  despite `IdentitiesOnly` (§4.2). An agent confirmation prompt (`ssh-add -c`) is a backstop, not
  a fix: accepting one prompt pushes as you.
- **Same-named repositories** can't share a Claude Code cloud session (§3).
- **An empty repository can't be forked** (public route): push a first commit as the owner.
- **A 404 is ambiguous.** Holding bot credentials, your private repository answers 404 whether or
  not it exists. That means the gate is working; it doesn't tell you whether the repository exists.
- **Claude Code cloud sessions can push to the fork but can't open the pull request.** The
  session's GitHub proxy scopes API calls to repositories attached with push access, and the owner
  repository is deliberately not one of them. Opening the PR is your one click on the compare page.
  The private route needs no pull requests at all: review happens in git, and its two repositories
  are deliberately not one fork network.
- **URL rewriting.** `git remote get-url` applies `insteadOf` rules; `agent-gate check` reads the
  raw `remote.<name>.url`.

## 8 Revocation

- Revoke the bot's tokens in the bot account's GitHub settings (authorized OAuth apps, GitHub Apps,
  personal access tokens).
- Remove the agent-side credential. For Claude Code, the docs section *Get started with cloud
  sessions → Connect from your terminal* describes what `/web-setup` stores and how to remove it.
- Rotate the bot's SSH key.
- Your repository needs nothing: nothing was ever granted on it.

## 9 Limits

- **No server-side enforcement on your `main`.** None is needed while you're its only writer; paid
  branch protection would guard against your own mistakes, not the agent.
- **Your machine holds both identities.** If it's compromised, so is the gate. That's out of scope.
- **The bot repository is visible to agent infrastructure.** Keep secrets out of the tree, as with
  any agent setup.
- **Solo-oriented.** With several human maintainers, an organization with roles and rulesets is the
  better fit.
