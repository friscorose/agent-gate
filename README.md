# agent-gate

Let AI coding agents commit to your project **without ever giving them write access to your
repositories**. Agents push branches to a bot account's copy; you review and merge into yours.
It works on GitHub Free and needs no paid features.

```
agent ──push branch──► bot/<repo> ◄──fetch── you ──merge, push──► you/<repo>
```

## Why

On GitHub Free you can't protect branches on private repos, and every collaborator on a personal
repo gets write access. agent-gate doesn't fight that. Agents only ever hold a **second account's**
credential, and that account has no access to your repos. The gate is account separation, and
your merge is the review.

## Quick start

Needs bash 3.2+ (the script's interpreter; your login shell can be anything), `git`, `gh`
(GitHub CLI) and OpenSSH. Create a second GitHub account for the bot first.

```sh
install -m 755 agent-gate ~/.local/bin/        # or anywhere on PATH
agent-gate setup --owner <you> --bot <bot>     # SSH key + alias, separate gh login for the bot
cd your-project
agent-gate repo                                 # private <you>/<repo> + <bot>/<repo>, remotes, push
agent-gate check                                # verify identities and gate invariants
```

Then connect your agent **as the bot**. For Claude Code cloud sessions:

```sh
eval "$(agent-gate env)"    # this shell's gh is now the bot
claude                         # then run /web-setup inside it
```

Start agent sessions from `<bot>/<repo>`. When an agent has pushed a branch:

```sh
agent-gate review <branch>  # commits and diffstat; -p full diff, --by-commit per commit
agent-gate land <branch>    # merge --no-ff, optional tests, push to both, delete the branch
```

**Public project?** `agent-gate repo --public` makes your repo public and the bot's copy a
fork, so changes arrive as ordinary pull requests. Same guarantee: the bot writes only to its fork.

## Rules that keep the gate closed

- Never add the bot as a collaborator on your repo.
- Never give an agent your own credential, and never run an agent inside your own clone.
- Agents push branches only; only you merge. The bot repo's `main` is a mirror only you update.

`agent-gate check` tests the ones a machine can see.

## More

[docs/GUIDE.md](docs/GUIDE.md) covers the full model, manual setup without the script, when to use
what, review practice, pitfalls, revocation and limits.

MIT licensed.
